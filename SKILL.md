# django-vue3-admin 项目二次开发 Skill 总览

> 本文档为基于 django-vue3-admin 开源框架的二次开发总指南，覆盖项目架构、核心功能、代码规范与扩展方法。后续基于本项目开发时，应同时关联对应的后端与前端 skill 子文档以获取更细节的规范。

---

## 一、项目概览

| 维度 | 说明 |
|------|------|
| **后端** | Django 4.x + Django REST Framework + SimpleJWT，支持 RBAC 权限、列级别权限、数据权限 |
| **前端** | Vue 3 + TypeScript + Vite + Element Plus + FastCrud + Pinia |
| **数据库** | 默认 SQLite，生产环境推荐 MySQL 8.0+ |
| **缓存/任务** | Redis（可选），Celery（可选） |
| **插件系统** | 支持插件化扩展，前后端均可插揔 |

### 项目目录结构

```
django-vue3-admin/
├── backend/                    # 后端工程
│   ├── application/            # Django 主工程
│   │   ├── settings.py         # 全局配置
│   │   ├── urls.py             # 根路由
│   │   ├── dispatch.py         # 系统配置/字典初始化
│   │   ├── celery.py           # Celery 配置
│   │   └── asgi.py / wsgi.py   # 应用入口
│   ├── dvadmin/                # 主应用
│   │   ├── system/             # 核心系统模块
│   │   │   ├── models.py       # 全部模型定义
│   │   │   ├── views/          # 各类视图集
│   │   │   ├── tasks.py        # 异步任务
│   │   │   └── signals.py      # 信号处理
│   │   └── utils/              # 工具类（核心 infra）
│   │       ├── models.py         # CoreModel / SoftDeleteModel
│   │       ├── viewset.py        # CustomModelViewSet
│   │       ├── serializers.py    # CustomModelSerializer
│   │       ├── permission.py     # 权限控制
│   │       ├── filters.py        # 自定义过滤器
│   │       ├── pagination.py     # 自定义分页
│   │       ├── json_response.py  # 统一响应格式
│   │       ├── exception.py      # 异常处理
│   │       ├── middleware.py     # 中间件
│   │       └── ...               # 其他工具
│   ├── plugins/                # 插件目录
│   ├── logs/                   # 日志目录
│   ├── conf/                   # 环境配置
│   └── manage.py               # 命令入口
├── web/                        # 前端工程
│   ├── src/
│   │   ├── views/              # 页面视图
│   │   ├── components/         # 公共组件
│   │   ├── stores/             # Pinia 状态管理
│   │   ├── router/             # 路由配置
│   │   ├── utils/              # 工具函数
│   │   ├── api/                # 接口请求
│   │   ├── plugin/permission/  # 权限插件
│   │   ├── directive/          # 自定义指令
│   │   └── i18n/               # 国际化
│   └── vite.config.ts          # Vite 配置
└── docker-compose.yml          # Docker 部署
```

---

## 二、核心功能模块

### 1. 权限体系（RBAC）

本框架实现了完整的基于角色的访问控制，包含五个层级：

- **菜单权限**：控制用户能看到哪些导航菜单
- **按钮权限**：控制界面上的新增、编辑、删除等按钮
- **接口权限**：控制 API 接口的访问
- **数据权限**：控制用户能看到哪些数据（本人/部门/全部/自定义）
- **字段权限**（列权限）：控制列表页面显示哪些字段

### 2. FastCrud 快速开发

前端基于 `@fast-crud/fast-crud` 实现的增删改查快速开发，通过 `crud.tsx` 配置文件定义列表和表单，无需写大量模板代码。

### 3. 插件系统

支持通过插件机制扩展功能，后端插件放在 `backend/plugins/` 目录，前端插件通过扫描自动注册。

### 4. 字典/系统配置

通过后端管理界面维护字典和系统配置，支持缓存到内存或 Redis，前端通过 `dictionary()` 函数获取字典数据。

---

## 三、二次开发规范

### 后端开发

1. **新增模型**：继承 `CoreModel`，自动拥有审计字段（creator、modifier、create_datetime 等）
2. **新增序列化器**：继承 `CustomModelSerializer`，自动处理审计字段填充
3. **新增视图集**：继承 `CustomModelViewSet`，自动获得 CRUD 接口、分页、过滤、权限控制
4. **表名规范**：使用 `table_prefix + "模块名_表名"` 的命名规则
5. **插件开发**：参考 `plugins/code_info/` 示例插件的目录结构和配置方式

### 前端开发

1. **新增页面**：按照 `views/模块名/页面名/` 的结构创建目录
2. **CRUD 页面**：包含 `index.vue` + `api.ts` + `crud.tsx` 三个文件
3. **API 封装**：所有接口请求放在 `api.ts` 中，统一使用 `request()` 或 `service()` 发起
4. **权限指令**：使用 `v-auth="'permission:code'"` 控制按钮显隐

---

## 四、子 Skill 文档导航

| 文件路径 | 说明 |
|---------|------|
| `skills/backend/tasks.md` | Celery 异步任务开发规范 |
| `skills/backend/logs.md` | 日志系统配置与使用规范 |
| `skills/backend/models.md` | 基础模型（CoreModel）开发规范 |
| `skills/backend/viewset.md` | 视图集（CustomModelViewSet）开发规范 |
| `skills/backend/serializers.md` | 序列化器（CustomModelSerializer）开发规范 |
| `skills/backend/permission.md` | 权限控制（RBAC）开发规范 |
| `skills/backend/filters.md` | 自定义过滤器开发规范 |
| `skills/backend/pagination.md` | 分页配置开发规范 |
| `skills/backend/json_response.md` | 统一响应格式规范 |
| `skills/backend/exception.md` | 异常处理规范 |
| `skills/frontend/fastcrud.md` | FastCrud 快速开发规范 |
| `skills/frontend/api.md` | API 接口封装规范 |
| `skills/frontend/component.md` | 组件开发规范 |

---

## 五、快速入门检查表

在基于本框架进行二次开发前，请先确认以下准备工作：

- [ ] 已安装 Python 3.9+ 、Node.js 16+ 、MySQL 8.0+
- [ ] 已复制 `backend/conf/env.example.py` 为 `env.py` 并配置数据库
- [ ] 已执行 `python manage.py migrate` 和 `python manage.py init`
- [ ] 已安装前端依赖并启动开发服务器

---

## 六、重要链接

- **官方文档**：https://www.django-vue-admin.com
- **FastCrud 文档**：http://fast-crud.docmirror.cn
- **DRF 文档**：https://www.django-rest-framework.org
- **SimpleJWT 文档**：https://django-rest-framework-simplejwt.readthedocs.io
