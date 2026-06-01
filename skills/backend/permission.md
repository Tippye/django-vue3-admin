# 后端 Skill — 权限控制（Permission）

> 本文档规范 django-vue3-admin 框架的权限体系，包括接口权限、数据权限、字段权限等核心能力。

---

## 一、权限体系架构

```
用户(Users)
  → 角色(Role) 
    → 菜单权限(RoleMenuPermission)
    → 按钮/接口权限(RoleMenuButtonPermission) → 数据权限范围
    → 字段权限(FieldPermission) → 字段级别控制
```

### 数据权限范围

| 值 | 含义 | 说明 |
|-----|------|------|
| 0 | 仅本人数据权限 | 只能看到自己创建的数据 |
| 1 | 本部门及以下数据权限 | 包含下级部门的数据 |
| 2 | 本部门数据权限 | 仅当前部门 |
| 3 | 全部数据权限 | 不受限制 |
| 4 | 自定义数据权限 | 指定特定部门 |

---

## 二、CustomPermission 接口权限

位置：`dvadmin/utils/permission.py`

### 工作原理

1. 超级管理员（`is_superuser=True`）拥有所有权限
2. 普通用户需要通过角色获取接口权限
3. 支持接口白名单（ApiWhiteList）

### 配置接口白名单

```python
# 在管理后台配置，或直接插入数据
ApiWhiteList.objects.create(
    url="/api/public/some-api/",
    method=0,  # 0=GET, 1=POST, 2=PUT, 3=DELETE
    enable_datasource=True  # 是否启用数据权限过滤
)
```

### 视图级权限覆盖

```python
from rest_framework.permissions import AllowAny, IsAuthenticated

class MyViewSet(CustomModelViewSet):
    permission_classes = [CustomPermission]  # 默认

    @action(methods=['GET'], detail=False, permission_classes=[AllowAny])
    def public_api(self, request):
        """公开接口，无需登录"""
        return SuccessResponse(data="公开数据")

    @action(methods=['GET'], detail=False, permission_classes=[IsAuthenticated])
    def private_api(self, request):
        """需要登录但不需要特定权限"""
        return SuccessResponse(data="私有数据")
```

---

## 三、数据权限过滤

位置：`dvadmin/utils/filters.py`（`DataLevelPermissionsFilter`）

### 自动应用

数据权限过滤器通过 `extra_filter_class` 自动应用到所有 `CustomModelViewSet`：

```python
class CustomModelViewSet(ModelViewSet):
    extra_filter_class = [CoreModelFilterBankend, DataLevelPermissionsFilter]
```

### 工作流程

1. 检查接口白名单（关闭数据权限的接口直接返回）
2. 超级管理员直接返回全部数据
3. 获取用户部门 ID
4. 检查数据是否有 `dept_belong_id` 字段
5. 根据角色的数据权限范围过滤

### 局部开关

在某些视图中不需要数据权限：

```python
class MyViewSet(CustomModelViewSet):
    extra_filter_class = [CoreModelFilterBankend]  # 去掉 DataLevelPermissionsFilter
```

---

## 四、字段权限（列权限）

位置：`dvadmin/utils/field_permission.py`

### 模型定义

```python
class MenuField(CoreModel):
    model = models.CharField(max_length=64, verbose_name='表名')
    menu = models.ForeignKey(to='Menu', on_delete=models.CASCADE, verbose_name='菜单')
    field_name = models.CharField(max_length=64, verbose_name='模型表字段名')
    title = models.CharField(max_length=64, verbose_name='字段显示名')

class FieldPermission(CoreModel):
    role = models.ForeignKey(to='Role', on_delete=models.CASCADE, verbose_name='角色')
    field = models.ForeignKey(to='MenuField', on_delete=models.CASCADE, verbose_name='字段')
    is_query = models.BooleanField(default=True, verbose_name='是否可查询')
    is_create = models.BooleanField(default=True, verbose_name='是否可创建')
    is_update = models.BooleanField(default=True, verbose_name='是否可更新')
```

### 使用方法

1. 在菜单管理中定义表字段（MenuField）
2. 在角色管理中分配字段权限（FieldPermission）
3. 视图集自动过滤未授权字段

### 视图集接入

```python
from dvadmin.utils.field_permission import FieldPermissionMixin

class MyViewSet(CustomModelViewSet, FieldPermissionMixin):
    """
    自动支持字段权限控制
    自动提供 field_permission 接口
    """
    serializer_class = MySerializer
```

---

## 五、开发规范

1. **默认使用 CustomPermission**，不随意使用 AllowAny
2. **公开接口加入白名单**，不要在视图中直接使用 AllowAny
3. **数据权限按需关闭**，确保感知到数据权限的影响范围
4. **字段权限提前规划**，在需要细粒度控制的页面提前定义好字段
