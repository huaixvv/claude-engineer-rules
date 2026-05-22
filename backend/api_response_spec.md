# 后端 API 响应封装规范

## **Critical**: 以下规则只针对新项目，或没有响应封装的项目设计，如果是老项目，已经有响应封装请忽略。

> **版本**：v1.0
> **适用范围**：所有后端服务
> **参考实现**：TypeScript（**Critical**: 其他语言按本规范严格落地）

---

## 一、设计原则

| # | 原则                                                                                         |
| - | -------------------------------------------------------------------------------------------- |
| 1 | **HTTP 状态码归 HTTP**：成功 200，失败用语义化状态码（401/403/404/422/500），遵循 REST |
| 2 | **响应体统一封装**：永远是 `{ code, msg, data }` 三件套                              |
| 3 | **业务码用字符串字面量**：如 `USER_NOT_FOUND`，禁止数字（`10001`）                 |
| 4 | **错误字典化**：每项目维护一份 `Err` 字典，`code + status + msg` 强绑定            |
| 5 | **零魔法字符串**：业务代码通过对象引用调用错误码                                       |
| 6 | **零硬编码**：所有响应从字典或默认值派生                                               |

---

## 二、Wire Protocol（响应契约）

### 2.1 响应体结构

```
{
  code: string    // "0" = 成功 | "USER_NOT_FOUND" 等业务错误码
  msg:  string    // 成功默认 "success" | 失败为字典中文消息
  data: any       // 业务数据 | 错误详情 | null
}
```

### 2.2 成功响应

```
HTTP/1.1 200 OK
{ "code": "0", "msg": "success", "data": <资源 | null> }
```

### 2.3 失败响应

```
HTTP/1.1 <语义化状态码>
{ "code": "<错误码>", "msg": "<中文消息>", "data": <错误详情 | null> }
```

---

## 三、HTTP 状态码策略

| 场景                              | HTTP |
| --------------------------------- | ---- |
| 业务成功                          | 200  |
| 鉴权失败（无 token / token 失效） | 401  |
| 无权限                            | 403  |
| 资源不存在                        | 404  |
| 业务冲突                          | 409  |
| 参数校验失败                      | 422  |
| 限流                              | 429  |
| 服务器错误 / 未捕获异常           | 500  |

**规则**：HTTP 状态码只表达粗粒度协议层语义，业务细分由 `code` 字段表达。

---

## 四、CRUD 动词响应对照表

| 方法                | 场景     | HTTP | `data`              |
| ------------------- | -------- | ---- | --------------------- |
| `GET /users`      | 列表     | 200  | `Paginated<T>` 结构 |
| `GET /users/1`    | 单资源   | 200  | 资源对象              |
| `POST /users`     | 创建     | 200  | 新建的资源对象        |
| `PUT /users/1`    | 全量更新 | 200  | 更新后的资源对象      |
| `PATCH /users/1`  | 部分更新 | 200  | 更新后的资源对象      |
| `DELETE /users/1` | 删除     | 200  | `null`              |

---

## 五、列表 / 分页标准结构（强制）

任何列表查询接口，`data` 必须遵循，字段命名强制约定：

```typescript
interface Paginated<T> {
  list:       T[]              // 数据列表
  total:       number           // 总数
  nextCursor?: string | null    // 游标分页用（推荐大数据集）
  page?:       number           // 页码分页用
  pageSize?:   number           // 页码分页用
}
```

### 示例

```json
// 游标分页
{
  "code": "0", "msg": "success",
  "data": { "list": [], "total": 1000, "nextCursor": "abc" }
}

// 页码分页
{
  "code": "0", "msg": "success",
  "data": { "list": [], "total": 100, "page": 1, "pageSize": 20 }
}
```

---

## 六、错误码字典规范

### 6.1 命名规则

全大写 + 下划线分隔.

示例：USER_NOT_FOUND, ORDER_PAY_FAILED, TOKEN_EXPIRED, EMAIL_ALREADY_EXISTS

### 6.2 字段名 = code 值（强制）

字典字段名必须与 `code` 字符串值完全一致：

```typescript
// ✅ 正确
USER_NOT_FOUND: { code: "USER_NOT_FOUND", status: 404, msg: "用户不存在" }

// ❌ 错误
UserNotFound:   { code: "USER_NOT_FOUND", ... }   // 字段名与 code 不一致
```

**其他语言同样要求**（即使违反默认 lint 风格，也要保持字段名与 code 值完全一致，换取跨字段对应）

### 6.3 基础字典（所有项目必备）

| code                   | HTTP | msg            |
| ---------------------- | ---- | -------------- |
| `UNAUTHORIZED`       | 401  | 未授权         |
| `TOKEN_EXPIRED`      | 401  | 登录已过期     |
| `FORBIDDEN`          | 403  | 无权访问       |
| `RESOURCE_NOT_FOUND` | 404  | 资源不存在     |
| `CONFLICT`           | 409  | 资源冲突       |
| `VALIDATION_FAILED`  | 422  | 校验失败       |
| `RATE_LIMITED`       | 429  | 请求过于频繁   |
| `INTERNAL_ERROR`     | 500  | 服务器内部错误 |

业务错误码由各项目按需扩展。

### 6.4 `VALIDATION_FAILED` 的 data 约定

`data` 为字段错误数组：

```typescript
data: Array<{
  field: string
  rule:  string
  msg:   string
}>
```

示例：

```json
{
  "code": "VALIDATION_FAILED",
  "msg": "校验失败",
  "data": [
    { "field": "email", "rule": "format", "msg": "邮箱格式不正确" },
    { "field": "age",   "rule": "min",    "msg": "年龄需大于等于 18" }
  ]
}
```

---

## 七、命名空间设计

统一在 `Api` 命名空间下，包含四个成员：

| 成员          | 作用                               |
| ------------- | ---------------------------------- |
| `Api.Err`   | 错误字典（对象引用，零魔法字符串） |
| `Api.ok`    | 成功响应工厂                       |
| `Api.fail`  | 抛错工厂（生成异常实例）           |
| `Api.Error` | 异常类（供 instanceof 判断）       |

---

## 八、结构定义

### 8.1 `Spec`（错误码规约）

```typescript
interface Spec {
  code:   string
  status: number   // HTTP 状态码
  msg:    string   // 中文消息
}
```

### 8.2 `Body<T>`（响应体）

```typescript
interface Body<T> {
  code: string
  msg:  string
  data: T | null
}
```

### 8.3 `Err`（错误字典）

```typescript
const Err = {
  [字段名: 同 code 值]: Spec
}
```

### 8.4 `AppError`（异常类）

```typescript
class AppError extends Error {
  code:   string   // 来自 Spec.code
  status: number   // 来自 Spec.status
  msg:    string   // 来自 Spec.msg
  data:   unknown  // 调用方注入的错误详情
}
```

---

## 九、接口签名（使用契约）

### 9.1 成功工厂

```typescript
Api.ok<T>(
  data?: T | null,    // 默认 null
  code?: string,      // 默认 "0"
  msg?:  string,      // 默认 "success"
): Body<T>
```

### 9.2 抛错工厂

```typescript
Api.fail(
  spec: Spec,         // 来自 Api.Err 字典
  data?: unknown,     // 默认 null
): AppError
```

---

## 十、业务调用范式

### 10.1 成功响应

```typescript
Api.ok()              // 删除等无数据场景
Api.ok(user)          // 最常见：返回资源
Api.ok(list)          // 列表场景（list 是 Paginated<T> 结构）
```

### 10.2 失败响应

```typescript
throw Api.fail(Api.Err.USER_NOT_FOUND)
throw Api.fail(Api.Err.VALIDATION_FAILED, [
  { field: "email", rule: "format", msg: "邮箱格式不正确" }
])
```

### 10.3 异常处理（中间件层）

```typescript
if (err instanceof Api.Error) {
  setHttpStatus(err.status)
  setBody({ code: err.code, msg: err.msg, data: err.data })
} else {
  setHttpStatus(500)
  setBody({ code: "INTERNAL_ERROR", msg: "服务器内部错误", data: null })
}
```


## 十一、前端使用契约（参考）

前端应基于 Wire Protocol 实现统一拦截，封装axios等拦截器：

```
on success (HTTP 2xx):
  返回 response.data.data        // 直接取业务数据

on failure (HTTP 4xx/5xx):
  body = response.data           // { code, msg, data }
  if body.code == "401": logout()
  else if body.code !== 200: toast(body.msg)
  reject(body)
```

---

## 十二、跨语言落地要求

其他语言（Go / Python / Rust 等）实施本规范时必须保证：

1. **Wire Protocol 完全一致**（响应体字段名、HTTP 状态码、`code` 值）
2. **命名空间统一**（包/模块名为 `api`，挂载 `Err` / `Ok` / `Fail` / `AppError`）
3. **错误字典字段名 = code 值**
4. **列表分页字段命名一致**（`list` / `total` / `nextCursor` / `page` / `pageSize`）
5. **业务调用风格对称**：
   - 成功：`api.ok(data)` 或等价写法
   - 失败：`api.fail(api.Err.XXX)` 或等价写法

实现细节（如可选参数模拟方式等）按语言惯用法处理。

---