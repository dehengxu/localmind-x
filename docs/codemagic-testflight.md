# LocalMind X → Codemagic CI + TestFlight 部署清单

> 这是一份**照着点就能做完**的控制台清单。CI 侧的配置文件是仓库根的 `codemagic.yaml`
> （workflow：`ios-testflight`），本文只讲控制台要填什么、字段名必须怎么写、值从哪来。
>
> 本仓库 fork 自 `abdulmominsakib/localmind`，原 Bundle ID `pro.momin.localmind`、
> 原 Team ID `TZX5LG7ZW5` 属于原作者，已全部换掉：Bundle ID 现在是 `top.headwindx.localmind`，
> `ios/Runner.xcodeproj/project.pbxproj` 里的 `DEVELOPMENT_TEAM` 现在是**空占位**（`DEVELOPMENT_TEAM = "";`），
> 由 CI 用环境变量 `XCODE_TEAM_ID` 覆盖（见 §3.4）。

## 0. 仓库统一管理在 code.headwindx.top（正本在哪，先看这段）

| 项 | 值 |
| --- | --- |
| 正本仓库 | 自建 Gitea `code.headwindx.top` → **`one/localmind-x`**（私有，默认分支 `main`） |
| SSH 远端 | `git@git.headwindx.top:one/localmind-x.git` |
| HTTPS 远端 | `https://code.headwindx.top/one/localmind-x.git` |
| Gitea 版本 | 1.26.1（2026-11-22 实测 `https://code.headwindx.top/api/v1/version`） |
| GitHub fork | `dehengxu/localmind-x` → **降级为只读备份/来源，不再是 CI 触发源** |
| 仓库 id | `109`（2026-11-22 15:01:47Z 创建） |
| 内容状态 | GitHub 侧已全量推入：**11 个分支 + 30 个 tag**（2026-11-22 23:15 实测；分支数随 CI/文档分支增减，以 `git ls-remote --heads` 为准）；`main` = `bb564e6` |
| GitHub ↔ Gitea 自动镜像 | 未配置；当前是一次性全量推送，之后靠人工同步 |

- **现状（2026-11-22 核对）**：`one/localmind-x` **已经建好，内容也全量推完了**——
  仓库 id `109`，2026-11-22 15:01:47Z 创建，私有，默认分支 `main`。
  GitHub 侧的内容已全部推入正本：**11 个分支 + 30 个 tag**（2026-11-22 23:15 实测）；
  `main` = `bb564e6`，CI 分支 `ci/codemagic-gitea-source` = `e8dab9b`，
  验收分支 `docs/codemagic-acceptance` = `b232502`（两端 commit hash 一致）。
  核对方式：`git ls-remote --heads git@git.headwindx.top:one/localmind-x.git` → 11 个分支；
  `git ls-remote --tags …` → 30 个 tag。
- **仓库已就位，CI 还差控制台两步**：Codemagic 还没接这个仓（Deploy Key 没加、webhook 没配），
  所以现在推 main 还不会自动触发构建。这两步见 §2.2 / §2.3，是 Henry 在控制台点几下。
- **接 Gitea 不会断 CI**：Codemagic 的 **Add application → HTTPS/SSH** 支持任意自建 Git 仓库
  （见 §2.2）。`codemagic.yaml` 一行都不用改，把应用接上即可。
- **过渡做法**：控制台还没接 Gitea 时，可以先按 §2.1 用 GitHub 授权接入跑通第一次构建
  （跑的是同一份 yaml、同一套脚本）；接好 Gitea 后在 Codemagic 里把应用改指 Gitea
  （§2.2 + §2.3），之后 GitHub 侧只保留只读备份角色。
- 网络前提（2026-11-22 实测）：`code.headwindx.top` 走 Cloudflare，HTTPS 443 可达；
  `git.headwindx.top` 指向 `107.191.113.17`，22 端口直连公网（SSH banner 正常）。
  Codemagic 的构建机需要能出网到这两个地址；若 Cloudflare 侧对未知出口 IP 有拦截，把
  Codemagic 的出口段加白名单（Codemagic 文档：接入自建仓库时需放行其出口 IP 与 SSH/HTTPS 端口）。

## 1. 一次性前置条件（Apple 侧，只做一次）

### 1.1 账号

| 条件 | 说明 | 取值来源 |
| --- | --- | --- |
| Apple Developer Program 付费账号 | $99/年；免费账号不能上传 TestFlight | [developer.apple.com/programs](https://developer.apple.com/programs/enroll/) |
| App Store Connect 访问权限 | 上面账号自带 | [appstoreconnect.apple.com](https://appstoreconnect.apple.com) |
| 10 位 Team ID | 签名用，CI 里填进 `XCODE_TEAM_ID` | [developer.apple.com/account](https://developer.apple.com/account) → Membership details → Team ID |
| Codemagic 账号 | [codemagic.io](https://codemagic.io) 注册，按构建分钟计费 | — |

> 计费参考：macOS M2 实例约 $0.015–0.04/分钟，一次 iOS 构建约 20–40 分钟（≈ $0.5–1.5）。
> 免费额度是否覆盖 macOS 机型以 [codemagic.io/pricing](https://codemagic.io/pricing) 实际显示为准。

### 1.2 注册 App ID（Bundle ID）

位置：[developer.apple.com → Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources/identifiers/list/bundleId) → Identifiers → 「+」

| 字段 | 填什么 | 说明 |
| --- | --- | --- |
| Type | **App IDs → App** | |
| Description | `LocalMind X` | |
| Bundle ID | **Explicit**：`top.headwindx.localmind` | **必须与 `codemagic.yaml` 的 `environment.ios_signing.bundle_identifier` 完全一致** |
| Capabilities | 勾 **Push Notifications** | 仓库 `ios/Runner/Runner.entitlements` 里有 `aps-environment`，不勾首次导出会报描述文件缺 capability |

### 1.3 创建 App Store Connect API Key（.p8）

位置：[App Store Connect → Users and Access → Integrations → App Store Connect API](https://appstoreconnect.apple.com/access/integrations/api) → **Team Keys** → 「+」

| 字段 | 填什么 | 说明 |
| --- | --- | --- |
| 名称 | `codemagic-ci` | 只在 App Store Connect 侧的名字，随便起；**Codemagic 侧的名字是另一回事，见 §3.1** |
| 角色 | **App Manager** | 签名（建描述文件）+ 上传必需；比 Admin 权限小，够用 |

生成后**立刻下载 `.p8`**（只能下载一次），并记下：

- **Key ID**（10 位，如 `ABC1234567`）
- **Issuer ID**（页面顶部 UUID）

> **不要把 `.p8` / Key ID / Issuer ID 发到聊天、Issue 或仓库里**，只在 Codemagic 控制台上传。

### 1.4 建 App 记录

位置：[App Store Connect → Apps](https://appstoreconnect.apple.com/apps) → 「+」→ New App

| 字段 | 填什么 |
| --- | --- |
| Name | `LocalMind X`（全球唯一，`LocalMind` 已被原作者占用） |
| Language | Simplified Chinese |
| Bundle ID | 选 §1.2 建的 `top.headwindx.localmind` |
| SKU | `localmind-x` |

> 上传后要在这条记录下找 **数字 Apple ID**（App 页面 → App 信息 / URL 里的 `id`），§3.2 要用。

### 1.5 建内部测试组

位置：[App Store Connect → TestFlight → LocalMind X](https://appstoreconnect.apple.com/testflight) → 「+」→ **Internal Testing**

- 分组名必须叫 **`Internal Testers`** —— 与 `codemagic.yaml` 的 `publishing.app_store_connect.beta_groups` 一字不差。
- 再把要测试的 Apple ID 加进 [Users and Access](https://appstoreconnect.apple.com/access/users)（角色 Developer 即可）。
- 内部测试组**不需要 Beta App Review**，构建处理完就能装。

## 2. Codemagic：把仓库接进来

### 2.1 过渡/备份用法：GitHub 授权接入

1. [codemagic.io](https://codemagic.io) → **Sign up with GitHub**（用 `dehengxu` 授权，最省事）。
2. **Add application** → 选 **GitHub** → 授权 → **Install GitHub App**（选账号 `dehengxu` → 只选 `localmind-x` 或全部都行）→ **Next: Select repository** → 选 `dehengxu/localmind-x`。
3. 项目类型选 **Flutter App** → **Finish: Add application**。

> 正本仓已经在 Gitea 建好（§0），GitHub 源已降级为只读备份；这条路只作**过渡**——
> 控制台还没接 Gitea 时，用它跑通第一次构建（跑的是同一份 yaml）。

### 2.2 正本方式：Gitea HTTPS/SSH 接入

自建 Gitea 没有官方 App，只能走 Codemagic 的 **HTTPS/SSH** 方式（Codemagic 官方文档：
"Apps that cannot be added via a repository integration can be cloned from a URL. Authentication for
private repositories can be set up via HTTPS or SSH."）：

1. **Add application** → 选 **HTTPS/SSH** → **Next: Select repository**。
2. **Repository URL** 填：`git@git.headwindx.top:one/localmind-x.git`
   （Gitea 仓库页 → **Clone** → SSH 那一行；注意是 `git.headwindx.top`，不是 `code.headwindx.top`）。
   想走 HTTPS 就填 `https://code.headwindx.top/one/localmind-x.git` + 用户名 + 私人令牌。
3. 点 **Generate key pair** → Codemagic 生成 4096 位密钥对 → **Copy to clipboard** 复制**公钥**。
   （也可以自己上传私钥；用 Generate key pair 更简单，公钥放 Gitea 即可。）
4. 到 Gitea 仓库 → **Settings → Deploy Keys → Add Deploy Key**：
   - Title：随意（如 `codemagic`）
   - Key：粘贴刚才的公钥
   - 权限：**只读就够**（Codemagic 只 clone/fetch，不往仓库写）；`Write` 只在以后想让 CI 回推 commit 时才需要。
5. 回到 Codemagic → **Next: Select repository** → 选 `one/localmind-x` → 项目类型 **Flutter App** → **Finish: Add application**。
6. 不要在第 3 步之后关掉弹窗，否则要重新生成密钥对。

> 公钥给的是 Gitea **仓库级** Deploy Key，不是用户级 SSH key；换机器/换仓库要重新加一份。
> Codemagic 侧换密钥：应用设置 → **Repository settings → Repository URL → Change repository**。

### 2.3 自动触发：Gitea webhook

Codemagic 官方口径（"Starting builds automatically with codemagic.yaml"）：
**"For repositories added via SSH or HTTP/HTTPS … you would have to set up webhooks manually."**
所以接了 Gitea 之后，自动触发要手配：

1. 在 Codemagic 打开**这个应用**（不是团队设置页）→ **Webhooks** → 复制 **payload URL**。
   官方原文："Webhook settings are found in your app settings, not your team settings … Go to the
   Webhooks section and copy your payload URL."（payload URL 从控制台复制，不要自己拼 `api.codemagic.io/hooks/<appId>`。）
2. Gitea 仓库 → **Settings → Webhooks → Add Webhook**：
   - Target URL：粘贴刚才复制的 payload URL
   - Content type：`application/json`（Codemagic 官方：json 与 x-www-form-urlencoded 都支持）
   - Events：勾 **Push**（PR 事件 Gitea 侧不用勾，Codemagic 的 PR 构建是 GitHub 专属）
   - Active：勾上 → **Add Webhook**
3. 校验：Codemagic 应用页 → **Webhooks** 标签 → **Recent deliveries** 能看到这次推送的记录。

> **已核（2026-11-22 证据）**：Codemagic 判断事件类型看的是 **GitHub 兼容请求头**，不是 payload 正文。
> ① Gitea 官方文档《Webhooks → Delivery headers》列明它同时发 `X-Gitea-Event` 与 **`X-GitHub-Event`**
>    （GitHub-style compatibility headers），push 事件的 `X-GitHub-Event: push`；
> ② Codemagic 官方知识库《Why are tag events not triggering automatic builds with Codeberg?》原文：
>    "while **push events successfully trigger automatic builds**, tag events do not work"，
>    根因是 Codeberg 在 tag 事件里发 `X-Github-Event: create`，而 Codemagic 只认 push / pull_request。
>    Codeberg 是 Gitea 的分支，同源同头。
> ⇒ **Gitea 推 main（push）能触发构建；tag 事件不触发是已知限制且无解，本项目只用 push，不受影响。**
>
> 兜底（万一首次推送没触发）：① 控制台手动 **Start build**（同一 workflow、同一脚本）；
> ② Builds API（`api.codemagic.io/builds`）触发；③ 给 `codemagic.yaml` 加 `scheduling` 定时跑。

### 2.4 选 workflow

应用设置 → **Workflow** → 勾 **List user workflows / based on codemagic.yaml** → 选中 `ios-testflight`。
构建触发已配置为「push 到 main」（触发源见 §2.3），也可以在应用页手动 **Start build**。

## 3. Codemagic：填签名与应用配置

### 3.1 建 Developer Portal 集成（App Store Connect API Key）

位置：**Team settings → Integrations → Developer Portal** → **Manage keys**

| 字段 | 填什么 | 取值来源 |
| --- | --- | --- |
| **App Store Connect API key name** | **`App Manager`** | **字段名必须一字不差**：与 `codemagic.yaml` 的 `integrations.app_store_connect` 一致。**依据**：同一 Codemagic 账号、同一 Apple 团队下的 `one/ToMobileScreen`，其 master 分支 `codemagic.yaml` 第 18 行写的就是 `app_store_connect: App Manager`（2026-11-22 从 `code.headwindx.top` 读到原文），这条集成已在跑通 |
| Issuer ID | §1.3 的 Issuer ID | appstoreconnect.apple.com → Users and Access → Integrations |
| Key ID | §1.3 的 Key ID | 同上 |
| API key | 上传 `.p8` | §1.3 下载的文件（复用现有集成就**不用再传**） |

**控制台里没有 `App Manager` 这个名字怎么办**（两条路二选一，改完必须同步改 yaml 那一行）：

- 路径 A（推荐）：Henry 在 **Team settings → Integrations → Developer Portal → Manage keys → Add another key**
  新建一条 ASC API Key 集成，**`App Store Connect API key name` 一字不差地写 `App Manager`**，
  填 §1.3 的 Issuer ID / Key ID / 上传 `.p8` → Save。yaml 不用动。
- 路径 B：直接复用现成那条 `App Manager` 集成（同一 Apple 团队，`.p8` 不用重新生成）。
- 路径 C：想换个名字（例如 `LocalMind ASC`）→ 控制台新建/改名 **并且** 把 `codemagic.yaml` 的
  `integrations.app_store_connect` 改成同一个字符串。两边对不上，构建会在取签名时直接失败。
- 控制台里改了名字，yaml 这一行必须同步改（反之亦然），这是硬要求。

> 注意区分两个「App Manager」：**Codemagic 集成名**（= yaml 里那行）要用 `App Manager`；
> **ASC 那条 API Key 的角色**也必须是 `App Manager`（权限位，§1.3）。前者是名字，后者是权限。

### 3.2 应用级环境变量

位置：应用设置 → **Environment variables**

| 变量名 | 值 | 必填？ | 取值来源 |
| --- | --- | --- | --- |
| `XCODE_TEAM_ID` | 你的 10 位 Team ID | 可选但建议填 | developer.apple.com/account → Membership details |
| `APP_STORE_APPLE_ID` | App 的**数字** Apple ID | 建议填 | App Store Connect → Apps → LocalMind X → App 信息里的 `id` |

- `XCODE_TEAM_ID`：CI 用它覆盖 `project.pbxproj` 里的 `DEVELOPMENT_TEAM = "";`；不填也能构建（签名团队由
  `xcode-project use-profiles` 按证书/描述文件决定），但填了更可控。格式必须是 10 位字母数字，填错 CI 会直接报错退出。
- `APP_STORE_APPLE_ID`：填了以后，构建号会参考「App Store Connect 上该 app 已用过的最大构建号」
  （§3.6 的结论）。不填也能构建，但构建号只按 Codemagic 计数与 `pubspec.yaml` 推导，
  手工上传过的构建可能撞号（TestFlight 会拒收重复构建号）。

### 3.3 环境变量组 `app_store`（可选）

`codemagic.yaml` 里引用了一个叫 `app_store` 的环境变量组，里面放 ASC 凭据
（`APP_STORE_CONNECT_PRIVATE_KEY` / `APP_STORE_CONNECT_KEY_IDENTIFIER` / `APP_STORE_CONNECT_ISSUER_ID`），
供构建脚本调 `app-store-connect` CLI 查最新构建号（CLI 认这三个环境变量名）。

- 你的 Codemagic 账号里**已经有**这个组（`one/ToMobileScreen` 在用）→ 不用动。
- 没有 → 把 `codemagic.yaml` 里 `environment.groups` 那两行删掉，构建号自动退回 `PROJECT_BUILD_NUMBER`，不影响其它环节。
- 如果 §3.1 走了路径 A（新建了新的 `.p8`），`app_store` 组里的三项也要换成**同一把新钥匙**，
  否则 CLI 用旧钥匙查号（钥匙与 App 记录不匹配时查询会失败，构建会退化成只按 Codemagic 计数）。
  注意这个组是账号级共享的 —— 直接改它会影响 `one/ToMobileScreen` 的构建；
  稳妥做法是新建一个组（例如 `localmind_asc`）并把 `codemagic.yaml` 的 `groups` 改成引用新组。

### 3.4 Team ID 与工程里的占位符

仓库里的 `ios/Runner.xcodeproj/project.pbxproj` 有 3 处 `DEVELOPMENT_TEAM = "";`（Profile / Debug / Release）。
这是**故意的空占位**：原作者的 Team ID 已移除。

- 本地 Xcode 打开工程 → 选中 **Runner** target → **Signing & Capabilities** → 勾 **Automatically manage signing** → 选你的 Team。
  选完 Xcode 会把这 3 处写成你的 Team ID；**这个改动不要提交**（它含你的 Team ID，且属于本机配置）。
- CI 不依赖上面的本地操作：CI 会用 `XCODE_TEAM_ID` 覆盖这 3 处，再由 `xcode-project use-profiles` 按
  App Store Connect 集成里的证书 + 描述文件把签名配置写回工程。

### 3.5 首次构建

应用页 → **Start build** → 分支选 `main` → workflow 选 `ios-testflight` → 开始。

流水线依次执行：

1. `Override Xcode development team` —— 有 `XCODE_TEAM_ID` 就覆盖工程里的 3 处 Team ID；没配就跳过并打 warning。
2. `Set up code signing settings on Xcode project` —— `xcode-project use-profiles`：从 App Store Connect 集成取
   distribution 证书 + 描述文件（按 `bundle_identifier: top.headwindx.localmind`），写进工程签名设置，
   并把生成的导出配置写到 `$HOME/export_options.plist`（下一步要用它）。
3. `Get Flutter packages` —— `flutter pub get`。
4. `Install pods` —— 在含 `Podfile` 的目录跑 `pod install`（仓库里是 `ios/Podfile`）。
5. `Build IPA` —— `flutter build ipa --release --build-number=<号> --export-options-plist=<路径>`。构建号算法见 §3.6。
   导出配置（ExportOptions.plist）按**确定性规则**选，日志里必有一行 `export options = <路径>`：

   ① 首选 `xcode-project use-profiles` 生成的 `$HOME/export_options.plist`
   （Codemagic 构建机上就是 `/Users/builder/export_options.plist`；cli-tools 的 `use-profiles` 默认就写这个路径，
   Codemagic 官方 Flutter 示例用的也是同一个路径）；
   ② 其次 `$CM_BUILD_DIR/export_options.plist`；
   ③ 都没有 → 用 `XCODE_TEAM_ID` + 已安装的 App Store 描述文件**现场生成**一份完整配置
   （method `app-store` / `teamID` / signingStyle `manual` / `provisioningProfiles` 映射到 `top.headwindx.localmind`）；
   ④ 仍然拿不到 → 脚本 `::error::` 退出（rc≠0），日志写清下一步该查什么，不会带着半份配置往下跑。

   **不再退回仓库里的 `ios/ExportOptions.plist`**。那份只有 method / signingStyle / destination 三个键，
   没有 `teamID`、没有 `provisioningProfiles` 映射：静默用它，失败点会落在 xcodebuild 的导出阶段，
   离根因（use-profiles 没产出配置、集成名对不上、App ID 没注册）很远；
   而且它带 `destination=upload`，xcodebuild 会先上传一次，之后 Codemagic 的 `submit_to_testflight`
   再传同一个 build 号 → 重复上传被 Apple 拒。该文件现在只是留在仓库里的历史文件，CI 不再读它。
6. 发布：`publishing.app_store_connect`（`auth: integration`）→ 上传并提交到 TestFlight 组 **`Internal Testers`**。

### 3.6 构建号（bundle version）策略：结论与理由

**结论：保留「多源取最大值再 +1」的方案，并把 ASC 查询从「第一个命令成功就用」改成「三个命令全查、取最大」；
不照抄参考项目的单一来源写法。**

`codemagic.yaml` 里实际的取值顺序：

```
BUILD = max( PROJECT_BUILD_NUMBER,
             pubspec.yaml 的 build 号 + 1,     # 现在是 1.9.1+96 → 97
             ASC 上该 app 已用过的最大构建号 + 1 )   # 需要 APP_STORE_APPLE_ID + 环境变量组 app_store
```

ASC 侧三个命令都查一遍、取返回数字的最大值（命令名取自 Codemagic 官方 CLI 文档，
`app-store-connect` 的三个 action 都存在）：

- `get-latest-app-store-build-number` —— 参考项目 `one/ToMobileScreen` 在用的那一个（App Store 上最高版本的最新 build 号）
- `get-latest-build-number` —— 该 app 跨 TestFlight/App Store 的最高号
- `get-latest-testflight-build-number` —— TestFlight 上的最高号

理由：

1. **单一来源都不可靠。** Apple 要求同一 marketing version 的 build 号严格递增。
   只用 `PROJECT_BUILD_NUMBER`：新应用的计数从 1 起，手工上传过的构建不在计数里 → 首次上传必被拒。
   只用 ASC 最新号：覆盖不了 `pubspec.yaml` 已声明的号，也可能与并发构建撞号。
2. **参考项目能跑通，是因为前提不同。** `one/ToMobileScreen` 用的是单一来源
   `get-latest-app-store-build-number + 1`，它没有「pubspec 已声明 build 号 + 手工上传过」的包袱。
   本仓库 `pubspec.yaml` 已经是 `1.9.1+96`，直接照抄会从 1 起号 → 必被拒。
3. **宁可跳号，不要撞号。** 撞号 = 构建失败 + 重跑 + 又跳一次；跳号只是浪费号段。
   所以取最大而不是取最小。
4. **不引入 `agvtool`。** 参考项目用 `agvtool new-version -all`（原生 Xcode 工程）；
   本项目是 Flutter，用 `flutter build ipa --build-number=` 直接写 `CFBundleVersion` 更直接。

## 4. 验收

1. Codemagic 构建日志全绿，结尾有上传 App Store Connect 成功的提示，Artifacts 里有 `build/ios/ipa/*.ipa`。
2. App Store Connect → TestFlight → LocalMind X → 构建列表出现新 build，处理完成（约 5–15 分钟）后状态变 **Ready to Test**。
3. iPhone 装 TestFlight App → 登录已加入 `Internal Testers` 的 Apple ID → 出现 LocalMind X → Install。
4. 内部测试不需要 Beta 审核，所以到这一步应该已经能装上。

## 5. 排错表

| 现象 | 处理 |
| --- | --- |
| 构建报 `Flutter version 3.44.2 not available` | Codemagic 版本列表落后：`codemagic.yaml` 的 `environment.flutter` 改成 `stable`，或改成控制台 Flutter 下拉里有的版本（仓库 `.fvmrc` 锁的是 3.44.2） |
| `No profiles for 'top.headwindx.localmind' were found` | App ID 没注册 / 勾了 Push Notifications 但描述文件没有该 capability → 重做 §1.2；或 §3.1 的集成没连上 |
| 签名报 team 不符 / `requires a development team` | `XCODE_TEAM_ID` 没配或格式不对（必须 10 位字母数字）→ §3.2；CI 也会把该错误直接打出来 |
| 401 / 403（ASC 相关） | `.p8` / Key ID / Issuer ID 不匹配，或角色不是 App Manager → 重做 §1.3 / §3.1 |
| 提示找不到 App Store Connect 集成 / 签名材料为空 | `integrations.app_store_connect` 的名字和控制台里的不一致 → §3.1（控制台改名必须同步改 yaml 这一行） |
| 上传被拒：`The bundle version must be greater than the previously uploaded version` | 构建号撞号：配 `APP_STORE_APPLE_ID`（§3.2）+ 环境变量组 `app_store`（§3.3），让号取自 ASC 最大号 + 1；不配则至少靠 `pubspec.yaml` 的 build 号兜底 |
| 导出配置没拿到（日志出现 `export options = (none)` + `::error::use-profiles 没有生成导出配置…`） | 按 §3.5 第 5 步的分支走：① 看 `xcode-project use-profiles` 那一步的日志（`integrations.app_store_connect` 的集成名要和控制台一致）；② 补 `XCODE_TEAM_ID`（§3.2，10 位 Team ID）；③ 确认 `environment.ios_signing.bundle_identifier` 与已注册的 App ID 一致。**CI 不会再退回仓库里那份 3 键 plist**，所以这个错会直接停在构建脚本里 |
| 上传被拒：`This upload is invalid / 已经上传过同一个 build 号` | 导出配置里带了 `destination=upload` → xcodebuild 已上传一次，Codemagic 的 `submit_to_testflight` 又传一遍。CI 自己生成的导出配置不带 `destination`；若日志显示用的是 use-profiles 生成的那份，确认里面没有 `destination=upload` |
| `pod install` 失败 | CocoaPods 仓库拉取超时；重跑构建，或在本地 `cd ios && pod install` 后再触发 |
| Codemagic 找不到 `app_store` 环境变量组 | 删掉 `codemagic.yaml` 里 `environment.groups` 两行 → §3.3 |
| 上传后 TestFlight 看不到 | App 记录的 Bundle ID 与构建包不一致 → 核对 §1.2 与工程里的 `PRODUCT_BUNDLE_IDENTIFIER`（应为 `top.headwindx.localmind`） |
| Codemagic 里保存仓库地址时报连不上 | 出网被拦：确认构建机能到 `git.headwindx.top:22` 与 `code.headwindx.top:443`；必要时放行 Codemagic 出口 IP → §0 |
| 推 main 没触发构建 | 仓库已在 Gitea（§0），多半是 webhook 还没配 → §2.3（含三条兜底：手动 Start build / Builds API / 定时） |

## 6. 配置对照表（yaml ↔ 控制台）

| `codemagic.yaml` 位置 | 必须一致的值 | 控制台位置 |
| --- | --- | --- |
| `integrations.app_store_connect` | `App Manager` | Team settings → Integrations → Developer Portal → API key 名字 |
| `publishing.app_store_connect.beta_groups` | `Internal Testers` | App Store Connect → TestFlight → 内部测试组名 |
| `environment.ios_signing.bundle_identifier` | `top.headwindx.localmind` | developer.apple.com → App ID |
| `environment.flutter` | `3.44.2`（同 `.fvmrc`） | Codemagic Flutter 版本 |
| `environment.instance_type` | `mac_mini_m2` | Codemagic 构建机型 |
| `environment.groups` | `app_store`（可选） | Codemagic 应用 → Environment variables → 组 |
| `XCODE_TEAM_ID` | 10 位 Team ID | 应用 → Environment variables |
| `APP_STORE_APPLE_ID` | 数字 Apple ID | 应用 → Environment variables |
| （不在 yaml 里）Repository URL | `git@git.headwindx.top:one/localmind-x.git` | Codemagic 应用 → Repository settings → Repository URL |
| （不在 yaml 里）webhook Target URL | 从 Codemagic 应用设置 → **Webhooks** 复制的 **payload URL**（不手拼） | Gitea 仓库 → Settings → Webhooks |

## 7. 仍需 Henry 本人提供的项

| # | 需要什么 | 从哪取 | 用来做什么 | 不给的后果 |
| --- | --- | --- | --- | --- |
| 1 | **10 位 Team ID** | developer.apple.com/account → Membership details | 填 `XCODE_TEAM_ID`，CI 覆盖 pbxproj 里的空占位 | 不给也能构建（签名团队由描述文件决定），但工程里 Team 仍是空占位 |
| 2 | **ASC 的数字 Apple ID**（App 记录 id，纯数字） | App Store Connect → Apps → LocalMind X → App 信息 | 填 `APP_STORE_APPLE_ID`，构建号才能参考 ASC 上的最大号 | 构建号只按 Codemagic 计数 + pubspec 推导，可能被 Apple 以「版本号必须更大」拒收 |
| 3 | **Codemagic 控制台里 ASC 集成的真实名字** | Team settings → Integrations → Developer Portal → Manage keys | 与 yaml 的 `integrations.app_store_connect` 对齐 | yaml 与控制台对不上 → 取不到签名材料，构建直接失败。当前 yaml 写的是 `App Manager`（依据见 §3.1） |
| 4 | **Gitea API 令牌**（已非必需，建议补一条） | Gitea → Settings → Applications → Generate New Token（勾 `repo` / `write:repository`），令牌不经过聊天 | 建仓已完成、正本内容已全量推送；后续用 API 做加 Deploy Key 等操作时用（现有可用凭据是账号口令走 HTTP Basic，不是 API token） | 不给不影响 CI 跑通；只是后续 API 操作要走别的路 |
| 5 | **Codemagic 应用接入 Gitea 的密钥方式**（Henry 在控制台点 Generate key pair 或上传自有私钥） | Codemagic → Add application → HTTPS/SSH | 决定用哪种密钥方式拿公钥去 Gitea 加 Deploy Key | 拉不到私有仓库 → 构建在 clone 阶段失败 |
| 6 | **Codemagic 应用的 payload URL** | Codemagic 应用设置 → **Webhooks** → 复制 payload URL | 贴进 Gitea 仓库 Settings → Webhooks 的 Target URL | 推 main 不自动触发构建（只能手动 Start build） |
| 7 | **内部测试组名** | App Store Connect → TestFlight | 与 yaml 的 `beta_groups` 一致 | 上传成功但发不进测试组 |
