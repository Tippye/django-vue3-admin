# 后端 Skill — 序列化器（Serializers）

> 本文档规范 django-vue3-admin 框架中序列化器的开发方法，核心类为 `CustomModelSerializer`，自动处理审计字段填充。

---

## 一、CustomModelSerializer 基类

位置：`dvadmin/utils/serializers.py`

### 基本结构

```python
from dvadmin.utils.serializers import CustomModelSerializer

class MySerializer(CustomModelSerializer):
    """
    自定义序列化器
    自动处理：创建人、修改人、部门归属等审计字段
    """
    creator_name = serializers.SlugRelatedField(slug_field="name", source="creator", read_only=True)
    modifier_name = serializers.SerializerMethodField(read_only=True)
    
    class Meta:
        model = MyModel
        fields = "__all__"           # 或指定字段列表
        read_only_fields = ["id"]    # 只读字段
```

### 自动提供的字段

```python
class CustomModelSerializer(DynamicFieldsMixin, ModelSerializer):
    # 修改人名称（通过 modifier 字段查询）
    modifier_name = serializers.SerializerMethodField(read_only=True)
    
    # 创建人名称
    creator_name = serializers.SlugRelatedField(
        slug_field="name", source="creator", read_only=True
    )
    
    # 时间字段格式化
    create_datetime = serializers.DateTimeField(
        format="%Y-%m-%d %H:%M:%S", required=False, read_only=True
    )
    update_datetime = serializers.DateTimeField(
        format="%Y-%m-%d %H:%M:%S", required=False, read_only=True
    )
```

---

## 二、序列化器类型

### 1. 列表/详情序列化器

用于 `list` 和 `retrieve` 操作：

```python
class MySerializer(CustomModelSerializer):
    dept_name = serializers.CharField(source='dept.name', read_only=True)
    
    class Meta:
        model = MyModel
        fields = "__all__"
        read_only_fields = ["id"]
```

### 2. 创建序列化器

用于 `create` 操作，可以添加字段校验：

```python
from dvadmin.utils.validator import CustomUniqueValidator

class MyCreateSerializer(CustomModelSerializer):
    name = serializers.CharField(
        max_length=50,
        validators=[
            CustomUniqueValidator(queryset=MyModel.objects.all(), message="名称必须唯一")
        ]
    )
    
    class Meta:
        model = MyModel
        fields = "__all__"
        read_only_fields = ["id"]
```

### 3. 更新序列化器

用于 `update` 操作，可以排除某些字段：

```python
class MyUpdateSerializer(CustomModelSerializer):
    class Meta:
        model = MyModel
        read_only_fields = ["id", "password"]  # 更新时不能修改
        fields = "__all__"
```

### 4. 导出序列化器

用于 `export_data` 操作，控制导出字段：

```python
class MyExportSerializer(CustomModelSerializer):
    status_display = serializers.CharField(source="get_status_display", read_only=True)
    
    class Meta:
        model = MyModel
        fields = ["name", "status_display", "create_datetime"]
```

### 5. 导入序列化器

用于 `import_data` 操作，处理导入数据：

```python
class MyImportSerializer(CustomModelSerializer):
    class Meta:
        model = MyModel
        exclude = ["user_permissions", "groups"]  # 排除字段
```

---

## 三、自动审计字段处理

### 创建时自动填充

```python
def create(self, validated_data):
    if self.request and str(self.request.user) != "AnonymousUser":
        validated_data["modifier"] = self.get_request_user_id()
        validated_data["creator"] = self.request.user
        validated_data["dept_belong_id"] = getattr(
            self.request.user, "dept_id", None
        )
    return super().create(validated_data)
```

### 更新时自动填充

```python
def update(self, instance, validated_data):
    if self.request and str(self.request.user) != "AnonymousUser":
        validated_data["modifier"] = self.get_request_user_id()
    return super().update(instance, validated_data)
```

---

## 四、常用方法

### 获取当前用户信息

```python
# 获取用户名
username = serializer.get_request_username()

# 获取用户姓名
name = serializer.get_request_name()

# 获取用户 ID
user_id = serializer.get_request_user_id()
```

### 错误信息中文化

`CustomModelSerializer` 会自动将字段名转换为中文 `verbose_name`：

```python
@property
def errors(self):
    errors = super().errors
    verbose_errors = {}
    # 将字段名转换为中文名称
    fields = {field.name: field.verbose_name for field in self.Meta.model._meta.get_fields()}
    for field_name, error in errors.items():
        if field_name in fields:
            verbose_errors[str(fields[field_name])] = error
        else:
            verbose_errors[field_name] = error
    return verbose_errors
```

---

## 五、开发规范

1. **必须继承 CustomModelSerializer**，不使用原生 ModelSerializer
2. **创建/更新/列表分离序列化器**，不同场景使用不同的序列化器
3. **read_only_fields 必须包含 id**，防止前端修改主键
4. **唯一字段使用 CustomUniqueValidator**，保持错误信息中文化
5. **导出序列化器字段尽量少**，只包含导出必需字段
6. **导入序列化器处理特殊逻辑**（如密码加密、多对多字段）
