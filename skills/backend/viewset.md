# 后端 Skill — 视图集（ViewSet）

> 本文档规范 django-vue3-admin 框架中视图集的开发方法，核心类为 `CustomModelViewSet`，继承后自动获得 CRUD 接口、分页、过滤、权限控制等能力。

---

## 一、CustomModelViewSet 基类

位置：`dvadmin/utils/viewset.py`

### 基本结构

```python
from dvadmin.utils.viewset import CustomModelViewSet

class MyViewSet(CustomModelViewSet):
    """
    自定义视图集
    list: 查询列表
    create: 新增
    update: 修改
    retrieve: 单例查询
    destroy: 删除
    """
    queryset = MyModel.objects.all()
    serializer_class = MySerializer
    create_serializer_class = MyCreateSerializer      # 创建时使用
    update_serializer_class = MyUpdateSerializer      # 更新时使用
    filter_fields = ['name', 'status']                # 过滤字段
    search_fields = ['name', 'description']           # 搜索字段
```

### 类属性说明

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `queryset` | QuerySet | 必填 | 数据查询集 |
| `serializer_class` | Serializer | 必填 | 默认序列化器 |
| `create_serializer_class` | Serializer | None | 创建操作使用的序列化器 |
| `update_serializer_class` | Serializer | None | 更新操作使用的序列化器 |
| `filter_fields` | list/str | `'__all__'` | 支持过滤的字段 |
| `search_fields` | tuple | () | 支持搜索的字段 |
| `ordering_fields` | str | `'__all__'` | 支持排序的字段 |
| `extra_filter_class` | list | `[CoreModelFilterBankend, DataLevelPermissionsFilter]` | 额外过滤器 |
| `permission_classes` | list | `[CustomPermission]` | 权限类 |

---

## 二、自动提供的接口

继承 `CustomModelViewSet` 后，自动获得以下接口：

| 方法 | 请求方式 | 路径 | 说明 |
|------|---------|------|------|
| `list` | GET | `/api/module/model/` | 列表查询（支持分页、过滤、排序） |
| `create` | POST | `/api/module/model/` | 新增数据 |
| `retrieve` | GET | `/api/module/model/{id}/` | 单条查询 |
| `update` | PUT | `/api/module/model/{id}/` | 更新数据 |
| `destroy` | DELETE | `/api/module/model/{id}/` | 删除数据 |
| `multiple_delete` | DELETE | `/api/module/model/multiple_delete/` | 批量删除 |
| `get_by_ids` | POST | `/api/module/model/get_by_ids/` | 通过 ID 列表查询 |

---

## 三、自定义行为

### 使用 @action 注解

```python
from rest_framework.decorators import action
from dvadmin.utils.json_response import DetailResponse, SuccessResponse

class MyViewSet(CustomModelViewSet):
    queryset = MyModel.objects.all()
    serializer_class = MySerializer

    @action(methods=['GET'], detail=False)
    def my_custom_action(self, request):
        """
        自定义接口（无需实例 ID）
        GET /api/module/model/my_custom_action/
        """
        data = self.get_queryset().filter(status=True)
        serializer = self.get_serializer(data, many=True)
        return SuccessResponse(data=serializer.data, msg="获取成功")

    @action(methods=['POST'], detail=True)
    def toggle_status(self, request, pk=None):
        """
        自定义接口（需要实例 ID）
        POST /api/module/model/{id}/toggle_status/
        """
        instance = self.get_object()
        instance.status = not instance.status
        instance.save()
        return DetailResponse(data=None, msg="状态切换成功")
```

### action 参数说明

| 参数 | 说明 |
|------|------|
| `methods` | 允许的 HTTP 方法列表，如 `['GET', 'POST']` |
| `detail` | `False`=集合级接口，`True`=单例接口 |
| `permission_classes` | 可覆盖默认权限类 |
| `url_path` | 自定义 URL 路径（默认为方法名） |
| `url_name` | URL 反向解析名称 |

---

## 四、批量操作

### 批量删除

内置支持批量删除，请求格式：

```http
DELETE /api/module/model/multiple_delete/
Content-Type: application/json

{
    "keys": [1, 2, 3]
}
```

### 批量创建

`CustomModelViewSet` 支接收列表格式的请求体自动触发批量创建：

```http
POST /api/module/model/
Content-Type: application/json

[
    {"name": "数据1", "status": true},
    {"name": "数据2", "status": false}
]
```

---

## 五、响应格式

### 成功响应

```json
// 列表响应
{
    "code": 2000,
    "msg": "获取成功",
    "page": 1,
    "limit": 10,
    "total": 100,
    "data": [...]
}

// 单条响应
{
    "code": 2000,
    "msg": "获取成功",
    "data": {...}
}
```

### 错误响应

```json
{
    "code": 4000,
    "msg": "错误消息",
    "data": null
}
```

---

## 六、开发规范

1. **必须继承 CustomModelViewSet**，不使用原生 ModelViewSet
2. **分离序列化器**：创建/更新/列表使用不同的 serializer_class
3. **过滤字段尽量完整**，方便前端筛选
4. **搜索字段覆盖常用字段**，支持模糊搜索
5. **自定义行为需添加文档字符串**，说明接口功能
