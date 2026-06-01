# 前端 Skill — FastCrud 快速开发

> 本文档规范 django-vue3-admin 框架中 FastCrud 的使用方法，FastCrud 是前端的核心 CRUD 开发框架，通过配置形式实现增删改查。

---

## 一、文件结构

每个 CRUD 页面通常包含以下文件：

```
views/模块/页面/
├── index.vue       # 页面组件
├── api.ts          # API 接口封装
└── crud.tsx        # FastCrud 配置
```

---

## 二、index.vue 组件结构

### 基本模板

```vue
<template>
  <fs-page>
    <fs-crud ref="crudRef" v-bind="crudBinding">
      <!-- 自定义插槽 -->
      <template #actionbar-right>
        <el-button @click="handleCustom"> 自定义按钮 </el-button>
      </template>
    </fs-crud>
  </fs-page>
</template>

<script lang="ts" setup name="页面名称">
import { ref, onMounted } from 'vue';
import { useFs } from '@fast-crud/fast-crud';
import { createCrudOptions } from './crud';

const { crudBinding, crudRef, crudExpose } = useFs({ 
  createCrudOptions 
});

// 页面打开后获取列表数据
onMounted(() => {
  crudExpose.doRefresh();
});

// 自定义方法
const handleCustom = () => {
  console.log('自定义操作');
};
</script>
```

### 关键组件

| 组件 | 说明 |
|------|------|
| `<fs-page>` | 页面容器，自动处理布局 |
| `<fs-crud>` | CRUD 核心组件，接受 crudBinding 配置 |
| `<fs-table>` | 表格组件（内部自动渲染） |
| `<fs-form>` | 表单组件（内部自动渲染） |

---

## 三、crud.tsx 配置

### 基本结构

```tsx
import * as api from './api';
import { 
  dict, UserPageQuery, AddReq, DelReq, EditReq,
  CreateCrudOptionsProps, CreateCrudOptionsRet 
} from '@fast-crud/fast-crud';

export const createCrudOptions = function (
  { crudExpose }: CreateCrudOptionsProps
): CreateCrudOptionsRet {
  // 基本请求
  const pageRequest = async (query: UserPageQuery) => {
    return await api.GetList(query);
  };
  const editRequest = async ({ form, row }: EditReq) => {
    form.id = row.id;
    return await api.UpdateObj(form);
  };
  const delRequest = async ({ row }: DelReq) => {
    return await api.DelObj(row.id);
  };
  const addRequest = async ({ form }: AddReq) => {
    return await api.AddObj(form);
  };

  return {
    crudOptions: {
      request: { pageRequest, addRequest, editRequest, delRequest },
      columns: {
        // 列配置
      }
    }
  };
};
```

---

## 四、列配置（columns）

### 基本字段配置

```tsx
columns: {
  _index: {
    title: '序号',
    form: { show: false },
    column: {
      type: 'index',
      align: 'center',
      width: '70px',
      columnSetDisabled: true,
    },
  },
  name: {
    title: '名称',
    search: { show: true },  // 可搜索
    type: 'input',
    column: { minWidth: 120 },
    form: {
      rules: [{ required: true, message: '名称必填' }],
      component: { placeholder: '请输入名称' },
    },
  },
  status: {
    title: '状态',
    type: 'dict-radio',
    column: {
      component: {
        name: 'fs-dict-switch',
        activeText: '',
        inactiveText: '',
      },
    },
    dict: dict({
      data: dictionary('button_status_bool'),
    }),
  },
}
```

### 常用列属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `title` | string | 列标题 |
| `type` | string | 字段类型（input、dict-select、dict-radio 等） |
| `search.show` | boolean | 是否显示在搜索区 |
| `form.show` | boolean | 是否显示在表单 |
| `form.rules` | array | 表单校验规则 |
| `column.show` | boolean | 是否显示在列表 |
| `column.width` | number | 列宽度 |
| `column.minWidth` | number | 最小列宽 |
| `dict` | object | 字典配置 |

### 常用字段类型

| 类型 | 说明 | 示例 |
|------|------|------|
| `input` | 文本输入 | `用户名、备注` |
| `password` | 密码输入 | `登录密码` |
| `number` | 数字输入 | `排序、数量` |
| `dict-select` | 下拉选择 | `状态、分类` |
| `dict-radio` | 单选框 | `性别、是/否` |
| `dict-checkbox` | 多选框 | `权限、标签` |
| `dict-switch` | 开关 | `启用/禁用` |
| `datetime` | 日期时间 | `创建时间` |
| `date` | 日期 | `生日` |
| `dict-tree` | 树形选择 | `部门、分类` |
| `avatar-uploader` | 头像上传 | `头像` |
| `file-uploader` | 文件上传 | `附件` |
| `editor` | 富文本编辑器 | `内容` |

---

## 五、字典配置（dict）

### 静态字典

```tsx
import { dictionary } from '/@/utils/dictionary';

columns: {
  gender: {
    title: '性别',
    type: 'dict-select',
    dict: dict({
      data: dictionary('gender'),  // 从系统字典获取
    }),
  },
}
```

### 动态字典

```tsx
columns: {
  dept: {
    title: '部门',
    type: 'dict-tree',
    dict: dict({
      isTree: true,
      url: '/api/system/dept/all_dept/',
      value: 'id',
      label: 'name',
    }),
  },
  role: {
    title: '角色',
    type: 'dict-select',
    dict: dict({
      url: '/api/system/role/',
      value: 'id',
      label: 'name',
    }),
  },
}
```

### 字典数据格式

```javascript
// 字典数据格式
[
  { label: '选项1', value: '1', color: 'success' },
  { label: '选项2', value: '2', color: 'danger' },
]
```

---

## 六、操作栏配置

### 顶部操作栏（actionbar）

```tsx
crudOptions: {
  actionbar: {
    buttons: {
      add: {
        show: auth('module:Create'),  // 权限控制
      },
      export: {
        text: '导出',
        show: auth('module:Export'),
        click: (ctx: any) => {
          // 自定义导出逻辑
        },
      },
    },
  },
}
```

### 行操作（rowHandle）

```tsx
crudOptions: {
  rowHandle: {
    fixed: 'right',  // 固定在右侧
    width: 200,
    buttons: {
      view: {
        show: false,  // 隐藏查看按钮
      },
      edit: {
        show: auth('module:Update'),
      },
      remove: {
        show: auth('module:Delete'),
      },
      custom: {
        text: '自定义',
        type: 'text',
        show: auth('module:Custom'),
        click: (ctx: any) => {
          const { row } = ctx;
          console.log('自定义按钮', row);
        },
      },
    },
  },
}
```

---

## 七、权限控制

### 按钮级权限

```tsx
import { auth } from '/@/utils/authFunction';

// 控制按钮显隐
buttons: {
  add: { show: auth('user:Create') },
  edit: { show: auth('user:Update') },
  remove: { show: auth('user:Delete') },
}
```

### 指令级权限

```vue
<!-- 模板中使用 -->
<template>
  <div v-auth="'user:Create'">
    只有拥有 user:Create 权限的用户才能看到
  </div>
</template>
```

---

## 八、开发规范

1. **每个 CRUD 页面必须有 index.vue + api.ts + crud.tsx**
2. **列配置尽量完整**，包含搜索、表单、列表三个角度
3. **按钮权限必须配置**，避免无权限用户误操作
4. **表单校验尽量完善**，前端后端双重校验
5. **字典使用系统字典**，保持一致性
