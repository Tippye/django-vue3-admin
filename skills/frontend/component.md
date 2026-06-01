# 前端 Skill — 组件开发

> 本文档规范 django-vue3-admin 框架中组件的开发方法，包括公共组件、自定义组件、指令、状态管理等。

---

## 一、组件目录结构

```
web/src/
├── components/           # 公共组件
│   ├── auth/               # 权限组件
│   │   ├── auth.vue       # 单个权限判断
│   │   ├── authAll.vue    # 全部权限判断
│   │   └── auths.vue      # 多个权限判断
│   ├── editor/           # 富文本组件
│   ├── importExcel/      # Excel 导入组件
│   ├── svgIcon/          # SVG 图标组件
│   └── table/            # 表格组件
├── layout/               # 布局组件
│   ├── component/        # 布局子组件
│   ├── navBars/          # 导航栏
│   ├── navMenu/          # 菜单
│   └── routerView/       # 路由视图
├── views/                # 页面组件
│   ├── system/           # 系统管理页面
│   └── plugins/          # 插件页面
├── stores/               # 状态管理
├── directive/            # 自定义指令
└── plugin/permission/    # 权限插件
```

---

## 二、公共组件

### 组件列表

| 组件 | 位置 | 用途 |
|------|------|------|
| `auth.vue` | `components/auth/` | 权限判断（单个） |
| `authAll.vue` | `components/auth/` | 权限判断（全部需满足） |
| `auths.vue` | `components/auth/` | 权限判断（任意一个） |
| `editor` | `components/editor/` | 富文本编辑器 |
| `importExcel` | `components/importExcel/` | Excel 导入 |
| `svgIcon` | `components/svgIcon/` | SVG 图标 |
| `table` | `components/table/` | 封装表格 |

### 使用示例

```vue
<template>
  <div>
    <!-- 权限组件 -->
    <auth auth="user:Create">
      <el-button>新增</el-button>
    </auth>
    
    <!-- SVG 图标 -->
    <SvgIcon name="iconfont icon-shouye" />
    
    <!-- Excel 导入 -->
    <importExcel api="api/system/user/"> 导入 </importExcel>
  </div>
</template>
```

---

## 三、自定义组件规范

### 单文件组件

```vue
<template>
  <div class="my-component">
    {{ title }}
  </div>
</template>

<script lang="ts" setup name="my-component">
import { defineProps } from 'vue';

const props = defineProps<{
  title: string;
}>();
</script>

<style scoped>
.my-component {
  padding: 16px;
}
</style>
```

### 多文件组件

```
components/组件名/
├── index.vue        # 组件入口
├── types.ts         # 类型定义
└── api.ts           # 组件内部接口（如需要）
```

---

## 四、自定义指令

### 指令目录

```
directive/
├── index.ts           # 指令注册
├── authDirective.ts   # 权限指令
├── customDirective.ts # 自定义指令
└── sizeDirective.ts   # 尺寸指令
```

### 权限指令（v-auth）

```vue
<template>
  <!-- 单个权限 -->
  <el-button v-auth="'user:Create'">新增</el-button>
  
  <!-- 多个权限（任意一个） -->
  <div v-auths="['user:Create', 'user:Update']">
    创建或更新
  </div>
  
  <!-- 全部权限 -->
  <div v-auth-all="['user:Create', 'user:Delete']">
    同时拥有创建和删除
  </div>
</template>
```

### 注册指令

```typescript
// directive/index.ts
import { authDirective } from './authDirective';

export function directive(app: App) {
  app.directive('auth', authDirective);
}
```

---

## 五、状态管理（Pinia）

### Store 目录

```
stores/
├── index.ts                # Pinia 实例
├── interface/index.ts      # 类型定义
├── themeConfig.ts          # 主题配置
├── userInfo.ts             # 用户信息
├── routesList.ts           # 路由列表
├── frontendMenu.ts         # 前端菜单
├── keepAliveNames.ts       # 缓存页面
├── tagsViewRoutes.ts       # TagsView 路由
├── messageCenter.ts        # 消息中心
├── dictionary.ts           # 字典数据
├── systemConfig.ts         # 系统配置
└── ...                     # 其他状态
```

### 定义 Store

```typescript
// stores/myStore.ts
import { defineStore } from 'pinia';

export const useMyStore = defineStore('myStore', {
  state: () => ({
    count: 0,
    list: [] as any[],
  }),
  getters: {
    doubleCount: (state) => state.count * 2,
  },
  actions: {
    increment() {
      this.count++;
    },
    async fetchList() {
      const res = await getListApi();
      this.list = res.data;
    },
  },
});
```

### 使用 Store

```vue
<script lang="ts" setup>
import { useMyStore } from '/@/stores/myStore';
import { storeToRefs } from 'pinia';

const myStore = useMyStore();
const { count, list } = storeToRefs(myStore);  // 响应式引用

const handleClick = () => {
  myStore.increment();
};
</script>
```

---

## 六、权限插件

### 插件位置

```
plugin/permission/
├── index.ts               # 插件入口
├── directive.permission.ts  # 指令实现
├── func.permission.ts       # 函数实现
└── store.permission.ts      # 状态实现
```

### 函数式权限

```typescript
import { auth } from '/@/utils/authFunction';

// 判断单个权限
const hasCreate = auth('user:Create');  // true/false

// 判断多个权限（任意）
const hasSome = auths(['user:Create', 'user:Update']);

// 判断多个权限（全部）
const hasAll = authAll(['user:Create', 'user:Update']);
```

---

## 七、国际化

### 使用方式

```vue
<template>
  <div>{{ t('message.home.home') }}</div>
</template>

<script lang="ts" setup>
import { useI18n } from 'vue-i18n';
const { t } = useI18n();
</script>
```

### 添加语言包

```typescript
// i18n/lang/zh-cn.ts
export default {
  message: {
    home: {
      home: '首页',
    },
  },
};
```

---

## 八、开发规范

1. **组件名必须使用 kebab-case** 或 **PascalCase**
2. **单文件组件放在 components/ 下**，多文件组件放在独立目录
3. **组件必须定义 name**，便于调试和缓存
4. **Props 必须定义类型**，使用 TypeScript 接口
5. **自定义指令必须注册到 directive/index.ts**
6. **Store 使用 composition API 风格**，避免过多嵌套
