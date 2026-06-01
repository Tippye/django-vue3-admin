# 后端 Skill — 日志系统

> 本文档规范 django-vue3-admin 框架的日志体系，包括系统日志配置、API 访问日志、操作日志、登录日志以及异常日志处理。

---

## 一、日志配置

### 1. 日志目录结构

```
backend/logs/
├── server.log          # 普通日志（INFO 级别）
└── error.log           # 错误日志（ERROR 级别）
```

### 2. 配置文件（`application/settings.py`）

系统在 `settings.py` 中定义了 `LOGGING` 配置：

```python
# 日志文件路径
SERVER_LOGS_FILE = os.path.join(BASE_DIR, "logs", "server.log")
ERROR_LOGS_FILE = os.path.join(BASE_DIR, "logs", "error.log")
LOGS_FILE = os.path.join(BASE_DIR, "logs")

# 自动创建日志目录
if not os.path.exists(os.path.join(BASE_DIR, "logs")):
    os.makedirs(os.path.join(BASE_DIR, "logs"))

# 日志格式
STANDARD_LOG_FORMAT = (
    "[%(asctime)s][%(name)s.%(funcName)s():%(lineno)d] [%(levelname)s] %(message)s"
)

LOGGING = {
    "version": 1,
    "disable_existing_loggers": False,
    "formatters": {
        "standard": {"format": STANDARD_LOG_FORMAT},
        "console": {
            "format": STANDARD_LOG_FORMAT,
            "datefmt": "%Y-%m-%d %H:%M:%S",
        },
    },
    "handlers": {
        "file": {
            "level": "INFO",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": SERVER_LOGS_FILE,
            "maxBytes": 1024 * 1024 * 100,  # 100 MB
            "backupCount": 5,
            "formatter": "standard",
            "encoding": "utf-8",
        },
        "error": {
            "level": "ERROR",
            "class": "logging.handlers.RotatingFileHandler",
            "filename": ERROR_LOGS_FILE,
            "maxBytes": 1024 * 1024 * 100,
            "backupCount": 3,
            "formatter": "standard",
            "encoding": "utf-8",
        },
        "console": {
            "level": "INFO",
            "class": "logging.StreamHandler",
            "formatter": "console",
        },
    },
    "loggers": {
        "": {
            "handlers": ["console", "error", "file"],
            "level": "INFO",
        },
        "django": {
            "handlers": ["console", "error", "file"],
            "level": "INFO",
            "propagate": False,
        },
        "uvicorn.error": {
            "level": "INFO",
            "handlers": ["console", "error", "file"],
        },
    },
}
```

---

## 二、API 访问日志中间件

### 1. 强制记录配置

```python
# settings.py
API_LOG_ENABLE = True  # 开启日志记录
API_LOG_METHODS = ["POST", "UPDATE", "DELETE", "PUT"]  # 需要记录的 HTTP 方法
# API_LOG_METHODS = 'ALL'  # 记录所有请求
```

### 2. 模型定义（`dvadmin/system/models.py`）

```python
class OperationLog(CoreModel):
    request_modular = models.CharField(max_length=64, verbose_name="请求模块")
    request_path = models.CharField(max_length=400, verbose_name="请求地址")
    request_body = models.TextField(verbose_name="请求参数")
    request_method = models.CharField(max_length=8, verbose_name="请求方式")
    request_msg = models.TextField(verbose_name="操作说明")
    request_ip = models.CharField(max_length=32, verbose_name="请求ip地址")
    request_browser = models.CharField(max_length=64, verbose_name="请求浏览器")
    request_os = models.CharField(max_length=64, verbose_name="操作系统")
    response_code = models.CharField(max_length=32, verbose_name="响应状态码")
    json_result = models.TextField(verbose_name="返回信息")
    status = models.BooleanField(default=False, verbose_name="响应状态")
```

### 3. 中间件工作原理（`dvadmin/utils/middleware.py`）

`ApiLoggingMiddleware` 的工作流程：

1. **process_view**: 检测视图是否有 queryset，创建空日志记录获取 log_id
2. **process_request**: 提取请求参数、IP 等信息
3. **process_response**: 更新日志记录，填充响应信息

```python
class ApiLoggingMiddleware(MiddlewareMixin):
    def __init__(self, get_response=None):
        super().__init__(get_response)
        self.enable = getattr(settings, 'API_LOG_ENABLE', False)
        self.methods = getattr(settings, 'API_LOG_METHODS', set())

    def process_view(self, request, view_func, view_args, view_kwargs):
        # 在视图处理前创建日志记录
        if hasattr(view_func, 'cls') and hasattr(view_func.cls, 'queryset'):
            if self.enable and (self.methods == 'ALL' or request.method in self.methods):
                log = OperationLog(request_modular=get_verbose_name(view_func.cls.queryset))
                log.save()
                request.request_data['log_id'] = log.id

    def process_response(self, request, response):
        # 响应时更新日志
        if self.enable:
            if self.methods == 'ALL' or request.method in self.methods:
                # ... 填充日志字段
                OperationLog.objects.update_or_create(defaults=info, id=log_id)
        return response
```

### 4. 安全处理

- **密码脱敏**：请求体中的 `password` 字段会被替换为 `****`
- **log_id 移除**：不将内部的 `log_id` 记录到请求体中

---

## 三、登录日志

### 模型定义

```python
class LoginLog(CoreModel):
    LOGIN_TYPE_CHOICES = ((1, "普通登录"), (2, "微信扫码登录"),)
    username = models.CharField(max_length=32, verbose_name="登录用户名")
    ip = models.CharField(max_length=32, verbose_name="登录ip")
    agent = models.TextField(verbose_name="agent信息")
    browser = models.CharField(max_length=200, verbose_name="浏览器名")
    os = models.CharField(max_length=200, verbose_name="操作系统")
    # ... IP 地理信息字段（州、国家、省份、城市、经纬度等）
    login_type = models.IntegerField(default=1, choices=LOGIN_TYPE_CHOICES)
```

### 记录登录日志

在 `dvadmin/utils/request_util.py` 中定义了 `save_login_log` 函数：

```python
def save_login_log(request):
    ip = get_request_ip(request=request)
    analysis_data = get_ip_analysis(ip)  # 获取 IP 地理信息
    analysis_data['username'] = request.user.username
    analysis_data['ip'] = ip
    analysis_data['agent'] = str(parse(request.META['HTTP_USER_AGENT']))
    analysis_data['browser'] = get_browser(request)
    analysis_data['os'] = get_os(request)
    analysis_data['creator_id'] = request.user.id
    analysis_data['dept_belong_id'] = getattr(request.user, 'dept_id', '')
    LoginLog.objects.create(**analysis_data)
```

---

## 四、异常日志处理

### 异常处理器（`dvadmin/utils/exception.py`）

系统统一了异常响应格式，避免暴露 500 错误给前端：

```python
def CustomExceptionHandler(ex, context):
    msg = ''
    code = 4000
    response = exception_handler(ex, context)
    
    if isinstance(ex, AuthenticationFailed):
        code = 401
        msg = ex.detail
    elif isinstance(ex, Http404):
        code = 400
        msg = "接口地址不正确"
    elif isinstance(ex, DRFAPIException):
        set_rollback()
        msg = ex.detail
    elif isinstance(ex, ProtectedError):
        set_rollback()
        msg = "删除失败:该条数据与其他数据有相关绑定"
    elif isinstance(ex, Exception):
        logger.exception(traceback.format_exc())  # 记录完整堆栈
        msg = str(ex)
    
    return ErrorResponse(msg=msg, code=code)
```

---

## 五、自定义日志使用

### 在视图中使用

```python
import logging

logger = logging.getLogger(__name__)

class MyViewSet(CustomModelViewSet):
    def create(self, request, *args, **kwargs):
        logger.info(f"用户 {request.user.username} 创建了新数据")
        try:
            result = super().create(request, *args, **kwargs)
            logger.info("创建成功")
            return result
        except Exception as e:
            logger.error(f"创建失败: {str(e)}")
            raise
```

### 在任务中使用

```python
@app.task
def my_task():
    logger = logging.getLogger(__name__)
    logger.info("任务开始执行")
    # ... 任务逻辑
    logger.info("任务执行完成")
```

---

## 六、健康检查中间件

`HealthCheckMiddleware` 提供 `/healthz` 和 `/readiness` 健康检查端点：

```python
class HealthCheckMiddleware(object):
    def __call__(self, request):
        if request.method == "GET":
            if request.path == "/readiness":
                return self.readiness(request)  # 检查数据库连接
            elif request.path == "/healthz":
                return self.healthz(request)    # 简单返回 OK
        return self.get_response(request)
```

---

## 七、日志相关设置对照表

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `API_LOG_ENABLE` | 是否开启 API 日志 | `True` |
| `API_LOG_METHODS` | 记录哪些 HTTP 方法 | `["POST", "UPDATE", "DELETE", "PUT"]` |
| `API_MODEL_MAP` | 自定义模块名称映射 | `{}` |
| `LOGGING['handlers']['file']['maxBytes']` | 日志文件大小上限 | `100MB` |
| `LOGGING['handlers']['file']['backupCount']` | 日志文件保留个数 | `5` |
