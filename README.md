# Huadian Drive — 华电云盘 Windows 挂载客户端

基于 Go + WinFsp 的原生 Windows 云盘挂载工具。将华电云盘 (`pan.ncepu.edu.cn`) 挂载为 Windows 盘符，支持资源管理器和命令行操作。当前无 GUI。

## 架构

```
┌─────────────────┐     Named Pipe IPC      ┌──────────────────┐
│    hddfs.exe     │ ◄─────────────────────► │  hddsyncd.exe     │
│  (WinFsp 挂载)    │   fs.list/stat/rename   │  (后台 daemon)     │
│                  │   fs.remove/uploadStaged │                  │
└────────┬─────────┘                         └────────┬─────────┘
         │                                            │
         │  Windows 资源管理器                         │  HTTP API
         │  PowerShell / cmd                           │
         ▼                                            ▼
   盘符 (S:)                               https://pan.ncepu.edu.cn
                                          /api/efast/v1/*
```

| 组件 | 文件 | 角色 |
|---|---|---|
| `hddsyncd.exe` | `cmd/hddsyncd/` | 后台守护进程。通过 Named Pipe 接收 WinFsp 请求，调用 AnyShare API 操作远端文件 |
| `hddfs.exe` | `cmd/hddfs/` + `internal/mount/winfsp/` | WinFsp 挂载程序。将 FUSE 回调转为 IPC 请求发给 daemon |
| `hddctl.exe` | `cmd/hddctl/` | 命令行维护工具。直接调 Provider 接口操作云盘，不经过 daemon |

## 前置条件

| 组件 | 要求 | 说明 |
|---|---|---|
| 操作系统 | Windows 10/11 x64 | |
| Go | ≥ 1.21 | 仅构建时需要 |
| WinFsp | 
| MinGW-w64 | GCC 工具链 | CGo 编译 WinFsp 需要。仓库 `.tools/mingw64/` 中已附带 |
| 华电账号 | netID + 密码 | 仅真实云盘模式需要 |

## 快速构建

```powershell
# 1. 设置 MinGW 工具链（使用仓库附带的版本）
$env:MINGW = Resolve-Path .tools\mingw64\mingw64
$env:PATH = "$env:MINGW\bin;$env:PATH"
$env:CGO_ENABLED = "1"

# 2. 编译三个 EXE
go build -p=1 -o bin\hddsyncd.exe ./cmd/hddsyncd
go build -p=1 -o bin\hddfs.exe   ./cmd/hddfs
go build -p=1 -o bin\hddctl.exe  ./cmd/hddctl

# 3. 验证
.\bin\hddsyncd.exe version   # → hddsyncd version 0.1.0-dev
.\bin\hddfs.exe version      # → hddfs version 0.1.0
.\bin\hddctl.exe version     # → hddctl version 0.1.0-dev
```

**首次构建较慢**（~3 分钟），因为 CGo 会编译 SQLite 和 WinFsp 绑定。后续增量构建几秒内完成。

## 快速启动

### Mock 模式（无需网络、无需云盘账号）

```powershell
# 终端 1：启动 Mock daemon
mkdir .tmp\mock-root -Force; Set-Content .tmp\mock-root\hello.txt "hello"
.\bin\hddsyncd.exe run --provider mock --root .tmp\mock-root --data-dir .tmp\data --no-background

# 终端 2：挂载为 R: 盘
.\bin\hddfs.exe mount --daemon --mount R: --mkdir-rename-move-file-rename-move-remove-copy-upload-only

# 终端 3：操作挂载盘
Get-ChildItem R:\
Get-Content R:\hello.txt
```

### 真实云盘模式（需要华电账号）

```powershell
# 1. 首次登录
.\bin\hddctl.exe login
.\bin\hddctl.exe remote ls /

# 2. 终端 1：启动 daemon
.\bin\hddsyncd.exe run --mkdir-rename-move-file-rename-move-remove-copy-upload-only --no-background

# 3. 终端 2：挂载
.\bin\hddfs.exe mount --daemon --mount S: --mkdir-rename-move-file-rename-move-remove-copy-upload-only
```

## 功能测试流程

### Mock 本地测试（14 项）

```powershell
# 准备（一次性）
$mockRoot = ".tmp\mock-test-root"
mkdir $mockRoot\source, $mockRoot\target, $mockRoot\upload -Force
Set-Content "$mockRoot\source\file.txt" "test-content"
Set-Content "$mockRoot\hello.txt" "hello"

# 启动（daemon + 挂载，两个终端）
.\bin\hddsyncd.exe run --provider mock --root $mockRoot --data-dir .tmp\mock-data --pipe \\.\pipe\mock --mkdir-rename-move-file-rename-move-remove-copy-upload-only --no-background 2>.tmp\mock-daemon.log
.\bin\hddfs.exe mount --daemon --pipe \\.\pipe\mock --mount R: --mkdir-rename-move-file-rename-move-remove-copy-upload-only --debug-log .tmp\mock-debug.log
```

```
□ 1. 浏览目录          Get-ChildItem R:\
□ 2. 读取文件          Get-Content R:\hello.txt → "hello"
□ 3. 新建目录          New-Item -ItemType Directory R:\newdir → $mockRoot\newdir 出现
□ 4. 文件同父重命名     Rename-Item R:\source\file.txt -NewName renamed.txt → 旧路径消失
□ 5. 文件跨父移动       Move-Item R:\source\renamed.txt R:\target\ → 目标出现
□ 6. 目录跨父移动       Move-Item R:\newdir R:\target\ → 目标出现
□ 7. 删除文件           Remove-Item R:\target\file.txt → 消失
□ 8. 删除空目录         Remove-Item R:\target\newdir → 消失
□ 9. 删除非空目录       Remove-Item R:\target -Recurse → 子文件和目录均消失
□ 10. 上传新文件        Copy-Item "$mockRoot\hello.txt" R:\upload\copied.txt → Mock 出现
□ 11. 同名冲突拒绝      Copy-Item "$mockRoot\hello.txt" R:\upload\copied.txt → 报错，不覆盖
□ 12. 文件创建拒绝      New-Item -ItemType File R:\nope.txt → 失败
□ 13. 写入拒绝          Set-Content R:\hello.txt "hack" → $mockRoot\hello.txt 未变
□ 14. 旧模式拒绝删除    （用 --mkdir-rename-move-file-rename-move-only 启动）Remove-Item → 失败
```

### 真实云盘测试（需登录 + 隔离目录）

```powershell
# 登录
.\bin\hddctl.exe login

# 创建隔离测试目录
$ts = Get-Date -Format "yyyyMMdd-HHmmss"
.\bin\hddctl.exe remote mkdir "/test/hdd-test-$ts"
.\bin\hddctl.exe remote mkdir "/test/hdd-test-$ts/sub"

# 上传文件
Set-Content .tmp\upload.txt "real-test"
.\bin\hddctl.exe remote upload .tmp\upload.txt "/test/hdd-test-$ts/sub/file.txt"

# 下载并比对 SHA256
.\bin\hddctl.exe remote download "/test/hdd-test-$ts/sub/file.txt" .tmp\downloaded.txt
Get-FileHash .tmp\upload.txt -Algorithm SHA256
Get-FileHash .tmp\downloaded.txt -Algorithm SHA256  # 必须一致

# 启动 daemon + 挂载（有 GUI 验证）
.\bin\hddsyncd.exe run --mkdir-rename-move-file-rename-move-remove-copy-upload-only --no-background 2>.tmp\daemon.log
.\bin\hddfs.exe mount --daemon --mount S: --mkdir-rename-move-file-rename-move-remove-copy-upload-only --debug-log .tmp\debug.log
# 打开资源管理器 S: → 在 /test/hdd-test-*/ 下执行功能测试

# 事后清理（停 daemon 后用 hddctl）
.\bin\hddctl.exe remote rm "/test/hdd-test-$ts/sub/file.txt"
.\bin\hddctl.exe remote rmdir "/test/hdd-test-$ts/sub"
.\bin\hddctl.exe remote rmdir "/test/hdd-test-$ts"
```

## 写策略模式

| Flag | mkdir | rename | move | file rename | file move | delete | upload |
|---|---|---|---|---|---|---|---|
| `--read-only` | | | | | | | |
| `--mkdir-only` | ✓ | | | | | | |
| `--mkdir-rename-only` | ✓ | ✓(dir) | | | | | |
| `--mkdir-rename-move-only` | ✓ | ✓(dir) | ✓(dir) | | | | |
| `--mkdir-rename-move-file-rename-only` | ✓ | ✓(dir) | ✓(dir) | ✓(same) | | | |
| `--mkdir-rename-move-file-rename-move-only` | ✓ | ✓ | ✓ | ✓(same) | ✓(同) | | |
| `--mkdir-rename-move-file-rename-move-remove-only` | ✓ | ✓ | ✓ | ✓(same) | ✓(同) | ✓ | |
| `--mkdir-rename-move-file-rename-move-remove-copy-upload-only` | ✓ | ✓ | ✓ | ✓(same) | ✓(同) | ✓ | ✓ |
| 默认（无 flag） | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

模式互斥，不支持组合。`hddsyncd` 和 `hddfs` 两端需使用**相同的**模式 flag。

## 项目结构

```
D:\ncepupan/
├── cmd/                      三个可执行程序入口
│   ├── hddctl/               hddctl.exe（login/remote/sync/rule/daemon-probe）
│   ├── hddsyncd/             hddsyncd.exe（daemon 全生命周期 + FUSE handler + IPC 服务）
│   └── hddfs/                hddfs.exe（WinFsp 挂载 + 参数解析）
├── internal/                  Go 库代码
│   ├── cloud/                云存储抽象 (Provider 接口 + huadian/mock 实现 + auth)
│   ├── mount/winfsp/         WinFsp FUSE 回调 (ipcfs/cloudfs/memfs)
│   ├── platform/windows/     Named Pipe / Windows 服务 / WebView2 登录
│   ├── store/sqlite/         SQLite 持久存储 (任务队列/同步根/文件元数据)
│   ├── worker/               异步任务消费池 (upload/download/remove + 重试退避)
│   ├── watch/                轮询目录监控 (防抖 + 删除检测)
│   ├── ipc/                  IPC 协议格式定义
│   ├── filter/               文件过滤规则
│   ├── domain/               共享基础类型
│   ├── logging/              安全日志 (SHA256 脱敏)
│   ├── sync/                 同步引擎 (预留)
│   └── config/               配置加载 (预留)
├── release/                  用户发行目录 (文档+脚本，不含 EXE)
├── docs/                     设计文档 + API 参考
│   ├── huadian-api.md        华电云盘完整 API 参考
│   ├── buildlog1.md          全阶段 AI 辅助开发报告
│   └── tree-explanation.md   文件树逐文件说明
├── scripts/                  开发辅助脚本 (build.ps1 等)
├── .tools/                   开发工具 (MinGW, Chromium)
├── go.mod                    Go 模块声明
└── AGENTS.md                 AI 辅助开发规范
```

## 开发验证

```powershell
# 定向测试（禁止 go test ./...）
go test -p=1 -count=1 -timeout=120s ./cmd/hddsyncd/...
go test -p=1 -count=1 -timeout=120s ./internal/mount/winfsp/...
go test -p=1 -count=1 -timeout=120s ./internal/ipc/...
go test -p=1 -count=1 -timeout=120s ./internal/cloud/mock/...

# 静态检查
go vet ./cmd/hddsyncd/ ./cmd/hddfs/ ./internal/mount/winfsp/ ./internal/cloud/mock/

# 格式
gofmt -l cmd/ internal/
git diff --check
```

## 文档索引

| 文档 | 内容 |
|---|---|
| `docs/huadian-api.md` | 华电云盘 16 个 API 端点的完整参考（请求/响应格式、ondup 冲突策略、安全设计） |
| `docs/buildlog1.md` | Codex + OpenCode 全阶段开发报告（从 mkdir-only 到 copy-upload） |
| `docs/tree-explanation.md` | 全仓库每个目录和文件的逐行职责说明 |
| `AGENTS.md` | AI 辅助开发的架构约束和代码规范 |
| `release/README.md` | 用户发行版说明（构建、测试、FAQ、已知限制） |

## 已知限制

- **覆盖上传不支持**：Explorer Replace 和 Copy-Item -Force 不可用（cgofuse 无法暴露 WinFsp create disposition）
- **在线编辑不支持**：已有文件不能直接编辑保存，只支持上传新文件
- **非实时双向**：本地 watcher 是 2s 轮询，不监听远端变更
- **无断点续传**：大文件上传失败从头重试
- **无 GUI**：资源管理器提供基础图形化体验

## License

课程项目 — 无公共许可。
