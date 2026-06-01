# 后端 Skill — 异常处理（Exception Handling）

> 本文档规范 django-vue3-admin 框架中统一的异常处理机制，确保所有异常都能被正确捕获并返回标准响应。

---

## 一、异常处理器

位置：`dvadmin/utils/exception.py`

### 配置

```python
# application/settings.py
REST_FRAMEWORK = {
    "EXCEPTION_HANDLER": "dvadmin.utils.exception.CustomExceptionHandler",
}
```

### 处理流程

```python
def CustomExceptionHandler(ex, context):
    response = exception_handler(ex, context)  # 先调用 DRF 默认处理
    
    if isinstance(ex, AuthenticationFailed):
        # 认证失败 → 401
        return ErrorResponse(msg=ex.detail, code=401)
    elif isinstance(ex, Http404):
        # 资源不存在 → 400
        return ErrorResponse(msg="接口地址不正确", code=400)
    elif isinstance(ex, DRFAPIException):
        # DRF 异常 → 回滚事务
        set_rollback()
        return ErrorResponse(msg=ex.detail, code=4000)
    elif isinstance(ex, ProtectedError):
        # 外键约束错误 → 4000
        set_rollback()
        return ErrorResponse(msg="删除失败:该条数据与其他数据有相关绑定", code=4000)
    elif isinstance(ex, Exception):
        # 未知异常 → 记录日志
        logger.exception(traceback.format_exc())
        return ErrorResponse(msg=str(ex), code=4000)
```

---

## 二、自定义异常

### CustomValidationError

位置：`dvadmin/utils/validator.py`

```python
from dvadmin.utils.validator import CustomValidationError

# 在序列化器中使用
class MySerializer(CustomModelSerializer):
    def validate_name(self, value):
        if len(value) < 3:
            raise CustomValidationError("名称长度不能少于3个字符")
        return value
```

**特点**：不暴露字段名，只返回纯错误信息文本

---

## 三、常见异常场景

### 1. 认证失败

| 异常 | 响应 | 处理 |
|------|------|------|
| Token 过期 | 401 | 前端跳转登录页 |
| Token 黑名单 | 401 | 前端跳转登录页 |
| 签发无效 | 401 | 前端跳转登录页 |

### 2. 数据校验失败

| 异常 | 响应 | 处理 |
|------|------|------|
| 字段必填 | 4000 | 前端表单提示 |
| 唯一性冲突 | 4000 | 前端表单提示 |
| 格式错误 | 4000 | 前端表单提示 |

### 3. 数据库异常

| 异常 | 响应 | 处理 |
|------|------|------|
| 外键约束 | 4000 | 提示数据关联 |
| 记录不存在 | 400 | 提示接口地址错误 |

---

## 四、在视图中使用异常

### 推荐写法

```python
from dvadmin.utils.json_response import ErrorResponse, DetailResponse

class MyViewSet(CustomModelViewSet):
    @action(methods=['POST'], detail=True)
    def do_something(self, request, pk=None):
        try:
            instance = self.get_object()
            result = some_risky_operation(instance)
            return DetailResponse(data=result, msg="操作成功")
        except SomeExpectedError as e:
            # 预期内的异常，直接返回给前端
            return ErrorResponse(msg=str(e), code=4000)
        except Exception as e:
            # 未预期的异常，记录日志后返回
            logger.error(f"操作失败: {str(e)}", exc_info=True)
            return ErrorResponse(msg="操作失败，请稍后重试", code=500)
```

### 避免的写法

```python
# 错误：直接抛出未处理的异常
def do_something(self, request):
    result = risky_operation()  # 可能抛出 500
    return DetailResponse(data=result)

# 错误：返回非标准格式
def do_something(self, request):
    return Response({"error": "出错了"}, status=400)  # 前端无法解析

# 错误：暴露敏感信息
def do_something(self, request):
    try:
        result = risky_operation()
    except Exception as e:
        return ErrorResponse(msg=str(e))  # 可能泄露数据库信息
```

---

## 五、事务回滚

### 自动回滚

DRF 在以下情况自动回滚事务：
- `APIException` 及其子类
- `认证失败`

### 手动回滚

```python
from rest_framework.views import set_rollback

def some_view(request):
    try:
        with transaction.atomic():
            # 一系列数据库操作
            if something_wrong:
                set_rollback()  # 手动标记回滚
                return ErrorResponse(msg="操作失败")
    except Exception as e:
        set_rollback()
        return ErrorResponse(msg=str(e))
```

---

## 六、开发规范

1. **预期异常直接处理**，不依赖全局异常处理器
2. **未预期异常记录日志**，便于排查问题
3. **不暴露敏感信息**，错误信息要友好
4. **使用 ErrorResponse 返回**，保持响应格式一致
5. **批量操作使用事务**，确保数据一致性
