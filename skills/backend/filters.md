# 后端 Skill — 过滤器（Filters）

> 本文档规范 django-vue3-admin 框架中自定义过滤器的使用方法，包括时间范围过滤、数据级权限过滤、字段查询等。

---

## 一、过滤器组成

| 过滤器 | 文件 | 说明 |
|--------|------|------|
| `CoreModelFilterBankend` | `dvadmin/utils/filters.py` | 创建/更新时间范围过滤 |
| `DataLevelPermissionsFilter` | `dvadmin/utils/filters.py` | 数据级权限过滤 |
| `CustomDjangoFilterBackend` | `dvadmin/utils/filters.py` | 字段查询过滤（支持模糊匹配） |

---

## 二、时间范围过滤

### 支持的查询参数

```http
GET /api/system/user/?create_datetime_after=2024-01-01&create_datetime_before=2024-12-31
GET /api/system/user/?update_datetime_after=2024-01-01&update_datetime_before=2024-12-31
```

| 参数 | 说明 | 格式 |
|------|------|------|
| `create_datetime_after` | 创建时间开始 | `YYYY-MM-DD` |
| `create_datetime_before` | 创建时间结束 | `YYYY-MM-DD` |
| `update_datetime_after` | 更新时间开始 | `YYYY-MM-DD` |
| `update_datetime_before` | 更新时间结束 | `YYYY-MM-DD` |

### 自动处理逻辑

```python
class CoreModelFilterBankend(BaseFilterBackend):
    def filter_queryset(self, request, queryset, view):
        # 同时指定开始和结束时间
        if create_datetime_after and create_datetime_before:
            queryset = queryset.filter(
                create_datetime__gte=create_datetime_after,
                create_datetime__lte=f'{create_datetime_before} 23:59:59'
            )
        # 只指定开始时间
        elif create_datetime_after:
            queryset = queryset.filter(create_datetime__gte=create_datetime_after)
        # 只指定结束时间
        elif create_datetime_before:
            queryset = queryset.filter(create_datetime__lte=f'{create_datetime_before} 23:59:59')
        return queryset
```

---

## 三、字段查询过滤

### 默认行为

`CustomDjangoFilterBackend` 默认对所有字段开启过滤，并自动将 `CharField` 转换为模糊查询（`icontains`）：

```python
class MyViewSet(CustomModelViewSet):
    filter_fields = '__all__'  # 对所有字段开启过滤
```

### 自定义过滤字段

```python
class MyViewSet(CustomModelViewSet):
    filter_fields = ['name', 'status', 'create_datetime']  # 只对指定字段
```

### 查询前缀

支持通过前缀指定匹配方式：

| 前缀 | 含义 | 示例 |
|------|------|------|
| `^` | 以... 开始 | `name^=abc` → `name__istartswith=abc` |
| `~` | 包含 | `name~=abc` → `name__icontains=abc` |
| `=` | 精确匹配 | `name==abc` → `name__iexact=abc` |

### 范围查询

```http
GET /api/system/user/?create_datetime_after=2024-01-01&create_datetime_before=2024-12-31
```

---

## 四、自定义过滤器

### 定义过滤器类

```python
from rest_framework.filters import BaseFilterBackend

class MyCustomFilter(BaseFilterBackend):
    def filter_queryset(self, request, queryset, view):
        param = request.query_params.get('my_param')
        if param:
            queryset = queryset.filter(my_field=param)
        return queryset
```

### 注入视图集

```python
class MyViewSet(CustomModelViewSet):
    extra_filter_class = [
        CoreModelFilterBankend,
        DataLevelPermissionsFilter,
        MyCustomFilter  # 自定义过滤器
    ]
```

---

## 五、开发规范

1. **过滤字段尽量完整**，方便前端筛选
2. **必要时限制过滤字段**，避免敏感字段被查询
3. **自定义过滤器握住性能**，避免复杂的数据库查询
4. **前端传参时注意空值处理**，后端会过滤掉空值参数
