# 20. CLI 命令设计

---

## 目录

- [1. 设计理念](#1-设计理念)
- [2. 命令树结构](#2-命令树结构)
- [3. 核心命令](#3-核心命令)
- [4. 交互式体验](#4-交互式体验)
- [5. 错误处理](#5-错误处理)
- [6. 实现方案](#6-实现方案)

---

## 1. 设计理念

### 1.1 设计目标

**简洁直观**: 命令结构清晰,易于记忆
**功能完整**: 覆盖所有管理操作
**友好反馈**: 清晰的错误提示和成功消息
**脚本友好**: 支持非交互式使用,输出格式化
**安全第一**: 危险操作需要确认

### 1.2 CLI 框架选择

使用 **Clap v4** (最新且功能强大的 Rust CLI 框架):

- 派生宏 (Derive) API - 声明式定义命令
- 自动生成帮助文档
- Shell 补全生成 (bash, zsh, fish, powershell)
- 彩色输出支持
- 子命令支持

```toml
[dependencies]
clap = { version = "4.4", features = ["derive", "color", "suggestions"] }
clap_complete = "4.4"  # Shell补全
dialoguer = "0.11"      # 交互式提示
console = "0.15"        # 彩色输出和进度条
indicatif = "0.17"      # 进度条
```

---

## 2. 命令树结构

```
synctv
├── server              # 启动服务器
├── admin               # 管理员管理
│   ├── add             # 添加管理员
│   ├── delete          # 删除管理员
│   └── list            # 列出所有管理员
├── root                # Root用户管理
│   ├── add             # 添加Root用户
│   ├── delete          # 删除Root用户
│   └── list            # 列出所有Root用户
├── user                # 用户管理
│   ├── list            # 列出用户
│   ├── search          # 搜索用户
│   ├── show            # 显示用户详情
│   ├── ban             # 封禁用户
│   ├── unban           # 解封用户
│   ├── delete          # 删除用户
│   └── approve         # 批准待审核用户
├── room                # 房间管理
│   ├── list            # 列出房间
│   ├── show            # 显示房间详情
│   ├── delete          # 删除房间
│   ├── ban             # 封禁房间
│   ├── unban           # 解封房间
│   └── approve         # 批准待审核房间
├── setting             # 设置管理
│   ├── list            # 列出所有设置
│   ├── show            # 显示设置详情
│   ├── set             # 修改设置
│   └── reset           # 重置为默认值
├── migrate             # 数据库迁移
│   ├── up              # 执行迁移
│   ├── down            # 回滚迁移
│   └── status          # 查看迁移状态
├── backup              # 备份与恢复
│   ├── create          # 创建备份
│   ├── restore         # 恢复备份
│   └── list            # 列出备份
├── completion          # 生成Shell补全脚本
└── version             # 显示版本信息
```

---

## 3. 核心命令

### 3.1 server - 启动服务器

**配置优先级**: CLI Args > Env > Config File > Database > Default

```bash
synctv server [OPTIONS]

OPTIONS:
    --host <HOST>                 监听地址 [default: 0.0.0.0]
-p, --port <PORT>                 监听端口 [default: 8080]
    --database-url <URL>          PostgreSQL连接URL
    --redis-url <URL>             Redis连接URL
    --external-url <URL>          外部访问URL (用于OAuth2回调)
-c, --config <FILE>               配置文件路径 [default: config.toml]
    --rtmp-port <PORT>            RTMP监听端口 [default: 1935]
    --disable-rtmp                禁用RTMP服务
    --force-migrate               强制执行数据库迁移
    --log-level <LEVEL>           日志级别 [default: info]
    --log-format <FORMAT>         日志格式 [json|pretty] [default: pretty]
-h, --help                        显示帮助信息

配置加载顺序:
1. 加载默认配置
2. 从数据库加载全局配置 (如果数据库可用)
3. 加载配置文件 (如果存在)
4. 加载环境变量
5. 应用 CLI 参数 (最高优先级)

EXAMPLES:
    # 使用配置文件启动
    synctv server --config config.toml

    # 使用CLI参数覆盖配置 (CLI参数优先级最高)
    synctv server --port 3000 --database-url postgres://...

    # 使用环境变量
    export DATABASE_URL="postgres://..."
    export REDIS_URL="redis://..."
    synctv server

    # 生产环境启动 (JSON日志)
    synctv server --log-level info --log-format json

    # 开发环境启动 (彩色日志)
    synctv server --log-level debug --log-format pretty
```

**实现**:

```rust
#[derive(Parser)]
#[command(about = "启动 SyncTV 服务器")]
pub struct ServerCommand {
    #[arg(long, default_value = "0.0.0.0", help = "监听地址")]
    pub host: String,

    #[arg(short, long, default_value_t = 8080, help = "监听端口")]
    pub port: u16,

    #[arg(long, env = "DATABASE_URL", help = "PostgreSQL连接URL")]
    pub database_url: Option<String>,

    #[arg(long, env = "REDIS_URL", help = "Redis连接URL")]
    pub redis_url: Option<String>,

    #[arg(long, help = "外部访问URL")]
    pub external_url: Option<String>,

    #[arg(short, long, default_value = "config.toml", help = "配置文件路径")]
    pub config: PathBuf,

    #[arg(long, default_value_t = 1935, help = "RTMP监听端口")]
    pub rtmp_port: u16,

    #[arg(long, help = "禁用RTMP服务")]
    pub disable_rtmp: bool,

    #[arg(long, help = "强制执行数据库迁移")]
    pub force_migrate: bool,

    #[arg(long, default_value = "info", help = "日志级别")]
    pub log_level: String,

    #[arg(long, default_value = "pretty", help = "日志格式")]
    pub log_format: String,
}

pub async fn execute(cmd: ServerCommand) -> Result<()> {
    // 配置加载顺序: CLI > Env > File > DB > Default
    let config = ConfigLoader::new()
        .with_defaults()           // 1. 默认配置
        .with_database_if_available()  // 2. 数据库配置
        .with_file(&cmd.config)    // 3. 配置文件
        .with_env()                // 4. 环境变量
        .with_cli_args(&cmd)       // 5. CLI参数 (最高优先级)
        .load()
        .await?;

    // 初始化日志
    init_logging(&config.logging)?;

    // 连接数据库
    let db_pool = connect_database(&config.database).await?;

    // 执行迁移
    if cmd.force_migrate || should_auto_migrate()? {
        run_migrations(&db_pool).await?;
    }

    // 启动服务器
    start_server(config, db_pool).await?;

    Ok(())
}
```

### 3.2 admin - 管理员管理

```bash
# 添加管理员
synctv admin add <USERNAME> [PASSWORD]

# 删除管理员
synctv admin delete <USERNAME>

# 列出所有管理员
synctv admin list [OPTIONS]
    --format <FORMAT>    输出格式 [table|json|csv] [default: table]
    --sort <FIELD>       排序字段 [username|created_at]
    --order <ORDER>      排序方向 [asc|desc] [default: asc]

EXAMPLES:
    # 添加管理员 (交互式输入密码)
    synctv admin add alice

    # 添加管理员 (非交互式)
    synctv admin add alice MySecurePassword123

    # 删除管理员 (需要确认)
    synctv admin delete alice

    # 列出管理员 (JSON格式)
    synctv admin list --format json
```

**实现**:

```rust
#[derive(Parser)]
#[command(about = "管理员管理")]
pub enum AdminCommand {
    /// 添加管理员
    Add {
        #[arg(help = "用户名")]
        username: String,

        #[arg(help = "密码 (如果不提供,将交互式输入)")]
        password: Option<String>,
    },

    /// 删除管理员
    Delete {
        #[arg(help = "用户名")]
        username: String,

        #[arg(long, help = "跳过确认提示")]
        yes: bool,
    },

    /// 列出所有管理员
    List {
        #[arg(long, default_value = "table", help = "输出格式")]
        format: OutputFormat,

        #[arg(long, help = "排序字段")]
        sort: Option<String>,

        #[arg(long, default_value = "asc", help = "排序方向")]
        order: SortOrder,
    },
}

pub async fn execute(cmd: AdminCommand) -> Result<()> {
    match cmd {
        AdminCommand::Add { username, password } => {
            add_admin(username, password).await
        }
        AdminCommand::Delete { username, yes } => {
            delete_admin(username, yes).await
        }
        AdminCommand::List { format, sort, order } => {
            list_admins(format, sort, order).await
        }
    }
}

async fn add_admin(username: String, password: Option<String>) -> Result<()> {
    // 连接数据库
    let db_pool = connect_database_from_config().await?;

    // 检查用户名是否已存在
    if user_exists(&db_pool, &username).await? {
        return Err(Error::UserExists(username));
    }

    // 获取密码
    let password = match password {
        Some(p) => p,
        None => {
            // 交互式输入密码
            dialoguer::Password::new()
                .with_prompt("密码")
                .with_confirmation("确认密码", "密码不匹配")
                .interact()?
        }
    };

    // 创建管理员
    create_admin(&db_pool, &username, &password).await?;

    println!("{} 管理员 {} 创建成功", "✓".green(), username.cyan());

    Ok(())
}

async fn delete_admin(username: String, skip_confirm: bool) -> Result<()> {
    let db_pool = connect_database_from_config().await?;

    // 检查用户是否存在
    let user = get_user_by_username(&db_pool, &username).await?;
    if !user.is_admin() {
        return Err(Error::NotAdmin(username));
    }

    // 确认删除
    if !skip_confirm {
        let confirmed = dialoguer::Confirm::new()
            .with_prompt(format!("确定要删除管理员 {} 吗?", username.cyan()))
            .default(false)
            .interact()?;

        if !confirmed {
            println!("{} 操作已取消", "ℹ".blue());
            return Ok(());
        }
    }

    // 删除管理员
    delete_user(&db_pool, user.id).await?;

    println!("{} 管理员 {} 已删除", "✓".green(), username.cyan());

    Ok(())
}
```

### 3.3 user - 用户管理

```bash
# 列出用户
synctv user list [OPTIONS]
    --role <ROLE>        按角色过滤 [admin|user|pending|banned]
    --status <STATUS>    按状态过滤
    --page <PAGE>        页码 [default: 1]
    --per-page <N>       每页数量 [default: 20]
    --format <FORMAT>    输出格式 [table|json|csv]

# 搜索用户
synctv user search <KEYWORD> [OPTIONS]
    --by <FIELD>         搜索字段 [username|email|id] [default: username]
    --format <FORMAT>    输出格式

# 显示用户详情
synctv user show <USERNAME_OR_ID>

# 封禁用户
synctv user ban <USERNAME_OR_ID> [OPTIONS]
    --reason <TEXT>      封禁原因
    --yes                跳过确认

# 解封用户
synctv user unban <USERNAME_OR_ID> [OPTIONS]
    --yes                跳过确认

# 删除用户
synctv user delete <USERNAME_OR_ID> [OPTIONS]
    --yes                跳过确认

# 批准待审核用户
synctv user approve <USERNAME_OR_ID>

EXAMPLES:
    # 列出所有用户
    synctv user list

    # 列出待审核用户
    synctv user list --role pending

    # 搜索用户
    synctv user search alice

    # 显示用户详情
    synctv user show alice

    # 封禁用户
    synctv user ban alice --reason "垃圾信息"

    # 批准用户
    synctv user approve alice
```

### 3.4 setting - 设置管理

```bash
# 列出所有设置
synctv setting list [OPTIONS]
    --category <CAT>     按分类过滤
    --format <FORMAT>    输出格式 [table|json]

# 显示设置详情
synctv setting show <KEY>

# 修改设置
synctv setting set <KEY> <VALUE>

# 重置为默认值
synctv setting reset <KEY> [OPTIONS]
    --yes                跳过确认

EXAMPLES:
    # 列出所有设置
    synctv setting list

    # 列出房间相关设置
    synctv setting list --category room

    # 显示设置详情
    synctv setting show room.disable_create

    # 修改设置
    synctv setting set room.disable_create true

    # 禁用用户注册
    synctv setting set user.disable_signup true

    # 设置房间TTL为24小时
    synctv setting set room.ttl_hours 24

    # 重置设置
    synctv setting reset room.disable_create
```

**实现**:

```rust
#[derive(Parser)]
#[command(about = "设置管理")]
pub enum SettingCommand {
    /// 列出所有设置
    List {
        #[arg(long, help = "按分类过滤")]
        category: Option<String>,

        #[arg(long, default_value = "table")]
        format: OutputFormat,
    },

    /// 显示设置详情
    Show {
        #[arg(help = "设置键名")]
        key: String,
    },

    /// 修改设置
    Set {
        #[arg(help = "设置键名")]
        key: String,

        #[arg(help = "设置值")]
        value: String,
    },

    /// 重置为默认值
    Reset {
        #[arg(help = "设置键名")]
        key: String,

        #[arg(long, help = "跳过确认")]
        yes: bool,
    },
}

async fn list_settings(
    category: Option<String>,
    format: OutputFormat,
) -> Result<()> {
    let db_pool = connect_database_from_config().await?;
    let settings = get_settings(&db_pool, category).await?;

    match format {
        OutputFormat::Table => {
            print_settings_table(&settings)?;
        }
        OutputFormat::Json => {
            println!("{}", serde_json::to_string_pretty(&settings)?);
        }
        _ => {}
    }

    Ok(())
}

fn print_settings_table(settings: &[Setting]) -> Result<()> {
    use comfy_table::{Table, presets::UTF8_FULL};

    let mut table = Table::new();
    table
        .load_preset(UTF8_FULL)
        .set_header(vec!["键名", "值", "类型", "分类", "描述"]);

    for setting in settings {
        table.add_row(vec![
            setting.key.clone(),
            setting.value.clone(),
            setting.value_type.clone(),
            setting.category.clone(),
            setting.description.clone().unwrap_or_default(),
        ]);
    }

    println!("{table}");

    Ok(())
}
```

### 3.5 migrate - 数据库迁移

```bash
# 执行迁移
synctv migrate up [OPTIONS]
    --to <VERSION>       迁移到指定版本
    --steps <N>          执行N步迁移

# 回滚迁移
synctv migrate down [OPTIONS]
    --to <VERSION>       回滚到指定版本
    --steps <N>          回滚N步
    --yes                跳过确认

# 查看迁移状态
synctv migrate status

EXAMPLES:
    # 执行所有待执行的迁移
    synctv migrate up

    # 执行1步迁移
    synctv migrate up --steps 1

    # 回滚1步迁移
    synctv migrate down --steps 1

    # 查看迁移状态
    synctv migrate status
```

### 3.6 backup - 备份与恢复

```bash
# 创建备份
synctv backup create [OPTIONS]
    --output <FILE>      输出文件路径
    --compress           压缩备份文件
    --tables <TABLES>    指定备份的表 (逗号分隔)

# 恢复备份
synctv backup restore <FILE> [OPTIONS]
    --yes                跳过确认

# 列出备份
synctv backup list [OPTIONS]
    --path <DIR>         备份文件目录

EXAMPLES:
    # 创建备份
    synctv backup create --output backup-2024-01-30.sql.gz --compress

    # 恢复备份
    synctv backup restore backup-2024-01-30.sql.gz

    # 列出备份
    synctv backup list --path ./backups
```

### 3.7 completion - Shell补全

```bash
# 生成Shell补全脚本
synctv completion <SHELL>

SHELLS:
    bash
    zsh
    fish
    powershell

EXAMPLES:
    # Bash
    synctv completion bash > /etc/bash_completion.d/synctv

    # Zsh
    synctv completion zsh > ~/.zfunc/_synctv

    # Fish
    synctv completion fish > ~/.config/fish/completions/synctv.fish
```

**实现**:

```rust
use clap_complete::{generate, Shell};

#[derive(Parser)]
#[command(about = "生成Shell补全脚本")]
pub struct CompletionCommand {
    #[arg(value_enum, help = "Shell类型")]
    shell: Shell,
}

pub fn execute(cmd: CompletionCommand) -> Result<()> {
    let mut app = Cli::command();
    let app_name = app.get_name().to_string();

    generate(
        cmd.shell,
        &mut app,
        app_name,
        &mut std::io::stdout(),
    );

    Ok(())
}
```

---

## 4. 交互式体验

### 4.1 彩色输出

```rust
use console::{style, Emoji};

// 成功消息
println!(
    "{} {} {}",
    style("✓").green().bold(),
    style("成功:").green(),
    message
);

// 错误消息
eprintln!(
    "{} {} {}",
    style("✗").red().bold(),
    style("错误:").red(),
    error_message
);

// 警告消息
println!(
    "{} {} {}",
    style("⚠").yellow().bold(),
    style("警告:").yellow(),
    warning_message
);

// 信息消息
println!(
    "{} {} {}",
    style("ℹ").blue().bold(),
    style("信息:").blue(),
    info_message
);
```

### 4.2 进度条

```rust
use indicatif::{ProgressBar, ProgressStyle};

pub async fn run_migrations(db: &PgPool) -> Result<()> {
    let migrations = get_pending_migrations(db).await?;

    let pb = ProgressBar::new(migrations.len() as u64);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("{spinner:.green} [{bar:40.cyan/blue}] {pos}/{len} {msg}")?
            .progress_chars("#>-")
    );

    for migration in migrations {
        pb.set_message(format!("执行 {}", migration.name));
        migration.apply(db).await?;
        pb.inc(1);
    }

    pb.finish_with_message("迁移完成");

    Ok(())
}
```

### 4.3 交互式确认

```rust
use dialoguer::{Confirm, Select, Input, Password};

// 确认对话框
let confirmed = Confirm::new()
    .with_prompt("确定要删除吗?")
    .default(false)
    .interact()?;

// 选择对话框
let options = vec!["选项1", "选项2", "选项3"];
let selection = Select::new()
    .with_prompt("请选择")
    .items(&options)
    .default(0)
    .interact()?;

// 输入框
let username: String = Input::new()
    .with_prompt("用户名")
    .validate_with(|input: &String| {
        if input.len() < 3 {
            Err("用户名至少3个字符")
        } else {
            Ok(())
        }
    })
    .interact_text()?;

// 密码输入框
let password = Password::new()
    .with_prompt("密码")
    .with_confirmation("确认密码", "密码不匹配")
    .interact()?;
```

### 4.4 表格输出

```rust
use comfy_table::{Table, presets::UTF8_FULL, Color, Attribute};

pub fn print_users_table(users: &[User]) {
    let mut table = Table::new();
    table.load_preset(UTF8_FULL);

    // 设置表头
    table.set_header(vec![
        "ID",
        "用户名",
        "角色",
        "状态",
        "创建时间",
    ]);

    // 添加行
    for user in users {
        let role_color = match user.role {
            UserRole::Root => Color::Red,
            UserRole::Admin => Color::Yellow,
            _ => Color::White,
        };

        let status_display = match user.status {
            UserStatus::Active => style("活跃").green(),
            UserStatus::Pending => style("待审核").yellow(),
            UserStatus::Banned => style("封禁").red(),
        };

        table.add_row(vec![
            user.id.to_string(),
            user.username.clone(),
            style(&user.role.to_string()).fg(role_color).to_string(),
            status_display.to_string(),
            user.created_at.format("%Y-%m-%d %H:%M").to_string(),
        ]);
    }

    println!("{table}");
}
```

---

## 5. 错误处理

### 5.1 错误类型

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum CliError {
    #[error("配置错误: {0}")]
    Config(String),

    #[error("数据库错误: {0}")]
    Database(#[from] sqlx::Error),

    #[error("用户 {0} 已存在")]
    UserExists(String),

    #[error("用户 {0} 不存在")]
    UserNotFound(String),

    #[error("{0} 不是管理员")]
    NotAdmin(String),

    #[error("权限不足")]
    PermissionDenied,

    #[error("操作已取消")]
    Cancelled,

    #[error("IO错误: {0}")]
    Io(#[from] std::io::Error),
}
```

### 5.2 错误显示

```rust
pub fn display_error(error: &CliError) {
    use console::style;

    eprintln!(
        "{} {} {}",
        style("✗").red().bold(),
        style("错误:").red().bold(),
        style(error.to_string()).red()
    );

    // 显示详细错误信息 (debug模式)
    if std::env::var("RUST_LOG").unwrap_or_default().contains("debug") {
        eprintln!("\n{} 详细信息:", style("ℹ").blue());
        eprintln!("{:?}", error);
    }

    // 显示解决建议
    match error {
        CliError::UserExists(username) => {
            eprintln!(
                "\n{} 尝试使用不同的用户名,或使用 {} 查看现有用户",
                style("💡").yellow(),
                style("synctv user list").cyan()
            );
        }
        CliError::PermissionDenied => {
            eprintln!(
                "\n{} 请确保您有足够的权限执行此操作",
                style("💡").yellow()
            );
        }
        _ => {}
    }
}
```

---

## 6. 实现方案

### 6.1 主入口

```rust
use clap::Parser;

#[derive(Parser)]
#[command(name = "synctv")]
#[command(version, about = "SyncTV - 多人实时视频同步观看平台", long_about = None)]
pub struct Cli {
    /// 数据目录
    #[arg(short, long, global = true, default_value = "./data")]
    pub data_dir: PathBuf,

    /// 配置文件路径
    #[arg(short, long, global = true, default_value = "config.toml")]
    pub config: PathBuf,

    /// 跳过配置文件加载
    #[arg(long, global = true)]
    pub skip_config: bool,

    /// 跳过环境变量加载
    #[arg(long, global = true)]
    pub skip_env: bool,

    /// 日志级别
    #[arg(long, global = true, default_value = "info")]
    pub log_level: String,

    /// 日志格式
    #[arg(long, global = true, default_value = "pretty")]
    pub log_format: String,

    #[command(subcommand)]
    pub command: Commands,
}

#[derive(Subcommand)]
pub enum Commands {
    /// 启动服务器
    Server(ServerCommand),

    /// 管理员管理
    Admin {
        #[command(subcommand)]
        command: AdminCommand,
    },

    /// Root用户管理
    Root {
        #[command(subcommand)]
        command: RootCommand,
    },

    /// 用户管理
    User {
        #[command(subcommand)]
        command: UserCommand,
    },

    /// 房间管理
    Room {
        #[command(subcommand)]
        command: RoomCommand,
    },

    /// 设置管理
    Setting {
        #[command(subcommand)]
        command: SettingCommand,
    },

    /// 数据库迁移
    Migrate {
        #[command(subcommand)]
        command: MigrateCommand,
    },

    /// 备份与恢复
    Backup {
        #[command(subcommand)]
        command: BackupCommand,
    },

    /// 生成Shell补全脚本
    Completion(CompletionCommand),
}

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();

    // 初始化日志
    init_logging(&cli.log_level, &cli.log_format)?;

    // 执行命令
    let result = match cli.command {
        Commands::Server(cmd) => server::execute(cmd).await,
        Commands::Admin { command } => admin::execute(command).await,
        Commands::Root { command } => root::execute(command).await,
        Commands::User { command } => user::execute(command).await,
        Commands::Room { command } => room::execute(command).await,
        Commands::Setting { command } => setting::execute(command).await,
        Commands::Migrate { command } => migrate::execute(command).await,
        Commands::Backup { command } => backup::execute(command).await,
        Commands::Completion(cmd) => completion::execute(cmd),
    };

    // 处理错误
    if let Err(e) = result {
        display_error(&e);
        std::process::exit(1);
    }

    Ok(())
}
```

---

## 总结

本章节设计了一个**功能完整、用户友好**的 CLI 系统:

✅ **清晰的命令树结构**: 易于记忆和使用
✅ **丰富的交互体验**: 彩色输出、进度条、交互式确认
✅ **脚本友好**: 支持非交互式使用,多种输出格式
✅ **安全设计**: 危险操作需要确认
✅ **Shell补全**: 提升使用效率
✅ **错误处理**: 清晰的错误提示和解决建议

关键改进:

- ❌ 移除了 `--skip-config` 选项 (配置加载自动处理)
- ❌ 移除了 `--skip-env` 选项 (环境变量自动处理)
- ❌ 移除了 `--disable-web` 选项 (只有一个统一的应用)
- ❌ 移除了 `--web-path` 选项 (不再需要单独的Web路径)
- ✅ 明确了配置优先级: CLI Args > Env > File > DB > Default
- ✅ 使用 `ConfigLoader` 统一管理配置加载流程

**下一章**: [12-时间同步与补偿](./12-时间同步与补偿.md)

---

**上一章**: [19-配置管理系统](./19-配置管理系统.md)
**下一章**: [21-关键实现](./21-关键实现.md)
