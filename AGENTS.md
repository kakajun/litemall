# AGENTS.md

This file provides guidance to Qoder (qoder.com) when working with code in this repository.

## 项目概述

litemall 是一个开源小商场系统，采用前后端分离架构：
- Spring Boot 后端（多模块 Maven 项目）
- Vue 管理后台前端
- 微信小程序用户前端（litemall-wx 和 renard-wx 两套）
- Vue 用户移动端（litemall-vue）

## 技术栈

- **后端**：Spring Boot 2.1.5, MyBatis, MySQL 8, Druid, Shiro, JWT, Swagger
- **管理后台前端**：Vue 2 + Element UI + vue-element-admin
- **移动端**：Vue 2 + Vant
- **微信小程序**：原生小程序（litemall-wx）和另一套 UI（renard-wx）
- **构建工具**：Maven（后端）, npm（前端）

## 项目模块结构

```
litemall/                          # 根 POM
├── litemall-core/                 # 核心模块：工具类、配置、存储、通知、快递查询
├── litemall-db/                   # 数据层：MyBatis domain/mapper/service，含代码生成器
├── litemall-wx-api/               # 小程序/移动端 REST API（端口 8080）
├── litemall-admin-api/            # 管理后台 REST API（端口 8083）
├── litemall-all/                  # 聚合启动模块，打包可执行 jar，内嵌前后端静态资源
├── litemall-all-war/              # WAR 打包模块
├── litemall-admin/                # 管理后台 Vue 前端
├── litemall-vue/                  # 移动端 Vue 前端
├── litemall-wx/                   # 微信小程序前端（原生）
├── renard-wx/                     # 另一套微信小程序前端
└── litemall-db/sql/               # 数据库初始化脚本
```

### 后端模块依赖关系

- `litemall-db`：无内部依赖，提供所有数据表对应的 domain、mapper、service
- `litemall-core`：依赖 `litemall-db`，提供系统配置、存储服务、通知服务、快递查询、工具类等
- `litemall-wx-api`：依赖 `litemall-core` 和 `litemall-db`，提供小程序/移动端接口
- `litemall-admin-api`：依赖 `litemall-core` 和 `litemall-db`，提供管理后台接口
- `litemall-all`：聚合所有后端模块，作为统一启动入口

## 常用命令

### 后端（Maven）

```bash
# 安装所有模块到本地仓库
mvn install

# 打包（跳过测试，根 POM 默认 maven.test.skip=true）
mvn clean package

# 运行单体服务（包含所有 API）
java -Dfile.encoding=UTF-8 -jar litemall-all/target/litemall-all-0.1.0-exec.jar

# 运行管理后台 API 服务（单独）
cd litemall-admin-api
mvn spring-boot:run

# 运行小程序 API 服务（单独）
cd litemall-wx-api
mvn spring-boot:run

# 运行 litemall-db 模块的 MyBatis 代码生成器
cd litemall-db
mvn mybatis-generator:generate
```

### 管理后台前端（litemall-admin）

```bash
cd litemall-admin
npm install
npm run dev          # 开发服务器，端口 9527，代理 /admin 到 localhost:8080
npm run build        # 生产构建
npm run lint         # ESLint 检查
```

### 移动端前端（litemall-vue）

```bash
cd litemall-vue
npm install
npm run dev          # 开发服务器，端口 6255，代理 /wx 到 localhost:8080
npm run build        # 生产构建
npm run lint         # ESLint 检查
```

### 微信小程序

使用微信开发者工具导入 `litemall-wx` 或 `renard-wx` 目录，需开启"不校验合法域名"。

## 数据库初始化

依次导入 `litemall-db/sql/` 下的文件：
1. `litemall_schema.sql` — 创建数据库
2. `litemall_table.sql` — 创建表结构
3. `litemall_data.sql` — 初始化数据

## 架构要点

### 统一响应格式

所有 REST API 返回统一 JSON 格式：
```json
{ "errno": 0, "errmsg": "成功", "data": {} }
```

使用 `org.linlinjava.litemall.core.util.ResponseUtil` 构造响应。错误码规范：
- `0`：成功
- `4xx`：前端错误（401 参数缺失，402 参数值错误）
- `5xx`：后端/系统错误（501 未登录，502 内部错误，503 未实现，504 乐观锁失效，505 更新失败，506 无权限）
- `6xx`：管理后台业务错误（见 `AdminResponseCode`）
- `7xx`：小程序业务错误（见 `WxResponseCode`）

### 认证与权限

- **管理后台**：基于 Apache Shiro 的 Session 认证，使用 `@RequiresPermissions` 和自定义的 `@RequiresPermissionsDesc(menu, button)` 注解控制权限。登录接口返回 Cookie-based Session。
- **小程序/移动端**：基于 JWT Token 认证，登录后返回 token，后续请求在 Header 中携带。使用 `@LoginUser` 注解在 Controller 方法参数中注入当前登录用户 ID。

### 数据层（litemall-db）

- 使用 MyBatis Generator 自动生成 domain（带 Example 查询类）、Mapper 接口和 XML。
- 自定义插件支持逻辑删除（`deleted` 字段）、单条查询、选择性返回等增强功能。
- JSON 数组字段（如 `gallery`, `specifications`, `pic_urls`）通过自定义 `JsonStringArrayTypeHandler` / `JsonIntegerArrayTypeHandler` 映射为 Java 数组。
- 手动编写的复杂查询在 `OrderMapper` 和 `StatMapper` 中。

### 核心服务（litemall-core）

- **SystemConfig**：系统配置缓存，从 `litemall_system` 表加载。
- **StorageService**：文件存储抽象，支持本地、阿里云 OSS、腾讯云 COS、七牛云。
- **NotifyService**：通知服务，支持邮件、阿里云短信、腾讯云短信。
- **ExpressService**：快递查询服务。
- **TaskService**：基于 Spring Scheduling 的定时任务管理。

### 静态资源打包

`litemall-all` 模块通过 `maven-resources-plugin` 在构建时将 `litemall-admin/dist` 和 `litemall-vue/dist` 复制到 `target/classes/static` 和 `target/classes/static/vue` 下，实现前后端一体化部署。

## 开发注意事项

- Java 版本为 1.8，Spring Boot 版本为 2.1.5，根 POM 默认跳过测试（`maven.test.skip=true`）。
- 管理后台前端代理 `/admin` 到 `localhost:8080`，移动端代理 `/wx` 到 `localhost:8080`。
- 单体启动时管理后台和小程序 API 共用同一端口（8080），独立启动时管理后台默认 8083、小程序默认 8080。
- 微信小程序登录、支付等功能需要配置真实的微信 AppID 和密钥才能正常工作，开发环境下可跳过。
