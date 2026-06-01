# 后端 Skill — 分页（Pagination）

> 本文档规范 django-vue3-admin 框架中分页组件的使用方法，核心类为 `CustomPagination`。

---

## 一、CustomPagination 配置

位置：`dvadmin/utils/pagination.py`

### 默认配置

```python
class CustomPagination(PageNumberPagination):
    page_size = 10                    # 默认每页条数
    page_size_query_param = "limit"   # 前端传参字段名
    max_page_size = 999               # 最大每页条数
    page_query_param = "page"         # 页码参数名
```

### 全局配置

```python
# application/settings.py
REST_FRAMEWORK = {
    "DEFAULT_PAGINATION_CLASS": "dvadmin.utils.pagination.CustomPagination",
}
```

---

## 二、响应格式

### 列表响应

```json
{
    "code": 2000,
    "msg": "获取成功",
    "page": 1,
    "limit": 10,
    "total": 100,
    "is_next": true,
    "is_previous": false,
    "data": [...]
}
```

### 字段说明

| 字段 | 说明 |
|------|------|
| `code` | 状态码，成功=2000 |
| `msg` | 提示信息 |
| `page` | 当前页码 |
| `limit` | 每页条数 |
| `total` | 总记录数 |
| `is_next` | 是否有下一页 |
| `is_previous` | 是否有上一页 |
| `data` | 数据列表 |

---

## 三、前端传参

### 分页参数

```http
GET /api/system/user/?page=1&limit=10
```

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `page` | int | 1 | 当前页码 |
| `limit` | int | 10 | 每页显示数量 |

### 排序参数

```http
# 指定字段排序
GET /api/system/user/?ordering=-create_datetime

# 多字段排序
GET /api/system/user/?ordering=status,-create_datetime
```

| 前缀 | 含义 |
|------|------|
| `无` | 正序（ASC） |
| `-` | 倒序（DESC） |

---

## 四、视图集中的分页

### 默认行为

所有继承 `CustomModelViewSet` 的视图集自动分页：

```python
class MyViewSet(CustomModelViewSet):
    queryset = MyModel.objects.all()
    serializer_class = MySerializer
    # 自动分页，无需额外配置
```

### 关闭分页

```python
class MyViewSet(CustomModelViewSet):
    pagination_class = None  # 关闭分页
```

### 自定义每页数量

```python
class MyViewSet(CustomModelViewSet):
    def get_queryset(self):
        queryset = super().get_queryset()
        # 当 limit=999 时相当于不分页
        return queryset
```

---

## 五、前端 FastCrud 分页适配

在 `web/src/settings.ts` 中配置了分页参数转换：

```typescript
commonOptions() {
    return {
        request: {
            transformQuery: ({page, form, sort}: any) => {
                if (sort.asc !== undefined) {
                    form['ordering'] = `${sort.asc ? '' : '-'}${sort.prop}`;
                }
                return {page: page.currentPage, limit: page.pageSize, ...form};
            },
            transformRes: ({res}: any) => {
                return {
                    records: res.data, 
                    currentPage: res.page, 
                    pageSize: res.limit, 
                    total: res.total
                };
            },
        },
    };
}
```

---

## 六、开发规范

1. **默认开启分页**，不使用无分页的列表接口
2. **排序字段可配置**，允许前端通过 `ordering` 参数排序
3. **limit 不得超过 max_page_size**，防止一次性返回过多数据
4. **心跳接口不分页**，健康检查类接口关闭分页
