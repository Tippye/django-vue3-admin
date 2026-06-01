# 前端 Skill — API 接口封装

> 本文档规范 django-vue3-admin 框架中前端 API 接口的封装方式，包括请求工具、响应处理、文件下载等。

---

## 一、请求工具

### 主要文件

| 文件 | 说明 |
|------|------|
| `web/src/utils/request.ts` | 基于 axios 的请求封装（旧版） |
| `web/src/utils/service.ts` | 基于 axios 的请求封装（新版，含抽象层） |

### 导出内容

```typescript
// service.ts 导出
export const service = createService();       // axios 实例
export const request = createRequestFunction(service);  // 封装后的请求方法
export const downloadFile = function({...});   // 文件下载
```

---

## 二、api.ts 文件规范

### 基本结构

```typescript
// views/模块/页面/api.ts
import { request, downloadFile } from '/@/utils/service';
import { PageQuery, AddReq, DelReq, EditReq, InfoReq } from '@fast-crud/fast-crud';

export const apiPrefix = '/api/模块/资源名/';

// 获取列表
export function GetList(query: PageQuery) {
  return request({
    url: apiPrefix,
    method: 'get',
    params: query,
  });
}

// 获取单条
export function GetObj(id: InfoReq) {
  return request({
    url: apiPrefix + id + '/',
    method: 'get',
  });
}

// 新增
export function AddObj(obj: AddReq) {
  return request({
    url: apiPrefix,
    method: 'post',
    data: obj,
  });
}

// 更新
export function UpdateObj(obj: EditReq) {
  return request({
    url: apiPrefix + obj.id + '/',
    method: 'put',
    data: obj,
  });
}

// 删除
export function DelObj(id: DelReq) {
  return request({
    url: apiPrefix + id + '/',
    method: 'delete',
    data: { id },
  });
}

// 导出
export function exportData(params: any) {
  return downloadFile({
    url: apiPrefix + 'export_data/',
    params: params,
    method: 'get',
  });
}
```

---

## 三、请求拦截器

### 请求拦截

```typescript
service.interceptors.request.use(
  (config) => {
    // 自动添加 JWT Token
    const token = Session.get('token');
    if (token != null) {
      config.headers.Authorization = 'JWT ' + token;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);
```

### 响应拦截

```typescript
service.interceptors.response.use(
  (response) => {
    const dataAxios = response.data;
    const { code } = dataAxios;
    
    switch (code) {
      case 2000:
        return dataAxios;                    // 成功
      case 401:
        Session.clear();
        ElMessageBox.alert('登录认证失败，请重新登录', '提示');
        return Promise.reject(dataAxios);
      case 4000:
        ElMessage.error(dataAxios.msg);      // 业务错误
        return Promise.reject(dataAxios);
      default:
        return Promise.reject(dataAxios);
    }
  },
  (error) => {
    const status = error.response?.status;
    switch (status) {
      case 400: error.message = '请求错误'; break;
      case 401: error.message = '登录授权过期'; break;
      case 403: error.message = '拒绝访问'; break;
      case 404: error.message = '请求地址出错'; break;
      case 500: error.message = '服务器内部错误'; break;
    }
    return Promise.reject(error);
  }
);
```

---

## 四、文件下载

### 普通下载

```typescript
import { downloadFile } from '/@/utils/service';

// 导出数据
export function exportData(params: any) {
  return downloadFile({
    url: '/api/system/user/export_data/',
    params: params,
    method: 'get',
  });
}
```

### Blob 下载

```typescript
request({
  url: '/api/system/user/export_data/',
  method: 'get',
  params: query,
  responseType: 'blob',  // 重要：设置响应类型为 blob
}).then((res: any) => {
  const blob = new Blob([res.data], { type: 'charset=utf-8' });
  const elink = document.createElement('a');
  elink.download = '导出文件.xlsx';
  elink.href = URL.createObjectURL(blob);
  document.body.appendChild(elink);
  elink.click();
  URL.revokeObjectURL(elink.href);
  document.body.removeChild(elink);
});
```

---

## 五、基础 URL 配置

### 环境变量

```
# web/.env.development
VITE_API_URL = 'http://127.0.0.1:8000/api'
```

### 动态获取

```typescript
import { getBaseURL } from '/@/utils/baseUrl';

// 获取 API 基础地址
const baseURL = getBaseURL();

// 拼接完整 URL
const fullUrl = getBaseURL('/media/avatar.jpg');
```

---

## 六、开发规范

1. **所有接口封装在 api.ts 中**，不在组件内直接写请求
2. **使用 apiPrefix 统一管理路径**，便于维护
3. **参数类型使用 FastCrud 提供的**（PageQuery、AddReq 等）
4. **错误处理统一在拦截器**，不在每个请求中重复处理
5. **文件下载使用 downloadFile 工具**，保持一致性
