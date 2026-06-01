# 后端 Skill — 基础模型（Models）

> 本文档规范 django-vue3-admin 框架中核心模型基类的使用方法，包括 CoreModel、SoftDeleteModel 以及工具函数。

---

## 一、CoreModel 基类

位置：`dvadmin/utils/models.py`

所有业务模型必须继承 `CoreModel`，自动拥有审计字段和通用方法。

### 字段定义

```python
class CoreModel(models.Model):
    id = models.BigAutoField(primary_key=True, help_text="Id", verbose_name="Id")
    description = models.CharField(max_length=255, verbose_name="描述", null=True, blank=True)
    creator = models.ForeignKey(
        to=settings.AUTH_USER_MODEL, 
        related_query_name='creator_query', 
        null=True,
        verbose_name='创建人', 
        on_delete=models.SET_NULL,
        db_constraint=False
    )
    modifier = models.CharField(max_length=255, null=True, blank=True, help_text="修改人")
    dept_belong_id = models.CharField(max_length=255, help_text="数据归属部门", null=True, blank=True)
    update_datetime = models.DateTimeField(auto_now=True, null=True, blank=True, verbose_name="修改时间")
    create_datetime = models.DateTimeField(auto_now_add=True, null=True, blank=True, verbose_name="创建时间")
    
    objects = CoreModelManager()       # 默认管理器（自动过滤删除数据）
    all_objects = models.Manager()      # 原始管理器
```

### 排除字段

以下字段在序列化/导出时默认排除：

```python
exclude_fields = [
    '_state', 'pk', 'id',
    'create_datetime', 'update_datetime',
    'creator', 'creator_id', 'creator_pk', 'creator_name',
    'modifier', 'modifier_id', 'modifier_pk', 'modifier_name',
    'dept_belong_id',
]
```

### 常用方法

```python
# 将模型转化为字典（排除 exclude_fields）
data = instance.to_data()

# 导出用（另一种字典格式）
data = instance.to_dict_data()

# 插入新记录（自动填充审计字段）
new_instance = MyModel().insert(request)

# 更新记录（自动填充审计字段）
instance.update(request, update_data={"name": "新名称"})

# 获取当前登录用户
user = instance.get_request_user(request)

# 获取当前登录用户 ID
user_id = instance.get_request_user_id(request)
```

---

## 二、使用示例

### 定义业务模型

```python
from dvadmin.utils.models import CoreModel, table_prefix

class MyModel(CoreModel):
    name = models.CharField(max_length=64, verbose_name="名称")
    status = models.BooleanField(default=True, verbose_name="状态")
    sort = models.IntegerField(default=1, verbose_name="排序")
    
    class Meta:
        db_table = table_prefix + "my_module_my_model"   # 表名规范
        verbose_name = "我的模型"
        verbose_name_plural = verbose_name
        ordering = ("sort",)  # 默认排序
```

### 表名命名规范

```python
# 使用 table_prefix 前缀 + 模块名 + 表名
db_table = table_prefix + "system_users"      # dvadmin_system_users
db_table = table_prefix + "system_role"       # dvadmin_system_role
db_table = table_prefix + "myapp_product"     # dvadmin_myapp_product
```

---

## 三、模型管理器

### CoreModelManager

自定义管理器，提供以下功能：

```python
class CoreModelManager(models.Manager):
    def create(self, request=None, **kwargs):
        """创建时自动填充创建人、修改人、部门"""
        data = {**kwargs}
        if request:
            request_user = request.user
            data["creator"] = request_user
            data["modifier"] = request_user.id
            data["dept_belong_id"] = request_user.dept_id
        return super().create(**data)
```

---

## 四、工具函数

### 获取应用下的模型

```python
from dvadmin.utils.models import get_custom_app_models

# 获取所有自定义模型
all_models = get_custom_app_models()

# 获取指定应用下的模型
app_models = get_custom_app_models('dvadmin.system')
```

### 获取所有模型对象

```python
from dvadmin.utils.models import get_all_models_objects

# 获取所有模型对象
all_objects = get_all_models_objects()

# 获取指定模型
user_model = get_all_models_objects('Users')
```

---

## 五、开发规范

1. **必须继承 CoreModel**，不能直接继承 models.Model
2. **使用 table_prefix 前缀**，保持表名一致性
3. **为每个字段添加 verbose_name**，便于后台管理
4. **为每个字段添加 help_text**，提供字段说明
5. **设置 ordering**，确保列表页默认排序
6. **外键设置 db_constraint=False**，避免数据库级外键约束（框架约定）
