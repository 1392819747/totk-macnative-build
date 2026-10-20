# totk-macnative-build

《塞尔达传说 王国之泪》Mac 原生版的 **Codemagic 云构建配置**。

> 上游项目：https://github.com/ly05010419/MacNativeGames
> **本仓库不含任何游戏数据、密钥、着色器**，只有流水线配置。构建使用你自己的游戏文件，产物仅供自用。

## 云构建方案是什么

在 Codemagic 的 **Mac mini M4（10 核 CPU / 16GB 内存）** 构建机上自动完成全部编译：

| 项 | 配置 |
|---|---|
| 构建机 | Mac mini M4，10 核 CPU / 16GB 内存（与官方推荐配置一致） |
| 系统镜像 | macOS 26.5 + 完整 Xcode 26.5（Metal 工具链，启动着色器可编入 App） |
| 磁盘 | 294GB 总量 / 约 144GB 可用（构建约需 45–60GB） |
| 时长上限 | 120 分钟 |
| 费用 | M4 $0.114/分钟；一次完整构建约 40–90 分钟（≈ $5–10）。免费档每月 500 分钟仅限 M2 机型 |

**当前进度**：Codemagic 应用 `totk-macnative-build`（ID `6ad724763e412ecb7e907cb5`）和变量组 `totk_secrets` 已创建；GitHub 仓库待创建（见下方「唯一待办」）。

**仓库可见性**：当前 Codemagic 应用以「手动添加（generic）」方式接入。
- 仓库设为**公开** → 直接可用，零额外配置（本仓库只有流水线配置，无任何敏感内容）
- 仓库设为**私有** → 需在 Codemagic UI 用 GitHub 集成重新添加该仓库（Add application → GitHub → 选择 `totk-macnative-build`），或为仓库配置 Codemagic 的 deploy key

## 唯一待办（30 秒）

在 GitHub 网页创建仓库：https://github.com/new
- Repository name: `totk-macnative-build`
- **Public**（推荐，零配置）或 Private（见上文可见性说明）
- **不要**勾选 "Add a README"（保持空仓库，配置文件由流水线推送）

创建后告知，随后自动完成：推送 `codemagic.yaml` + README → 触发一次验证构建。

## 一次性准备

### 第 1 步：从你的 Switch 提取文件（全部来自你自己的设备）

| 文件 | 怎么来 | 是否必需 |
|---|---|---|
| `prod.keys` | Switch 上运行 **Lockpick_RCM**（或 Lockpick），导出到 SD 卡 `/switch/prod.keys` | ✅ 必需 |
| 本体 1.0.0（`0100F2C0115B6000`） | **nxdumptool** 从卡带转储 **XCI**（推荐，含系统更新分区可提取字体），或转储已安装本体的 **NSP**（需含通用票据） | ✅ 必需 |
| 共享字体目录（六个 .bfttf） | 仅 NSP 路线需要。**已代劳提取**：用你的 prod.keys 从固件 22.5.0 提取并通过全部校验（NCA 头 + 六级 IVFC + BFTTF 结构），成品 `shared-fonts-22.5.0.zip`（约 12MB，含 `0-standard.bfttf`～`5-nintendo-extension.bfttf` + manifest.json）。其中 4 个字体与官方参考转储哈希不同（22.5.0 新版），构建时走「按实际字节绑定」路径，属正常支持行为。备选：上传 325MB 固件目录（`mv 解压目录 firmware && tar czf firmware.tar.gz firmware/`）由构建机现提取 | NSP 必需 |
| 更新包 1.1.2 / 1.2.1 / 1.4.3 | nxdumptool 转储已下载的更新标题，NSP 或 NSZ | 可选 |

> ⚠️ **`prod.keys` 是你 Switch 的根密钥**，泄露等于把机器和账号交给任天堂封禁。不要发给任何人（包括 AI 助手）、不要提交进任何仓库、不要写进任何日志。它只应存在于：你的私密存储 + Codemagic 的 Secret 环境变量 + 构建机内存/磁盘（构建结束销毁）。

### 第 2 步：把文件放到可下载的直链

推荐腾讯云 COS（你有使用经验）：

1. 建一个**私有** bucket，上传：`totk.nsp`（你的 NSP 本体）、`prod.keys`、`1.4.3.nsp`（更新包）、`shared-fonts-22.5.0.zip`（共享字体，已备好）
2. 为每个文件生成**带过期时间的预签名下载 URL**（建议有效期 24 小时，构建完成后失效）：
   - COS 控制台 → 文件详情 → 复制预签名链接；或用 `coscmd` / COS SDK 生成
3. 记下 bucket 名、区域、API 密钥（SecretId/SecretKey）

### 第 3 步：在 Codemagic 配置环境变量

> 变量组 `totk_secrets` 已通过 API 创建完毕（应用 `totk-macnative-build`，ID `6ad724763e412ecb7e907cb5`）。
> 你只需要往里填值——UI 添加，或把值发给我用 API 批量导入：
> `POST https://codemagic.io/api/v3/variable-groups/6ad7255b7550aed810906c55/variables`
> body：`{"secure": true, "variables": [{"name":"GAME_URL","value":"..."}, ...]}`

进入应用 → **Environment variables** → 变量组 `totk_secrets` → 添加以下变量（敏感项勾选 **Secret**）：

| 变量 | 值 | 必需 |
|---|---|---|
| `GAME_URL` | 本体（XCI/NSP）预签名直链 | ✅ |
| `KEYS_URL` | `prod.keys` 预签名直链 | ✅ |
| `GAME_EXT` | `xci` 或 `nsp`（与本体格式一致） | ✅ |
| `UPDATE_URL` | 更新包直链 | 可选 |
| `UPDATE_EXT` | `nsp` 或 `nsz` | 可选 |
| `SHARED_FONTS_URL` | `shared-fonts-22.5.0.zip` 直链（**首选**，约 12MB） | 仅 NSP 路线 |
| `FIRMWARE_URL` | `firmware.tar.gz` 直链（备选，325MB，不配 SHARED_FONTS_URL 时才用） | 仅 NSP 路线 |
| `COS_SECRET_ID` | 腾讯云 API SecretId | 可选* |
| `COS_SECRET_KEY` | 腾讯云 API SecretKey | 可选* |
| `COS_BUCKET` | bucket 名 | 可选* |
| `COS_REGION` | 如 `ap-guangzhou` | 可选* |

\* 不配 COS 四个变量也能构建，成品改为从 Codemagic 构建页的 **Artifacts** 下载（约 16GB，走 Codemagic 下载通道）。

`UPDATE_EXT` 默认 `nsp`（符合你的更新包）；`GAME_EXT=nsp` 已通过 API 预填。NSP 路线只需再配 `SHARED_FONTS_URL`。

### 第 4 步：启动构建

- **UI**：应用页 → `Start new build` → 分支 `main` → 工作流 `TOTK macOS Build (MacNativeGames)`
- **API**（仓库就绪后）：
  ```sh
  curl -X POST "https://codemagic.io/api/v3/apps/6ad724763e412ecb7e907cb5/builds" \
    -H "x-auth-token: <你的 Codemagic API token>" -H "Content-Type: application/json" \
    -d '{"workflow_id":"totk-mac-build","branch":"main","instance_type":"mac_mini_m4"}'
  ```

未配置直链时构建会自动跳过（退出码 0），可用来先验证流水线本身是否通畅。

### 第 5 步：取回成品

- **COS 路线**：从 bucket 下载 `totk/Tears of the Kingdom.tar`（约 16GB），解压得到 `Tears of the Kingdom.app`
- **制品路线**：构建页 → Artifacts → 下载 `Tears of the Kingdom.tar`
- 下载后如遇「无法打开，来自身份不明的开发者」：右键 → 打开；或 `xattr -dr com.apple.quarantine "Tears of the Kingdom.app"`
- 首次启动校验游戏数据约 20 秒；标题画面约 2 秒；首次进入世界最多再等 15 秒（1.4.3 已预热 15,266 个着色器）

## 流水线做了什么（codemagic.yaml）

1. **预检**：缺少 `GAME_URL`/`KEYS_URL` 时安全退出（验证模式）
2. **装依赖**：`brew install opus libvpx python@3.14`；尝试下载 Metal 工具链（失败则退回本机 Metal 缓存方案）
3. **下载**：游戏文件 + `prod.keys`（+ 更新包/固件）到构建机临时目录
4. **构建**：`git clone` 上游仓库 → `python3.14 build_app.py` → 输出到 `/tmp/totk-out`
5. **打包上传**：`tar` 打包 → 有 COS 变量则上传 COS，否则只发布 Codemagic 制品 → 邮件通知

## 排障

| 现象 | 处理 |
|---|---|
| 构建秒过、日志显示「本次构建跳过」 | 环境变量没配上；检查组名必须是 `totk_secrets`，`GAME_URL`/`KEYS_URL` 已设且勾了 Secret |
| 下载游戏文件失败（curl 22） | 预签名 URL 过期（重新生成）或链接不可公开访问 |
| 哈希校验失败、构建停止 | 本体不是 1.0.0（`0100F2C0115B6000`）或文件损坏；重新转储 |
| NSP 报缺字体 | 提供 `FIRMWARE_URL`，tar 内顶层目录必须是 `firmware/` |
| 提示「别的程序正在用 Metal 缓存」 | 不会发生在 CI（全新构建机）；本地构建才会遇到 |
| 制品下载太慢 | 改用 COS 路线；或下次构建加 COS 变量 |
| 想换版本 | 改 `UPDATE_URL` 指向 1.1.2/1.2.1/1.4.3，重新构建（窗口标题显示版本） |
| 构建超过 120 分钟被截断 | 下载 16GB 太慢；换更快的存储/就近 region，或分次构建 |

## 安全与合规

- 你必须拥有正版游戏；产物含游戏本体，**仅供自用，不要分享**
- 仓库、流水线、日志中不出现任何密钥值；`prod.keys` 只经 Secret 环境变量与预签名 URL 传递
- Codemagic 构建机在构建结束后销毁，临时文件随之清除
- 用完建议在 Codemagic 后台 **Revoke 并重新生成 API token**（曾在聊天中明文出现）
