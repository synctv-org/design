# 24. API 文档生成方案

---

## 目录

- [1. 技术选型](#1-技术选型)
- [2. utoipa 集成](#2-utoipa-集成)
- [3. 文档组织](#3-文档组织)
- [4. 在线文档服务](#4-在线文档服务)
- [5. 最佳实践](#5-最佳实践)

---

## 1. 技术选型

### 1.1 方案对比

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| **utoipa** | - 编译时类型安全<br>- 与Axum完美集成<br>- 自动生成OpenAPI 3.0<br>- 活跃维护 | - 学习曲线稍陡 | ⭐⭐⭐⭐⭐ |
| **aide** | - 更现代的API<br>- 支持更多框架 | - 社区较小<br>- 文档较少 | ⭐⭐⭐⭐ |
| **rweb** | - 简单易用 | - 仅支持warp框架<br>- 不支持Axum | ⭐⭐⭐ |
| **手写OpenAPI** | - 完全自定义 | - 维护成本高<br>- 容易与代码不同步 | ⭐⭐ |

**结论**: 使用 **utoipa** (最成熟、最适合Axum)

### 1.2 utoipa 优势

- ✅ **编译时验证**: 代码变更后,文档自动更新
- ✅ **类型安全**: 请求/响应类型自动生成schema
- ✅ **零成本**: 不影响运行时性能
- ✅ **与Axum集成**: 专门的 `utoipa-axum` 支持
- ✅ **OpenAPI 3.0**: 标准化,兼容所有工具
- ✅ **活跃维护**: 2024年仍在积极开发

---

## 2. utoipa 集成

### 2.1 依赖配置

```toml
[dependencies]
# API 文档生成
utoipa = { version = "5.0", features = ["axum_extras", "chrono", "uuid"] }
utoipa-axum = "0.1"
utoipa-swagger-ui = { version = "8.0", features = ["axum"] }
utoipa-redoc = { version = "4.0", features = ["axum"] }
utoipa-rapidoc = { version = "4.0", features = ["axum"] }
```

### 2.2 基本使用

```rust
use utoipa::{OpenApi, ToSchema};

// 1. 定义 API 模型
#[derive(Serialize, Deserialize, ToSchema)]
pub struct User {
    /// 用户ID
    #[schema(example = "a1b2c3d4e5f6")]
    pub id: String,  // nanoid(12)

    /// 用户名 (3-20字符)
    #[schema(example = "alice", min_length = 3, max_length = 20)]
    pub username: String,

    /// 用户角色
    #[schema(example = "user")]
    pub role: UserRole,

    /// 创建时间
    #[schema(example = "2024-01-30T10:00:00Z")]
    pub created_at: DateTime<Utc>,
}

// 2. 定义 API 路由 (使用 utoipa::path 宏)
#[utoipa::path(
    get,
    path = "/api/user/me",
    tag = "User",
    summary = "获取当前用户信息",
    responses(
        (status = 200, description = "成功", body = User),
        (status = 401, description = "未认证", body = ErrorResponse),
    ),
    security(
        ("bearer_auth" = [])
    )
)]
pub async fn get_current_user(
    Extension(user): Extension<User>,
) -> Json<User> {
    Json(user)
}

// 3. 注册路由和文档
#[derive(OpenApi)]
#[openapi(
    info(
        title = "SyncTV API",
        version = "1.0.0",
        description = "多人实时视频同步观看平台 API",
        contact(
            name = "SyncTV Team",
            email = "support@synctv.example.com",
        ),
        license(
            name = "MIT",
        ),
    ),
    servers(
        (url = "https://synctv.example.com", description = "生产环境"),
        (url = "http://localhost:8080", description = "本地开发"),
    ),
    paths(
        get_current_user,
        // ... 其他路由
    ),
    components(
        schemas(User, UserRole, ErrorResponse)
    ),
    tags(
        (name = "User", description = "用户管理"),
        (name = "Room", description = "房间管理"),
        (name = "Movie", description = "视频管理"),
        (name = "Admin", description = "管理员功能"),
    ),
    security(
        ("bearer_auth" = [])
    )
)]
pub struct ApiDoc;

// 4. 集成到 Axum
use utoipa_swagger_ui::SwaggerUi;

pub fn create_app() -> Router {
    let api_doc = ApiDoc::openapi();

    Router::new()
        // API 路由
        .route("/api/user/me", get(get_current_user))
        // Swagger UI
        .merge(SwaggerUi::new("/swagger-ui")
            .url("/api-docs/openapi.json", api_doc.clone()))
        // Redoc UI (可选)
        .merge(Redoc::with_url("/redoc", api_doc))
}
```

---

## 3. 文档组织

### 3.1 按模块分组

```rust
// user/api_doc.rs
#[derive(OpenApi)]
#[openapi(
    paths(
        login_user,
        register_user,
        get_current_user,
        update_username,
        // ...
    ),
    components(schemas(User, UserRole, LoginRequest, RegisterRequest)),
    tags((name = "User", description = "用户管理"))
)]
pub struct UserApiDoc;

// room/api_doc.rs
#[derive(OpenApi)]
#[openapi(
    paths(
        create_room,
        get_room_info,
        join_room,
        // ...
    ),
    components(schemas(Room, RoomSettings, CreateRoomRequest)),
    tags((name = "Room", description = "房间管理"))
)]
pub struct RoomApiDoc;

// 合并所有模块的文档
pub fn merge_api_docs() -> utoipa::openapi::OpenApi {
    use utoipa::openapi::OpenApiBuilder;

    let mut api_doc = ApiDoc::openapi();

    // 合并用户模块
    api_doc.merge(UserApiDoc::openapi());

    // 合并房间模块
    api_doc.merge(RoomApiDoc::openapi());

    // 合并电影模块
    api_doc.merge(MovieApiDoc::openapi());

    // 合并管理员模块
    api_doc.merge(AdminApiDoc::openapi());

    api_doc
}
```

### 3.2 完整的 API 示例

```rust
/// 用户登录
#[utoipa::path(
    post,
    path = "/api/user/login",
    tag = "User",
    summary = "用户登录",
    description = "使用用户名/邮箱和密码登录",
    request_body(
        content = LoginRequest,
        description = "登录请求",
        example = json!({
            "username": "alice",
            "password": "MySecurePassword123"
        })
    ),
    responses(
        (
            status = 200,
            description = "登录成功",
            body = LoginResponse,
            example = json!({
                "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
                "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
                "expires_in": 3600,
                "user": {
                    "id": "a1b2c3d4e5f6",
                    "username": "alice",
                    "role": "user"
                }
            })
        ),
        (
            status = 400,
            description = "请求参数错误",
            body = ErrorResponse,
            example = json!({
                "code": "INVALID_REQUEST",
                "message": "用户名或密码不能为空"
            })
        ),
        (
            status = 401,
            description = "用户名或密码错误",
            body = ErrorResponse,
            example = json!({
                "code": "INVALID_CREDENTIALS",
                "message": "用户名或密码错误"
            })
        ),
        (
            status = 403,
            description = "用户被封禁或待审核",
            body = ErrorResponse,
            example = json!({
                "code": "USER_BANNED",
                "message": "您的账号已被封禁"
            })
        ),
    ),
)]
pub async fn login_user(
    State(state): State<AppState>,
    Json(req): Json<LoginRequest>,
) -> Result<Json<LoginResponse>> {
    // 实现...
}

/// 登录请求
#[derive(Debug, Serialize, Deserialize, ToSchema)]
pub struct LoginRequest {
    /// 用户名或邮箱
    #[schema(example = "alice", min_length = 3)]
    pub username: String,

    /// 密码
    #[schema(example = "MySecurePassword123", min_length = 8)]
    pub password: String,
}

/// 登录响应
#[derive(Debug, Serialize, ToSchema)]
pub struct LoginResponse {
    /// 访问令牌 (有效期1小时)
    #[schema(example = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...")]
    pub access_token: String,

    /// 刷新令牌 (有效期30天)
    #[schema(example = "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...")]
    pub refresh_token: String,

    /// 过期时间 (秒)
    #[schema(example = 3600)]
    pub expires_in: i64,

    /// 用户信息
    pub user: UserInfo,
}
```

### 3.3 认证配置

```rust
use utoipa::openapi::security::{SecurityScheme, HttpAuthScheme, HttpBuilder};

impl ApiDoc {
    pub fn configure_security() -> utoipa::openapi::OpenApi {
        let mut api_doc = Self::openapi();

        // 添加 Bearer Token 认证
        let security_scheme = SecurityScheme::Http(
            HttpBuilder::new()
                .scheme(HttpAuthScheme::Bearer)
                .bearer_format("JWT")
                .build()
        );

        api_doc
            .components
            .as_mut()
            .unwrap()
            .add_security_scheme("bearer_auth", security_scheme);

        api_doc
    }
}
```

---

## 4. 在线文档服务

### 4.1 Swagger UI (推荐)

**特点**: 最流行、功能完整、可交互测试

```rust
use utoipa_swagger_ui::SwaggerUi;

let app = Router::new()
    .merge(SwaggerUi::new("/swagger-ui")
        .url("/api-docs/openapi.json", api_doc.clone())
        .config(Config::new(["/api-docs/openapi.json"]))
    );
```

**访问**: `http://localhost:8080/swagger-ui`

### 4.2 Redoc (可选)

**特点**: 三栏布局、更美观、只读

```rust
use utoipa_redoc::Redoc;

let app = Router::new()
    .merge(Redoc::with_url("/redoc", api_doc.clone()));
```

**访问**: `http://localhost:8080/redoc`

### 4.3 RapiDoc (可选)

**特点**: 现代化设计、高度可定制

```rust
use utoipa_rapidoc::RapiDoc;

let app = Router::new()
    .merge(RapiDoc::new("/api-docs/openapi.json")
        .path("/rapidoc")
        .config(RapiDocConfig::new()));
```

**访问**: `http://localhost:8080/rapidoc`

### 4.4 同时提供多个UI

```rust
pub fn create_api_docs_routes(api_doc: utoipa::openapi::OpenApi) -> Router {
    Router::new()
        // Swagger UI (主推荐)
        .merge(SwaggerUi::new("/swagger-ui")
            .url("/api-docs/openapi.json", api_doc.clone()))

        // Redoc (备选)
        .merge(Redoc::with_url("/redoc", api_doc.clone()))

        // RapiDoc (备选)
        .merge(RapiDoc::new("/api-docs/openapi.json").path("/rapidoc"))

        // 原始 OpenAPI JSON
        .route("/api-docs/openapi.json", get(|| async move {
            Json(api_doc)
        }))
}
```

---

## 5. 最佳实践

### 5.1 使用示例值

```rust
#[derive(ToSchema)]
pub struct CreateRoomRequest {
    /// 房间名称 (1-50字符)
    #[schema(example = "周五电影之夜", min_length = 1, max_length = 50)]
    pub name: String,

    /// 房间密码 (可选,6-20字符)
    #[schema(example = "secret123", min_length = 6, max_length = 20)]
    pub password: Option<String>,

    /// 房间设置
    #[schema(example = json!({
        "hidden": false,
        "enable_guest": true
    }))]
    pub settings: RoomSettings,
}
```

### 5.2 枚举类型

```rust
#[derive(Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "lowercase")]
pub enum UserRole {
    /// 超级管理员
    Root,
    /// 平台管理员
    Admin,
    /// 普通用户
    User,
    /// 待审核用户
    Pending,
    /// 封禁用户
    Banned,
}
```

### 5.3 错误响应

```rust
#[derive(Serialize, ToSchema)]
pub struct ErrorResponse {
    /// 错误代码
    #[schema(example = "INVALID_REQUEST")]
    pub code: String,

    /// 错误消息
    #[schema(example = "请求参数错误")]
    pub message: String,

    /// 详细信息 (可选)
    #[schema(example = json!({
        "field": "username",
        "error": "用户名长度必须在3-20字符之间"
    }))]
    pub details: Option<serde_json::Value>,
}
```

### 5.4 分页响应

```rust
#[derive(Serialize, ToSchema)]
pub struct PaginatedResponse<T: ToSchema> {
    /// 总数
    #[schema(example = 100)]
    pub total: i64,

    /// 当前页码
    #[schema(example = 1)]
    pub page: i64,

    /// 每页数量
    #[schema(example = 20)]
    pub per_page: i64,

    /// 数据列表
    pub items: Vec<T>,
}
```

### 5.5 测试 API 文档

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_openapi_spec_valid() {
        let api_doc = ApiDoc::openapi();

        // 验证 OpenAPI 规范有效
        assert!(!api_doc.paths.is_empty());
        assert!(!api_doc.components.as_ref().unwrap().schemas.is_empty());

        // 输出到文件 (用于CI检查)
        let json = serde_json::to_string_pretty(&api_doc).unwrap();
        std::fs::write("openapi.json", json).unwrap();
    }

    #[test]
    fn test_all_routes_documented() {
        let api_doc = ApiDoc::openapi();

        // 检查所有路由都有文档
        let documented_paths: Vec<_> = api_doc.paths.paths.keys().collect();

        // 实际路由列表
        let actual_routes = vec![
            "/api/user/me",
            "/api/user/login",
            "/api/room/create",
            // ...
        ];

        for route in actual_routes {
            assert!(
                documented_paths.contains(&&route.to_string()),
                "Route {} is not documented",
                route
            );
        }
    }
}
```

### 5.6 CI/CD 集成

```yaml
# .github/workflows/api-docs.yml
name: API文档检查

on: [push, pull_request]

jobs:
  check-api-docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: 安装 Rust
        uses: actions-rs/toolchain@v1

      - name: 生成 OpenAPI 文档
        run: cargo test test_openapi_spec_valid

      - name: 验证 OpenAPI 规范
        uses: char0n/swagger-editor-validate@v1
        with:
          definition-file: openapi.json

      - name: 上传文档到GitHub Pages
        if: github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./api-docs
```

---

## 总结

本章节设计了一个**自动化、类型安全**的 API 文档生成方案:

✅ **自动生成**: 代码变更自动更新文档
✅ **类型安全**: 编译时验证,防止文档与代码不一致
✅ **标准化**: OpenAPI 3.0 标准,兼容所有工具
✅ **多种UI**: Swagger UI, Redoc, RapiDoc 任选
✅ **示例丰富**: 请求/响应示例,易于理解
✅ **CI/CD集成**: 自动验证和发布文档

**对比传统方案**:

- ❌ 手写文档: 容易过时,维护成本高
- ✅ utoipa: 代码即文档,零维护成本

---

**新增文档完成**: 17-24章 已全部创建
**下一步**: 修订现有 01-16 章文档

---

**上一章**: [17-数据流设计](./17-数据流设计.md)
**下一章**: [19-配置管理系统](./19-配置管理系统.md)
