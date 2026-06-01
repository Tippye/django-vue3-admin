# 后端 Skill — 统一响应（JSON Response）

> 本文档规范 django-vue3-admin 框架中统一的 JSON 响应格式，确保前后端数据交互一致。

---

## 一、响应类

位置：`dvadmin/utils/json_response.py`

### 响应类对照

| 类名 | 用途 | 使用场景 |
|------|------|---------|
| `SuccessResponse` | 列表响应（带分页） | list、multiple_delete |
| `DetailResponse` | 单条数据响应 | create、retrieve、update、destroy、action |
| `ErrorResponse` | 错误响应 | 异常、参数错误、权限不足 |

---

## 二、响应格式

### 成功响应 — 列表（SuccessResponse）

```python
from dvadmin.utils.json_response import SuccessResponse

class MyViewSet(CustomModelViewSet):
    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())
        page = self.paginate_queryset(queryset)
        serializer = self.get_serializer(page, many=True)
        return self.get_paginated_response(serializer.data)
        # 自动返回：
        # {
        #     "code": 2000,
        #     "msg": "获取成功",
        #     "page": 1,
        #     "limit": 10,
        #     "total": 100,
        #     "data": [...]
        # }
```

### 成功响应 — 单条（DetailResponse）

```python
from dvadmin.utils.json_response import DetailResponse

class MyViewSet(CustomModelViewSet):
    def retrieve(self, request, *args, **kwargs):
        instance = self.get_object()
        serializer = self.get_serializer(instance)
        return DetailResponse(data=serializer.data, msg="获取成功")
        # 自动返回：
        # {
        #     "code": 2000,
        #     "data": {...},
        #     "msg": "获取成功"
        # }
```

### 错误响应（ErrorResponse）

```python
from dvadmin.utils.json_response import ErrorResponse

class MyViewSet(CustomModelViewSet):
    def create(self, request, *args, **kwargs):
        if not request.data.get('name'):
            return ErrorResponse(msg="名称不能为空", code=4000)
        # 自动返回：
        # {
        #     "code": 4000,
        #     "data": null,
        #     "msg": "名称不能为空"
        # }
```

---

## 三、响应参数

### SuccessResponse / DetailResponse

```python
SuccessResponse(
    data=None,      # 响应数据
    msg='success',  # 提示信息
    page=1,         # 当前页码（SuccessResponse 专用）
    limit=1,        # 每页数量（SuccessResponse 专用）
    total=1         # 总记录数（SuccessResponse 专用）
)

DetailResponse(
    data=None,      # 响应数据
    msg='success'   # 提示信息
)
```

### ErrorResponse

```python
ErrorResponse(
    data=None,      # 响应数据（通常为 None）
    msg='error',    # 错误信息
    code=400        # 错误状态码，默认 400
)
```

---

## 四、状态码对照表

| 状态码 | 含义 | 说明 |
|--------|------|------|
| 2000 | 操作成功 | 正常响应 |
| 400 | 请求参数错误 | 参数校验失败 |
| 4000 | 业务逻辑错误 | 自定义错误 |
| 401 | 认证失败 | Token 过期或无效 |
| 403 | 权限不足 | 没有接口或按钮权限 |
| 404 | 资源不存在 | 接口地址错误 |
| 500 | 服务器内部错误 | 异常未捕获 |

---

## 五、前端对接

前端在 `web/src/utils/service.ts` 中对响应进行统一处理：

```typescript
service.interceptors.response.use(
    (response) => {
        const dataAxios = response.data;
        const { code } = dataAxios;
        
        switch (code) {
            case 2000:
                return dataAxios;      // 正常返回
            case 401:
                Session.clear();        // 清除登录状态
                window.location.href = '/';
                return Promise.reject(dataAxios);
            case 4000:
                ElMessage.error(dataAxios.msg);
                return Promise.reject(dataAxios);
            default:
                return Promise.reject(dataAxios);
        }
    }
);
```

---

## 六、开发规范

1. **使用统一响应类**，不直接返回 `Response`
2. **错误响应必须包含 msg**，方便前端展示
3. **不暴露敏感信息**，错误信息不要包含堆栈跟踪
4. **状态码保持一致**，不自定义状态码范围
