# 后端 Skill — 异步任务（Celery Tasks）

> 本文档规范基于 django-vue3-admin 框架的异步任务开发，主要负责 Celery 任务定义、异步导出、下载中心等功能。

---

## 一、模块位置

| 文件 | 说明 |
|------|------|
| `backend/application/celery.py` | Celery 应用实例配置 |
| `backend/dvadmin/system/tasks.py` | 系统级异步任务定义 |
| `backend/dvadmin/system/models.py`中 `DownloadCenter` | 下载任务状态模型 |

---

## 二、Celery 配置

### 1. 配置文件（`application/celery.py`）

```python
import os
from celery import Celery

# 设置默认 Django 设置模块
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'application.settings')

app = Celery('application')

# 使用 Django 配置文件中以 CELERY_ 为前缀的配置
app.config_from_object('django.conf:settings', namespace='CELERY')

# 自动从注册的 Django app 中发现任务
app.autodiscover_tasks()
```

### 2. 启动 Celery Worker

```bash
# 启动 Worker
celery -A application worker -l info

# 启动 Beat（定时任务）
celery -A application beat -l info

# 一起启动
celery -A application worker -l info --beat
```

---

## 三、异步数据导出规范

### 1. 导出任务函数

系统提供了 `async_export_data` 任务函数，位于 `dvadmin/system/tasks.py`：

```python
from application.celery import app

@app.task
def async_export_data(data: list, filename: str, dcid: int, export_field_label: dict):
    """
    异步导出数据到 Excel
    
    :param data: 导出的数据列表（序列化后的字典列表）
    :param filename: 导出的文件名
    :param dcid: DownloadCenter 实例 ID，用于更新任务状态
    :param export_field_label: 字段映射 {model字段: 显示名}
    
    任务状态：
      0=任务已创建, 1=任务进行中, 2=任务完成, 3=任务失败
    """
    instance = DownloadCenter.objects.get(pk=dcid)
    instance.task_status = 1  # 进行中
    instance.save()
    try:
        # ... 生成 Excel 逻辑
        wb = Workbook()
        # ... 存储文件
        instance.task_status = 2  # 完成
    except Exception as e:
        instance.task_status = 3  # 失败
        instance.description = str(e)[:250]
    instance.save()
```

### 2. 视图集中触发导出

在 `CustomModelViewSet` 的子类中，继承 `ExportSerializerMixin` 后自动拥有 `export_data` 接口：

```python
from dvadmin.utils.import_export_mixin import ExportSerializerMixin
from dvadmin.utils.viewset import CustomModelViewSet

class MyViewSet(CustomModelViewSet, ExportSerializerMixin):
    """
    自动拥有 export_data 接口
    GET /api/system/user/export_data/  → 触发异步导出
    """
    export_field_label = {
        "username": "用户账号",
        "name": "用户名称",
        "email": "邮箱",
    }
    export_serializer_class = MyExportSerializer
```

---

## 四、异步任务开发规范

### 1. 新增异步任务的步骤

**Step 1**：在 `dvadmin/某模块/tasks.py` 中定义任务

```python
from application.celery import app

@app.task
def my_async_task(param1: str, param2: int):
    """
    自定义异步任务说明
    """
    try:
        # 任务逻辑
        result = do_something(param1, param2)
        return {"status": "success", "result": result}
    except Exception as e:
        # 记录异常，任务失败
        return {"status": "error", "message": str(e)}
```

**Step 2**：在视图中调用

```python
from dvadmin.system.tasks import my_async_task

class MyViewSet(CustomModelViewSet):
    @action(methods=['post'], detail=False)
    def trigger_task(self, request):
        """触发异步任务"""
        result = my_async_task.delay(
            request.data.get('param1'),
            request.data.get('param2')
        )
        return DetailResponse(data={"task_id": result.id}, msg="任务已提交")
```

### 2. 任务函数编写规范

- **必须使用 `@app.task` 装饰器**
- **参数必须可序列化**（JSON 序列化）
- **异常必须 capture**，避免任务崩溃无法跟踪
- **长任务应更新状态到 DownloadCenter 或自定义状态模型**

```python
@app.task
def long_running_task(user_id: int, record_id: int):
    """
    长任务示例：定期更新进度
    """
    record = TaskRecord.objects.get(id=record_id)
    record.status = 1  # 进行中
    record.save()
    
    try:
        total = 100
        for i in range(total):
            # 处理逻辑
            do_step(i)
            # 可选：更新进度
            record.progress = int((i / total) * 100)
            record.save()
        
        record.status = 2  # 完成
        record.save()
    except Exception as e:
        record.status = 3  # 失败
        record.error_msg = str(e)
        record.save()
        raise  # 重新抛出以便 Celery 记录失败
```

---

## 五、DownloadCenter 下载中心

### 模型定义

`DownloadCenter` 模型用于跟踪异步任务状态：

```python
class DownloadCenter(CoreModel):
    TASK_STATUS_CHOICES = [
        (0, '任务已创建'),
        (1, '任务进行中'),
        (2, '任务完成'),
        (3, '任务失败'),
    ]
    task_name = models.CharField(max_length=255, verbose_name="任务名称")
    task_status = models.SmallIntegerField(default=0, choices=TASK_STATUS_CHOICES)
    file_name = models.CharField(max_length=255, null=True, blank=True)
    url = models.FileField(upload_to=media_file_name_downloadcenter, null=True, blank=True)
    size = models.BigIntegerField(default=0)
    md5sum = models.CharField(max_length=36, null=True, blank=True)
```

### 使用场景

1. 用户触发导出
2. 系统创建 DownloadCenter 记录（状态=0）
3. 异步任务执行（状态=1→2或3）
4. 用户在下载中心页面查看任务状态并下载文件

---

## 六、常用任务模板

### 定时任务（Beat）

```python
from celery import shared_task
from celery.schedules import crontab

# 在 settings.py 中配置 beat_schedule
CELERY_BEAT_SCHEDULE = {
    'cleanup-login-logs': {
        'task': 'dvadmin.system.tasks.cleanup_old_logs',
        'schedule': crontab(hour=2, minute=0),  # 每天凌晨2点
    },
}

@app.task
def cleanup_old_logs():
    """清理 30 天前的登录日志"""
    from datetime import datetime, timedelta
    cutoff = datetime.now() - timedelta(days=30)
    LoginLog.objects.filter(create_datetime__lt=cutoff).delete()
```

### 链式任务

```python
@app.task(bind=True)
def chained_task(self, data):
    """
    链式任务示例，使用 bind=True 获取 self 访问任务实例
    """
    try:
        result = step1(data)
        # 自动重试配置在 settings.py 中
        return result
    except Exception as exc:
        # 手动触发重试
        raise self.retry(exc=exc, countdown=60, max_retries=3)
```

---

## 七、开发注意事项

1. **任务必须是纯函数**：不依赖 request 对象，需要的数据作为参数传入
2. **数据库操作**：任务中使用 Django ORM 时，Celery 会自动处理数据库连接
3. **文件操作**：任务中生成的文件应存储到 `MEDIA_ROOT` 下的适当位置
4. **日志记录**：使用 Python 的 `logging` 模块记录任务执行日志
