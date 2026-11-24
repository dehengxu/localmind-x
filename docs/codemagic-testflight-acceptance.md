# Codemagic 首次 TestFlight 构建 · 验收测试计划

> 归属：QA（Rigel）｜被验收对象：Indigo 的 CI 配置与首次构建｜回收人：Yuki（PM）
> 被测物基线：**触发时 `docs/gitea-repo-live` 的实时 tip**（**本文不写死被测物 SHA**）。取法，每次构建前留档、两条命令的第一个字段必须逐字相同：`git ls-remote origin refs/heads/docs/gitea-repo-live`（GitHub `dehengxu/localmind-x`）与 `git ls-remote gitea refs/heads/docs/gitea-repo-live`（Gitea 源码仓库 `one/localmind-x`）。**本文唯一保留的写死 commit SHA 是 `bb564e6`** —— 它是 `main` 的**冻结标记**（Henry 2026-11-22 决定：LocalMind X 独立发展、不回流上游，main 不再动），不随任何分支推进而漂，必须写死才可核；其余一律写「触发时 tip」。**为什么钉 `docs/gitea-repo-live`（最关键的一条）**：main 上仍是旧 yaml（`LocalMind ASC API Key` / `$FCI_BUILD_DIR` / flutter 3.44.0，无 `use-profiles` / `ios_signing`），**按 main 构建必然失败**；承载 CI 配置的是 `docs/gitea-repo-live`（`triggering.branch_patterns` 的唯一条目），**被测物锚 = 该分支触发时的 SHA —— 不是 main 的 SHA，也不是本文任何一次写作时的 SHA**。
> 历史沿革（**只记历史提交，不作判据基准**；判据一律以触发时 tip 为准，历史 SHA 只在本链出现）：`main bb564e6` → `ci/codemagic-first-run @ 30c2f68`（对齐 Flutter 3.44.2、pbxproj Team ID 置空；CI 自此有 `use-profiles` / `ios_signing`、不再用 `$FCI_BUILD_DIR`，yaml 与 `.fvmrc` 的 flutter 版本不一致也在这一带修掉）→ `ci/codemagic-gitea-source @ ae3a247`（构建源改 Gitea、ASC 集成名对齐）→ `@ 72afcd0`（`PROJECT_BUILD_NUMBER` 为空时的构建号兜底）→ `@ e8dab9b`（webhook 口径定稿）→ `docs/gitea-repo-live @ 0d48f33`（刷新仓库计数，yaml 仅加注释）→ `@ a11f7fe`（导出配置改为确定性选取）→ `@ 845c8d1`（报错示例改中性占位；导出配置定为**四级确定性选取**、明确不读仓库内 `ios/ExportOptions.plist`）→ `@ 6cfa317`（触发分支由 `main` 改为 `docs/gitea-repo-live`，main 冻结）→ `@ 265ca06`（措辞改成独立项目语言，只动文案不动 CI 逻辑）→ `@ 9e0d9df`（`environment.xcode` 由 `latest` 钉成 `26.6`）→ `@ 423dc62`（首版移除 `ios/Runner/Runner.entitlements` 的 `aps-environment` 两行 + 同步文档口径；`codemagic.yaml` 零改动）→ `@ 4a743e7`（`environment.xcode` 的值加引号，`+2/−2`、行数不变，解析值仍 26.6、类型 float→str）→ `@ 291bb14`（§3.5「以后要开 Push」更正为「这三步只是必要条件」+ 文末新增 §8 已知限制；只改 `docs/codemagic-testflight.md` `+5/−1`）。**父链**：`291bb14 ← 4a743e7 ← 423dc62 ← 9e0d9df ← 265ca06 ← 6cfa317 ← 845c8d1 ← a11f7fe ← 0d48f33 ← f2e1616`（`72afcd0` 构建号兜底、`ae3a247` 交办点）。`291bb14` 是本文写作时看到的 tip，**它本身不是判据**：tip 前移后本文全部判据照旧有效，只需重跑附录 A 的复核命令。
> 仓库侧基线（2026-11-23 本轮 `git ls-remote` 实测）：Gitea 源码仓库 `one/localmind-x` **已创建（私有，默认分支 `main`）并全量推送** —— `HEAD`/`main` = `bb564e6`（冻结标记）、`docs/gitea-repo-live` = 触发时 tip（以实时为准）、`ci/codemagic-gitea-source` 与 `docs/codemagic-acceptance` = 各自 tip（以实时为准）；30 个 tag、11 个分支（tag / 分支数只作现状记录，随分支增减，**不作判定依据**）；与 GitHub 侧同名分支 commit 逐一相同。**执行时的两条硬要求**：① 基线若前移，只按前移后的 tip 执行，**不改本文口径、不重锚本文**；② 每次都按 G6.3 用**触发当时的实时 tip** 比对 commit SHA —— 日志 SHA 等于触发时 `docs/gitea-repo-live` 的 SHA 就是**有效构建**，不得因 SHA 与本文任何字面值不同而误记无效。**Codemagic 控制台已重接该仓库：仓库级 Deploy Key（只读，id=1）与 push webhook（id=4，Active）均已配，自动触发已实证（见 G6.1 / G6.2 与 R5）；Gitea 仓保持私有，对外可见的公开镜像与反馈入口在 GitHub**。
> 被测物（不在本计划改动范围内，归 Indigo 改、Henry 复核 diff）：`codemagic.yaml`、`ios/Runner.xcodeproj/project.pbxproj`、`docs/codemagic-testflight.md`
> 本文件只描述“怎么判定通过”，不含任何密钥、Team ID 真实值或 .p8 内容。

## 0. 判定总则

- 每条用例写成「当……时，……应为……」的可测句；判定只认三类证据：**① Codemagic 构建日志原文**、**② 构建产物（.ipa / .xcarchive）的机器可读元数据**、**③ App Store Connect / 真机上的可见状态**。口头描述不算证据。
- “全绿”定义：该次构建状态为 `finished`（build success），且 `scripts` 中每一步状态为 success、`publishing` 无 failed 子步骤、日志中 `::error::` 出现次数为 0。
- **本文的判据一律写成可执行命令，不写死行号**（行号会随任何一次改动漂，命令不会）。每条命令都注明了**写作时在当前 tip 上的实测输出**；输出与标注不符时，先按 G6.3 确认构建的是不是触发时的 tip —— 若是，说明被测物已变，按附录 A 的复核命令重核，不要拿旧期望值判失败。凡写 `docs/codemagic-testflight.md` / `codemagic.yaml` / `project.pbxproj` / `ios/Runner/Runner.entitlements` 而未注明版本的，均指**触发时 `docs/gitea-repo-live` tip 上的那一份**。
- 所有日志取证命令均在构建日志的 “Raw log / 完整日志” 下载后本地执行，避免网页搜索漏行：

  ```bash
  # 假设下载为 build.log
  grep -nE '::error::|BUILD FAILED|error:' build.log
  grep -n 'DEVELOPMENT_TEAM overridden' build.log
  grep -nE 'Flutter [0-9]+\.[0-9]+\.[0-9]+' build.log | head
  # 构建号三条来源（基线 yaml 的固定日志格式）
  grep -nE '^ASC get-latest-(app-store|testflight)?-?build-number = |^CFBundleVersion = |::notice::命令 |::warning::取 ASC 最新构建号失败' build.log
  ```

- 下表 **门禁编号** 决定该用例“现在能不能跑”：

| 门禁 | 内容 | 谁能做 |
| --- | --- | --- |
| G1 | Codemagic 应用已按 **Gitea 仓库**接入：Add application → HTTPS/SSH，Repository URL `git@git.headwindx.top:one/localmind-x.git`（HTTPS 备用 `https://code.headwindx.top/one/localmind-x.git`），workflow 选 `ios-testflight`。**在 Gitea 接入尚未完成时**，可先用 GitHub OAuth 接 `dehengxu/localmind-x` 跑一次构建——但**必须在 Start build 时选 `docs/gitea-repo-live` 分支**（两端该分支同 commit，以 `ls-remote` 实时为准，被测 yaml 才是同一份；选 `main` 就是旧 yaml，按 G6.3 记无效构建）。GitHub 侧**不参与 CI 构建**（同步策略待定，见 `docs/codemagic-testflight.md` §0） | Henry（控制台） |
| G2 | Team integrations → App Store Connect 有一条可用 API Key 集成，**名称与 `codemagic.yaml` 的 `integrations.app_store_connect` 一致（当前值 `App Manager`）**，角色 App Manager。该集成复用 `one/ToMobileScreen` 已在跑通的同一条（依据：同账号同 Apple 团队、其 master 分支 `codemagic.yaml` 里 `integrations.app_store_connect` 的值即 `App Manager`），Henry **不需要**再传 .p8；控制台若没有这个名字，按 `docs/codemagic-testflight.md` **§3.1** 三条路径处理（路径 A 新建同名集成 / 路径 B 复用现成 `App Manager` 那条 / 路径 C 改名并同步改 yaml 一行），改名由 Indigo 改 yaml 一行 | Henry（控制台） |
| G3 | 应用级环境变量与环境变量组，明细见 G3.1–G3.3：`XCODE_TEAM_ID`（10 位，**可选但建议**）、`APP_STORE_APPLE_ID`（**可选**）、`app_store` 变量组（**可选**） | Henry（控制台）；缺 G3.3 变量组时由 Indigo 删 yaml 两行 |
| G4 | Apple 侧：App ID `top.headwindx.localmind` 已注册（**首版不要求勾 Push Notifications**：工程 entitlements 已无 `aps-environment`（复核：`grep -c 'aps-environment' ios/Runner/Runner.entitlements` = 0）、v1 不带 Push，见 T1 失败根因「签名类」与 T2 新增判据；若勾了 Push 而工程未同步，才会在签名阶段报 entitlement 不匹配）；ASC 应用记录已建；TestFlight 内部组名为 `Internal Testers` 且含至少 1 个已接受邀请的 Apple ID | Henry（Apple 侧） |
| G5 | 一台可装 TestFlight 的 iPhone + 已加入 `Internal Testers` 的 Apple ID | Henry / 测试执行人 |
| G6.1 | **Gitea 仓库与访问**：`one/localmind-x` 仓库存在 ✓（**已创建并全量推送**）＋ 仓库级 Deploy Key 加入 Codemagic 公钥（**只读权限就够** —— CI 只 clone/fetch 不回推）。**现状：仓库 ✓ / Deploy Key ✓（只读，id=1，title=codemagic）—— 已就绪** | 已完成（2026-11-24 Gitea API 实测） |
| G6.2 | **自动触发链**：Gitea → Settings → Webhooks → Target URL = 从 **Codemagic 应用设置 → Webhooks 复制的 payload URL**（**不要手拼** `api.codemagic.io/hooks/<appId>`），Content type `application/json`，Events 勾 Push，Active。**现状：控制台已重接 Gitea（应用 `localmind-x`，SSH 私有仓）、webhook 已配（id=4，events=push，active，Target URL 取自 Codemagic 应用 → Webhooks 的 payload URL）—— 已就绪** | 已完成（2026-11-24 实测） |
| G6.3 | **被测物一致性（验收对象成立的先决条件）**：当构建日志中的 commit SHA（`CM_COMMIT` / `CM_COMMIT_ID`，或 clone 阶段的 `HEAD is now at <sha>`、`Checking out <sha>`、`Successfully checked out` 后的 sha）**不等于触发时 `docs/gitea-repo-live` 的 SHA** 时，该次构建记为**无效构建**，不计入 T1–T7 任何判定。**被测物锚 = `docs/gitea-repo-live` 的 SHA，不是 `main` 的 SHA** —— 原因：`main` 已按 Henry 2026-11-22 的决定**冻结在 `bb564e6` 不再动**（LocalMind X 独立发展、不回流上游），main 上仍是旧 yaml（`LocalMind ASC API Key` / `$FCI_BUILD_DIR` / flutter 3.44.0，无 `use-profiles` / `ios_signing`），**按 `main` 建的构建一律记无效构建**（不是失败，不计入 T1–T7），因为验的会是另一份 yaml；`triggering.branch_patterns` 现在唯一条目就是 `docs/gitea-repo-live`，被测物锚就是该分支**触发时的 tip**，两者必须同 commit。**现状：未验证**（尚无第一次构建；`main` 两端恒为 `bb564e6` 即冻结标记，`docs/gitea-repo-live` 两端同 commit、逐次 `ls-remote` 为准） | 执行人每次构建前 `git ls-remote origin refs/heads/docs/gitea-repo-live`（或 `git rev-parse origin/docs/gitea-repo-live`）留档，与日志 SHA 比对 |

**G3 明细（Codemagic 应用设置 → Environment variables）**

| 项 | 变量 / 对象 | 要求 | 留空或缺失时的影响 |
| --- | --- | --- | --- |
| G3.1 | `XCODE_TEAM_ID` | 10 位字母数字（Apple Team ID），**可选但建议** | 脚本打 `::warning::未设置 XCODE_TEAM_ID…` 并跳过覆盖，工程里 3 处 `DEVELOPMENT_TEAM = "";` 保持空占位，签名团队由 `xcode-project use-profiles` 按证书/描述文件决定（T2 走降级口径）。**格式配错**（非 10 位字母数字）仍 `::error::` 退出、构建 failed |
| G3.2 | `APP_STORE_APPLE_ID` | App 的**数字** Apple ID（纯数字；ASC → Apps → LocalMind X → App 信息 / URL 里的 `id`），**可选** | 日志打 `::warning::未设置 APP_STORE_APPLE_ID…`，构建号只按 `PROJECT_BUILD_NUMBER` 与 `pubspec.yaml` build 号 +1 推导；ASC 上已有更大的号会被 Apple 拒收，T4 只能记**部分验证** |
| G3.3 | 环境变量组 `app_store` | `codemagic.yaml` 的 `environment.groups` 引用了它（内含 `APP_STORE_CONNECT_PRIVATE_KEY` / `_KEY_IDENTIFIER` / `_ISSUER_ID`，供 `app-store-connect` CLI 查 ASC 最新构建号，见 `docs/codemagic-testflight.md` **§3.3**）。账号里已有（`one/ToMobileScreen` 在用）→ 不用动（该组是**账号级共享**，直接改里面的钥匙会影响 ToMobileScreen 的构建；若 §3.1 走了路径 A 新建 `.p8`，稳妥做法是另建一组如 `localmind_asc` 并改 yaml 的 `groups`）；**没有 → 由 Indigo 删掉 yaml 里 `environment.groups` 两行**（Henry 侧决定，属 Indigo 改动范围） | 三条 ASC 查询全部失败并打 `::warning::取 ASC 最新构建号失败`，构建号退回计数推导；其余环节不受影响 |

**G6 明细与现状（构建源）**

- 构建源 = 自建 Gitea `code.headwindx.top` → `one/localmind-x`（私有，默认分支 `main`）；GitHub `dehengxu/localmind-x` **不参与 CI 构建**（yaml 注释措辞：角色待定，镜像 / 归档 / 删除三选一）（`docs/codemagic-testflight.md` **§0「仓库与发布源」**；该节标题改名的沿革见文首）。
- **触发分支 = `docs/gitea-repo-live`，不是 `main`**：`main` 已按 Henry 2026-11-22 的决定冻结在 `bb564e6` 不再动（LocalMind X 独立发展、不回流上游），main 上仍是旧 yaml。因此 webhook 的“推哪条分支”与“验哪个 commit”都是 `docs/gitea-repo-live`；**按 main 建的构建记无效构建**。
- 接入方式 = **Add application → HTTPS/SSH → Generate key pair** → 把 Codemagic 生成的**公钥**加到 Gitea 仓库 Settings → Deploy Keys。**权限只读就够**：Codemagic 只 clone/fetch，不往仓库回推，`Write` 只在以后要让 CI 回推 commit 时才需要。
- 自动触发必须**手配 Gitea webhook**（Codemagic 官方口径：SSH/HTTPS 接入的仓库必须手配 webhook）。**webhook 口径自定稿以来未再变过，其后所有改动都落在导出配置 / xcode 版本 / entitlements / 文档口径上，均不涉及 webhook 段，已核（逐条沿革见文首）**：Target URL = 在 **Codemagic 应用 → Webhooks** 复制的 **payload URL**，**不要自己拼** `api.codemagic.io/hooks/<appId>`。触发可行性**已核**：Gitea 推 push 会同时发 `X-Gitea-Event` 与 GitHub 兼容头 **`X-GitHub-Event: push`**，Codemagic 按该头判断事件类型 → **push 可触发**（依据：Gitea 官方《Webhooks → Delivery headers》＋ Codemagic 知识库 Codeberg 一条：push 触发正常、tag 事件因发 `X-Github-Event: create` 而不触发 —— 本项目只用 push，不受影响）。**2026-11-24 直接实证**：推 `8ecf9c2` → Gitea 投递 → Codemagic 收到 `X-Gitea-Event: push`（兼容头 `X-GitHub-Event: push`）、`ref=refs/heads/docs/gitea-repo-live`、`after=8ecf9c21`，响应 **202「Started builds: ios-testflight」** → 生成构建 **#3**；Gitea Webhooks → Recent deliveries 存有该次请求头与 payload。万一首次推送没触发，按 `docs/codemagic-testflight.md` §2.3 三条兜底：控制台 Start build / Builds API / yaml 加 `scheduling`。
- **仓库侧现状（2026-11-24 本轮实测，G6.1 全部满足：仓库 ✓ + Deploy Key ✓）**：`git ls-remote git@git.headwindx.top:one/localmind-x.git` 正常返回 —— `HEAD`/`refs/heads/main` = `bb564e6`、`refs/heads/docs/gitea-repo-live` = 触发时 tip、`refs/heads/ci/codemagic-gitea-source` / `refs/heads/docs/codemagic-acceptance` = 各自 tip（三者一律以 `git ls-remote` 实时为准），共 **30 个 tag**（分支数以 `git ls-remote --heads` 实时为准，本轮实测 11 个；分支数随分支增减而变，**不作为验收判定项**），与 GitHub 侧同名分支 commit 逐一相同。即**仓库已创建并全量推送**（未带凭据的 `GET https://code.headwindx.top/api/v1/repos/one/localmind-x` 返回 **404** 是私有仓库的预期表现，**不等于仓库缺失**；实例可达，`/api/v1/version` 返回 1.26.1）。仓库级 **Deploy Key 已加 Codemagic 公钥（只读，id=1）**、**push webhook 已配（id=4，events=push，active）**（2026-11-24 Gitea API 实测）。Gitea 仓**保持私有**（构建源）；对外可见的公开镜像与反馈入口在 GitHub `dehengxu/localmind-x`（应用内「崩溃上报 / Report a problem / 仓库链接」已改指该公开仓，其 Issues 已开启）。
- **现已就绪（2026-11-24 实测）**：① 仓库 Deploy Key 已加 Codemagic 公钥（只读，id=1，G6.1）；② Codemagic 应用已按 Gitea SSH 接入（G1）；③ Gitea 侧 push webhook 已配（id=4、Active、G6.2）。**G6 整体判就绪**；剩余阻塞不在 G6，而在 Apple 侧签名资产（缺 `top.headwindx.localmind` 的 App Store 描述文件 → 构建 #3 在配置校验阶段即停）与 ASC 应用本体。
- “自动触发”这项已**实证通过**（见 R5：推 `8ecf9c2` → Gitea 投递 webhook → Codemagic 返回 202「Started builds: ios-testflight」→ 生成构建 #3）；T1 的判定仍以真实构建日志为准（构建 #3 因 Apple 侧缺描述文件在配置校验阶段停止，不计入 T1–T7 的失败判定）。

> **执行结论前置**：G1、G2、G4 全部就绪前，**T1–T7 一条都不具备执行条件**（T3–T5 还额外需要 G5）；G3 建议就绪，G3.1/G3.2 留空时按各用例的“降级口径”判；G6.2 决定自动触发能否判定（不影响手动构建）。G6.1 / G6.2 已于 2026-11-24 就绪并实证自动触发，G1（应用已接 Gitea）亦已就绪；仍未就绪的是 G2 / G4 / G5（Apple 侧签名资产、ASC 应用与真机），故当前结论是“计划就绪、执行卡在 Apple 侧资产”。

---

## T1｜构建日志全绿且签名步骤无报错 · 覆盖点①

**用例句**：当触发 `ios-testflight` workflow 构建时（向 Gitea 的触发分支 `docs/gitea-repo-live` push，经 webhook 自动触发；或控制台 Start build 选该分支），该次构建状态应为 `finished`，且完整日志中 `::error::`、`BUILD FAILED`、`error:` 三类字符串出现次数均为 0，签名相关步骤（`Override Xcode development team`、`Set up code signing settings on Xcode project`、`Build IPA`、publishing 阶段）不得出现 `Code Sign error`、`No profiles for`、`Provisioning profile` 失败行。

- **前置条件**：G1、G2、G4 就绪（自动触发项另需 G6.1 的 Deploy Key 与 G6.2 的 webhook）；**G6.3 被测物一致性成立**——构建日志的 commit SHA 必须等于触发时 **`docs/gitea-repo-live`** 的 SHA（构建前 `git ls-remote origin refs/heads/docs/gitea-repo-live` 留档比对，**不是** `main`）；被测 commit 上 `codemagic.yaml` 与附录 A 的一致性核对结果一致（附录 A 的复核命令照抄执行、输出须与标注一致）。
- **操作步骤**：
  1. 触发：推触发分支 **`docs/gitea-repo-live`** → 由 Gitea push webhook 自动触发（G6.2 就绪后）；未配 webhook 时在控制台 Start build **选 `docs/gitea-repo-live`**。注意 `main` 已冻结在 `bb564e6`、且 main 上是旧 yaml，**按 main 构建的一律记无效构建**（见 G6.3），不作判定。webhook 不生效时先用 Start build 兜底，并把“自动触发”单列一条未验证。
  2. 构建结束后下载完整日志。
  3. 执行第 0 节的 grep 取证命令。
  4. 记录 `CM_BUILD_ID`、`PROJECT_BUILD_NUMBER`、`CM_COMMIT` 三个值以便与其他用例对齐。
- **判定口径**：
  - 通过：状态 `finished`；上述 grep 命中数全为 0；日志末尾出现 App Store Connect 上传成功行（形如 `Build uploaded to App Store Connect` / `Uploaded ... to App Store Connect`）。
  - 自动触发（Gitea webhook）：Gitea → Settings → Webhooks → Recent deliveries 有该次 push 记录，且 Codemagic `CM_COMMIT` 等于该次 push 的 commit（与手动构建的判定分开记）。
  - 失败：任一步骤 failed，或 grep 有命中，或构建 `canceled`/`timed out`（`max_build_duration: 90` 到顶）。
  - **无效构建（不计入判定，不是失败）**：日志 commit SHA ≠ 触发时 **`docs/gitea-repo-live`** 的 SHA → 按 G6.3 记为无效构建，改用该分支的正确 commit 重跑后再判定；**若构建来自 `main`（或任何非 `docs/gitea-repo-live` 的分支），一律记无效构建**——main 已冻结在 `bb564e6`、其 yaml 是旧版（`LocalMind ASC API Key` / `$FCI_BUILD_DIR` / flutter 3.44.0、无 `use-profiles` / `ios_signing`），验的不是本计划钉的那份 yaml。
- **失败根因方向**：
  - 签名类：G2 的集成名与 yaml 的 `integrations.app_store_connect: App Manager` 不一致（大小写/空格）；Key 角色不足 App Manager。**Push / capability 已不是首版失败原因**：v1 不带 Push —— `ios/Runner/Runner.entitlements` 已无 `aps-environment`（复核：`grep -c 'aps-environment' ios/Runner/Runner.entitlements` = 0，`plistlib` 解析 `keys=[]`），应用申请的 entitlement 必须是描述文件的子集，不申请就永不冲突；因此 **「App ID 勾 Push Notifications」不是首版构建的失败条件**，G4 的该项从阻塞项降为「以后开 Push 才需要」。**要开 Push 的正确顺序**：① 在 Apple 侧给 App ID `top.headwindx.localmind` 勾上 Push Notifications → ② 把 `ios/Runner/Runner.entitlements` 改回 `aps-environment = production` → ③ 重新构建（描述文件由 `use-profiles` 按新 capability 生成）。**只改 ② 会冲突**（描述文件里还没有 `production`）；**只改 ① 不会冲突**，但推送功能不生效。
  - 依赖类：`flutter pub get` 失败 → 私有依赖/网络，或 Flutter 版本与 pubspec 的 `flutter: ">=3.44.0"` 边界不符。
  - 超时类：`ios/Podfile.lock` 与 Podfile 不一致导致 pod install 重解析；首次无缓存时 90 分钟可能不够（记录实际耗时，超 75 分钟即预警）。
  - 构建脚本类：原 R1（`$FCI_BUILD_DIR` 未定义）**已在 `ci/codemagic-gitea-source` 修复**（沿革见文首，现行 yaml 沿用）。导出配置是**四级确定性选取**（复核：`grep -c 'FCI_BUILD_DIR' codemagic.yaml` = 1 且该唯一一行以 `#` 开头、描述的是 main 上旧 yaml；`grep 'FCI_BUILD_DIR' codemagic.yaml | grep -vc '^[[:space:]]*#'` = 0，即代码里 0 处）：① `$HOME/export_options.plist`（`use-profiles` 生成，构建机上即 `/Users/builder/export_options.plist`）→ ② `$CM_BUILD_DIR/export_options.plist` → ③ 用 `$XCODE_TEAM_ID` + 已装描述文件**现场生成完整配置** → ④ 都没有则 `export options = (none)` + `::error::` 退出 rc≠0。日志定位行：`export options = <路径> (来源：…)` 或 `export options = (none)`。**该行若指向仓库内 `ios/ExportOptions.plist`（或是不带来源标注的裸 `export options = <路径>`）→ 构建的不是触发时 tip 那份 yaml，记无效构建**，按 G6.3 处理。
  - 触发类：push 了但没构建 → 仓库侧已确认存在且可达（`ls-remote` 正常），所以按顺序看：① G6.1 Deploy Key 是否已把 Codemagic 公钥加进去（拉不到私有库 → clone 阶段就失败）；② G6.2 webhook 是否 Active、Target URL 是否为**从 Codemagic 应用 Webhooks 复制的 payload URL**（手拼的 `<appId>` 拼错同样静默不触发）；③ Codemagic 应用是否仍指着 GitHub 源。

## T2｜DEVELOPMENT_TEAM 覆盖生效且产物不含原作者 Team ID · 覆盖点②

**用例句**：当构建执行到 `Override Xcode development team` 步骤时：配了 `XCODE_TEAM_ID` 应出现 `DEVELOPMENT_TEAM overridden to <XCODE_TEAM_ID>`；未配应出现 `::warning::未设置 XCODE_TEAM_ID…` 并跳过覆盖。两种情况下，导出的 `.ipa` 中 `Payload/Runner.app` 的 `TeamIdentifier` 都必须等于 `embedded.mobileprovision` 的 `TeamIdentifier[0]`（即与签名描述文件一致），**且都不应为 `TZX5LG7ZW5`**。

- **前置条件**：T1 已产出可用 `.ipa`（Codemagic Artifacts 里下载）。
- **操作步骤**：

  ```bash
  unzip -q LocalMindX.ipa -d ipa
  codesign -dvvv ipa/Payload/Runner.app 2>&1 | grep -E 'TeamIdentifier|Authority|Identifier='
  security cms -D -i ipa/Payload/Runner.app/embedded.mobileprovision > prov.plist
  /usr/libexec/PlistBuddy -c 'Print :TeamIdentifier' prov.plist
  /usr/libexec/PlistBuddy -c 'Print :Entitlements:application-identifier' prov.plist
  codesign -d --entitlements :- ipa/Payload/Runner.app 2>/dev/null | grep -A1 aps-environment   # v1 期望命中 0（产物不带 aps-environment）
  grep -c 'TZX5LG7ZW5' build.log            # 基线上工程已是空占位，应为 0
  ```

- **判定口径**：
  - 通过：日志出现 `overridden to <XCODE_TEAM_ID>`（G3.1 已配）**或** 出现 `::warning::未设置 XCODE_TEAM_ID`（G3.1 留空，走降级）；产物 `codesign` 的 TeamIdentifier **等于** `embedded.mobileprovision` 的 `TeamIdentifier[0]`（G3.1 已配时还应等于该值）；`application-identifier` 前缀为 `<该 TeamID>.top.headwindx.localmind`；**v1 产物 entitlements 不应含 `aps-environment`**（工程已移除该键，期望命中 0，复核命令见本节失败根因；以后开 Push 才要求 `production`）；`grep -c 'TZX5LG7ZW5' build.log` 为 0。
  - 失败：产物 TeamIdentifier 与签名描述文件不一致；或 TeamIdentifier 仍为 `TZX5LG7ZW5`；或产物 entitlements 出现 `aps-environment`（v1 应为 0 命中；出现即按同节新增判据处置）；或出现 `::error::` 使构建 failed（其中 `XCODE_TEAM_ID` **格式错**属配置问题，记为 G3.1 不合规，不计为 CI 回归）。
  - **新增可测判据（entitlements 与描述文件匹配 · v1 口径）**：当构建执行到签名 / 导出阶段（`Set up code signing settings on Xcode project`、`Build IPA`、导出前的重签）时，完整日志中**不得出现** entitlements 与描述文件不匹配的错误 —— 形如 `doesn't match the entitlements`、`not permitted by provisioning profile`、`Provisioning profile ... doesn't include the aps-environment entitlement`，以及任何提到 `aps-environment` 的 entitlement 报错。**出现任一 → T2 判失败**；根因＝被测物不是触发时 tip 的那份工程（或有人手工把 `aps-environment` 加回 `ios/Runner/Runner.entitlements`）；取证＝错误原文行（前后各留 2 行，共 3–10 行）+ 构建日志里的 commit SHA（须等于触发时 `docs/gitea-repo-live` 的 SHA）。
- **失败根因方向**：
  - `project.pbxproj` 是 **3 处空占位** `DEVELOPMENT_TEAM = "";`（Debug/Profile/Release 各一处），原作者的 `TZX5LG7ZW5` 已移除。**定位一律用计数、不用行号**：`grep -c 'DEVELOPMENT_TEAM = "";' ios/Runner.xcodeproj/project.pbxproj` = 3；`grep -c 'TZX5LG7ZW5' <同文件>` = 0；`grep -c 'CODE_SIGN_ENTITLEMENTS = Runner/Runner.entitlements' <同文件>` = 3（三个配置都指向同一份 entitlements 文件，这正是「首版不带 Push」只需改这一个文件的原因）。我在当前 tip 的 pbxproj 副本上本轮实测覆盖脚本四态：不配 → warning 且 3 处仍为空；配 `ABCDE12345` → 3 处替换、残留 0；配 `ABC` → `::error::`、退出码 1、工程未改（小写 `abcde12345` 同样被拒）。因此若日志显示已 overridden 但产物 Team 不符，方向是**覆盖被后续步骤写回**（`xcode-project use-profiles` 按签名文件重写团队）或路径/时机错，而不是 sed 表达式。
  - 若产物 entitlements 出现 `aps-environment`：**`ios/Runner/Runner.entitlements` 已无该键**（复核：`plistlib` 解析 `keys=[]`、`grep -c 'aps-environment' ios/Runner/Runner.entitlements` = 0），所以 v1 产物里出现 `development`（或任何 `aps-environment` 值）说明**构建的不是触发时 tip** → 按 G6.3 记**无效构建**（不计入 T1–T7），不是 CI 回归。原来那三个根因方向在 v1 口径下不再是首版失败原因，**改为「以后开 Push 后才适用」**：① `xcode-project use-profiles` 拿到的是 **development** 描述文件（未拿到 App Store 分发描述文件）→ 导出配置 `method` 与描述文件类型不符；② App ID `top.headwindx.localmind` **未勾 Push Notifications**（capability 缺失 → 在导出阶段才报错或被降级）；③ 导出配置 `method` 非 `app-store`（若走运行时生成分支，`method` 由脚本固定写 `app-store`，此项只可能是外部/手改 plist 介入）。**取证动作保留**：先看日志 `export options = <路径> (来源：…)` 那行定位实际用的是哪个文件，再 `PlistBuddy -c 'Print :method'` 读该文件确认 `method`；同时用 `codesign -d --entitlements :- ipa/Payload/Runner.app` 读产物实际签入的 entitlements，与构建日志里的 commit SHA 一并留档。
  - 若 profile 与 App ID 不匹配：G4 的 App ID 未建或 bundle id 拼写不一致。

## T3｜IPA 产出并上传成功，`Internal Testers` 收到构建 · 覆盖点③

**用例句**：当构建进入 publishing 阶段时，Codemagic 应产出至少 1 个 `.ipa` 工件且其 `bundleId` 为 `top.headwindx.localmind`，上传后 App Store Connect → TestFlight 应在 15 分钟内出现该 build（状态从 `Processing` 变为 `Ready to Test`），且该 build 的 **内部组** 列表中应出现 `Internal Testers`。

- **前置条件**：G1、G2、G4（G3 建议）；T1、T2 通过。
- **操作步骤**：
  1. Codemagic 构建页 Artifacts 里确认 `.ipa` 存在并下载；`unzip -p ... Info.plist` 或 `PlistBuddy -c 'Print :CFBundleIdentifier'` 读 bundleId。
  2. 用 `$CM_ARTIFACT_LINKS`（日志 posting 段有 JSON）核对 `bundleId` 字段。
  3. App Store Connect → Apps → LocalMind X → TestFlight，按 build 号定位；等待处理完成。
  4. 打开该 build 的 “Group” 列/详情，确认 `Internal Testers` 在列。
- **判定口径**：
  - 通过：工件存在且 bundleId 一致；ASC 出现同 build 号；build 详情里 `Internal Testers` 已挂载；tester 侧 TestFlight App 里能看到该版本。
  - 失败：工件不存在（`flutter build ipa` 未产出或 `--export-options-plist` 路径错）；上传 401/403；ASC 出现 build 但无组（publishing 的 `beta_groups` 未生效）；超过 15 分钟仍未出现在 ASC（Codemagic post-processing 第一步超时）。
- **失败根因方向**：
  - `submit_to_testflight: true` 在 Codemagic 语义上是“提交 beta 审核”，与 `beta_groups` 一起在 **post-processing（构建结束后异步）** 执行，不在构建日志主流程里；因此“日志全绿但组里没构建”最常见原因是 post-processing 失败或超时 —— 需看构建页的 post-processing 状态，日志本身不会报。
  - `beta_groups` 名称与 ASC 里实际组名不一致（大小写、前后空格）会静默不挂组。
  - ASC 侧 `Internal Testers` 里没有任何已接受邀请的成员时，组存在但收不到通知。
  - 基线 yaml 的 `artifacts` 只收 `build/ios/ipa/*.ipa` 与 `/tmp/xcodebuild_logs/*.log`；`.xcarchive` 不在工件里，需要签名材料时从日志里的路径取。

## T4｜build 号自动递增且不与既有编号冲突 · 覆盖点④

**用例句**：当触发 `ios-testflight` 构建时，`Build IPA` 步骤应把**三条 ASC 查询命令全部执行**（不是第一条成功就 break），最终 `CFBundleVersion` 应等于 `max(PROJECT_BUILD_NUMBER, ASC 三命令返回的最大号 + 1, pubspec.yaml build 号 + 1)`；连续两次构建的 `CFBundleVersion` 严格递增，且每次都**严格大于 App Store Connect 上同一 `CFBundleShortVersionString` 下已存在的最大 build 号**（当前 `pubspec.yaml` 为 `1.9.1+96`，故任何一次都不得 ≤ 96）。

- **前置条件**：G1、G2、G4（G3.2/G3.3 决定能否走严格口径）；已发生至少 1 次成功构建。
- **操作步骤**：
  1. 记下第一次构建的 `PROJECT_BUILD_NUMBER`（日志环境信息段）与产物的 `CFBundleVersion`：

     ```bash
     /usr/libexec/PlistBuddy -c 'Print :CFBundleVersion' ipa/Payload/Runner.app/Info.plist
     /usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' ipa/Payload/Runner.app/Info.plist
     ```

  2. 从构建日志取出三条 ASC 查询的返回与最终号：

     ```bash
     grep -nE '^ASC get-latest-(app-store|testflight)?-?build-number = |^CFBundleVersion = ' build.log
     grep -nE '::notice::命令 |::warning::(取 ASC 最新构建号失败|未设置 APP_STORE_APPLE_ID)' build.log
     ```

  3. 再触发一次（再次 Start build 即可，不要求改代码），重复读取。
  4. 在 ASC → TestFlight 列表里读取同版本号下最大 build 号作独立对照。
- **判定口径**：
  - 通过（严格口径，G3.2 + G3.3 均就绪）：日志出现**三行** `ASC get-latest-app-store-build-number = <n>`、`ASC get-latest-build-number = <n>`、`ASC get-latest-testflight-build-number = <n>`（三条都执行过）；`CFBundleVersion = <三命令最大值 + 1>` 且大于 `PROJECT_BUILD_NUMBER` 与 `pubspec` build 号；两次构建严格递增；每次值都严格大于 ASC 同版本已有最大号；`CFBundleShortVersionString` 等于 `pubspec.yaml` 的 `1.9.1`；日志 `CFBundleVersion = <号>` 与产物 `Info.plist` 一致；上传未被 Apple 以 “redundant binary / build number already used” 拒收。
  - **分支：三条中只有一条可用**（另两条打出 `::notice::命令 <cmd> 未返回纯数字构建号（输出：…）`）→ 仍必须取到号（脚本 `FOUND=1` 即继续），判定按该唯一可用号 + 1 计入，其余两条仅记 notice；此时仍算**通过**（号来自 ASC 真实数据），但要记录哪两条不可用（`get-latest-app-store-build-number` 对未上架 App 常返回失败属正常）。
  - **分支：三条全失败或全返回非数字** → 日志三条 notice + `::warning::取 ASC 最新构建号失败（app=<APP_STORE_APPLE_ID>），沿用当前号 <n>`；此时号退回 `max(PROJECT_BUILD_NUMBER, pubspec+1)`，ASC 侧比较需人工在 ASC → TestFlight 列表核对一次，结论记**部分验证**，不得记通过。
  - 降级口径（G3.2 未配 `APP_STORE_APPLE_ID`）：日志只有 `::warning::未设置 APP_STORE_APPLE_ID…`，无任何 `ASC … = ` 行；只能判“两次严格递增且均 > 97”，结论记**部分验证**。
  - **分支：`PROJECT_BUILD_NUMBER` 缺失/为空**（构建号兜底已引入，沿革见文首；本轮已在当前 tip 的 `Build IPA` 脚本上重跑 mock 验证）：脚本以 `${PROJECT_BUILD_NUMBER:-0}` 起算，仍应凭 `pubspec` build 号得出 97，**不得出现空的 `CFBundleVersion = `**；若 `pubspec` build 号也不可用（version 行被改坏）则打 `::error::构建号无法确定（PROJECT_BUILD_NUMBER 未设置，且 pubspec version 不可用）` 并退出 1 —— 这是期望行为，记 G3/仓库配置问题，不算 CI 回归。
  - 失败：两次相同或第二次更小；任一次 ≤ ASC 同版本最大号（严格口径下）；`CFBundleVersion` 读成 `96`（`--build-number` 未生效）；`CFBundleVersion` 为空或非整数；Apple 拒收重复号。
- **失败根因方向**（策略与理由见 `docs/codemagic-testflight.md` **§3.6「构建号策略：结论与理由」**）：
  - 日志只有**一行** `ASC … = ` 就开始构建 → yaml 还是旧版“第一个命令成功即 break”的写法，没切到基线；基线要求三条全查取最大。
  - 出现 `::warning::取 ASC 最新构建号失败` → 三条**全部**失败，方向是 G3.2/G3.3（`APP_STORE_APPLE_ID` 没配、`app_store` 组缺、或组里凭据与 App 记录不匹配），**不是**命令名写错；单条失败只产生 `::notice::`。
  - 出现 `::notice::命令 … 未返回纯数字构建号` 但号仍正常 → 预期降级，记录哪条不可用即可，不要当失败。
  - `PROJECT_BUILD_NUMBER` 是**项目级累计计数**，不是“上次成功值 +1”：删构建、换 workflow、重建 Codemagic 应用都会让它与 ASC 已有编号错位 —— 这就是基线要并入 ASC 最大号与 `pubspec` build 号的原因；若仍撞号，先确认三条 ASC 是否真的都跑过。
  - `CFBundleVersion` 读成 `$(FLUTTER_BUILD_NUMBER)` 字面量：`ios/Runner/Info.plist` 的变量替换未生效（缺 `flutter build` 前的 `flutter pub get`，导致 `Generated.xcconfig` 缺失）。

## T5｜真机可安装并启动 · 覆盖点⑤

**用例句**：当用已加入 `Internal Testers` 的 Apple ID 在 iPhone 的 TestFlight App 中安装该 build 时，安装应完成、图标名称为 `LocalMind X`、首次启动应进入应用主界面且 60 秒内不闪退，同时安装不得因 Push Notifications capability 缺失而失败。

- **前置条件**：G5；T3、T4 通过；iPhone iOS 版本 ≥ `IPHONEOS_DEPLOYMENT_TARGET`（17.6）。
- **操作步骤**：
  1. TestFlight → LocalMind X → Install；记录从点击到 `OPEN` 出现的耗时。
  2. 点击打开，观察启动到主界面的时间；停留 60 秒；切后台再切回一次。
  3. 记录 `Settings → General → iPhone Storage → LocalMind X` 里的版本号（应与 T4 的 build 号一致）。
  4. 若安装报 “This app cannot be installed”，截屏并记录弹窗原话。
- **判定口径**：
  - 通过：安装成功、启动进入主界面、无闪退、存储里版本号与 T4 一致；**通知授权弹窗出现或至少不阻塞启动**（允许用户拒绝，拒绝后仍可正常使用）。
  - 失败：安装阶段报 profile/UDID/最低系统版本错误；启动即退（crash）；推送权限请求导致启动卡死。
- **失败根因方向**：
  - 装不上：build 为 “TestFlight Internal Only” 以外的审核状态阻塞；或设备 iOS 低于 17.6；或 bundle id 与登录的 ASC 记录不一致（会表现为 TestFlight 里根本没有该 App）。
  - 闪退：本仓库 iOS 侧有新引入的 native 依赖（`OsBackgroundDownloadManager.swift`、本地推理后端），首次启动崩溃多来自动态库未嵌入或模型文件缺失 —— 需拉 `Console.app`/Xcode Devices 的 crash log 定位，不要先怀疑签名。
  - 收不到推送但能装：属预期（APNs Key 与服务端未配，且 **v1 首版不带 Push**：`ios/Runner/Runner.entitlements` 已无 `aps-environment`；`docs/codemagic-testflight.md` 的 **§1.2 已写明「首次构建不勾 Push」**（定位：`grep -n '首次构建不勾' docs/codemagic-testflight.md` 命中 1 行，位于 §1.2 表内）、**§3.5 写明首版不带 Push 与开启顺序**（定位：`grep -n '首版不带 Push（entitlement 口径）' <同文件>` 与 `grep -n '这三步只是必要条件' <同文件>` 各命中 1 行，均落在 `### 3.5` 与 `### 3.6` 之间），均不涉推送下发；该文档 §3.4 仍是「Team ID 与工程里的占位符」、不涉 APNs），不作为本用例失败判据。

## T6｜CI 实际使用的 Flutter 版本与 `.fvmrc` 一致 · 覆盖点⑥

**用例句**：当构建在 `Get Flutter packages` 之前完成环境准备时，日志中 `flutter --version` 报告的版本号应等于仓库 `.fvmrc` 中的 `flutter` 值（`.fvmrc` 当前为 `3.44.2`），且该版本应同时满足 `pubspec.yaml` 的 `environment.flutter: ">=3.44.0"`。

- **前置条件**：G1；可读取构建日志（不必 T1 通过）。
- **操作步骤**：
  1. 从日志 “Preparing build machine / Environment” 段找出 Flutter 版本行（`Flutter 3.x.y • channel ...`）。
  2. `git show <CM_COMMIT>:.fvmrc` 读取该 commit 的 `.fvmrc`。
  3. 对照 `codemagic.yaml` 的 `environment.flutter`。
- **判定口径**：
  - 通过：日志版本 == `.fvmrc` 版本 == yaml 声明版本（当前三处均应为 `3.44.2`）。
  - 失败：三者任一不一致，或日志版本高于/低于 yaml 声明。
  - **已在 `ci/codemagic-gitea-source` 修复**（复核：`grep -c 'flutter: 3.44.2' codemagic.yaml` = 1，且 `cat .fvmrc` = `{"flutter": "3.44.2"}`）：`environment.flutter` 已从 `3.44.0` 改为 `3.44.2`（对齐 `.fvmrc`，Dart 3.12.2）。基线 tip 上本用例预期通过；`main`（`bb564e6`）上预期失败。首次真实构建日志待验。
- **失败根因方向**：
  - 三处版本来源（yaml / .fvmrc / pubspec 下限）没有单一事实源；本地用 fvm、CI 用 yaml，长期必然漂移。
  - 若日志版本比 yaml 高（例如解析成 channel 最新）：yaml 的值被 `flutter: stable` 之类覆盖，或该精确版本在 Codemagic 上不可用而被回退（`docs/codemagic-testflight.md` §5 排错表**首行**「构建报 `Flutter version 3.44.2 not available`」即此现象；定位：`grep -n 'Flutter version 3.44.2 not available' <同文件>` 命中 1 行）。

## T7｜Flutter 版本不一致时必须失败并给出可定位信息 · 覆盖点⑦

**用例句**：当把 `.fvmrc` 的版本改成 Codemagic 上不存在的版本（或把 yaml 改成 `flutter: fvm` 而 `.fvmrc` 缺失）并触发构建时，构建应在“环境准备”阶段以 failed 结束，日志中应出现包含该版本号的错误文本，且**不得**静默换用其他 Flutter 版本继续到 `Build IPA`。

- **前置条件**：G1；操作在**一次性分支**（如 `tmp/qa-fvm-negative`）上进行，绝不推 `main`（main 已冻结，且 CI 配置只在 `docs/gitea-repo-live` 上生效）
- **操作步骤**：
  1. 在临时分支把 `.fvmrc` 改成 `{"flutter": "3.44.99"}`（不存在的版本），推送该分支并在 Codemagic 手动选该分支构建（yaml 当前不用 fvm 时，改 yaml 的 `flutter: 3.44.99` 亦可，但仅改临时分支）。
  2. 记录构建状态与失败步骤名。
  3. 重建后删除临时分支。
- **判定口径**：
  - 通过：状态 failed；失败发生在环境准备/flutter 安装步骤；日志含 `3.44.99` 字样与 “not found / unable to install / no such version” 一类明确措辞；**没有** 后续 `Build IPA` 步骤的成功行。
  - 失败：构建继续并成功（说明版本不可用时被静默替换）；或失败但日志只有泛化 `script exited with code 1`，无版本信息（可定位性不足 —— 这本身就是缺陷）。
- **失败根因方向**：
  - `environment.flutter` 可被 `flutter: stable/latest` 类值“兜底”，导致无法定位。
  - 反向风险：若改成 `flutter: fvm` 而 `.fvmrc` 不存在，官方文档明确“构建会失败”，这属于**期望行为**，不要误判为回归。

---

## 附加检查（不阻塞首次发布，但必须记录）

| 编号 | 检查 | 可测句 | 现状（静态核对；命令输出为写作时 tip 实测） |
| --- | --- | --- | --- |
| R1 | 导出配置的四级确定性选取 | 当 `Build IPA` 步骤执行时，日志应打印一行 `export options = <路径>`，该路径为下列之一：(a) `$HOME/export_options.plist`（构建机上即 `/Users/builder/export_options.plist`，来源标注 `来源：xcode-project use-profiles 生成`）；(b) `$CM_BUILD_DIR/export_options.plist`；(c) 运行时生成的 `$CM_BUILD_DIR/export_options.generated.plist`（来源标注 `运行时生成：teamID=…, provisioningProfile=…`）；(d) 当日志出现 `export options = (none)` 时应紧跟 `::error::use-profiles 没有生成导出配置…` 且 rc≠0。**当该行路径指向仓库内的 `ios/ExportOptions.plist`，或出现不带来源标注的裸行 `export options = <路径>` 时** → 说明构建的不是触发时 tip 那份 yaml（四级选取是现行 yaml 的实现，引入点见沿革链）→ 记**无效构建**，不计入 T1–T7。yaml 的 **5 个 script 块内**不得出现 `$FCI_BUILD_DIR`（全文件 `grep -c 'FCI_BUILD_DIR' codemagic.yaml` = 1，且该唯一一行以 `#` 开头、描述的是 main 上旧 yaml；`grep -cE '^[^#]*FCI_BUILD_DIR' codemagic.yaml` = 0（即非注释行 0 处））。**当 `use-profiles` 未生成配置、且 `XCODE_TEAM_ID` 未设或非 10 位大写字母数字时** → 日志应为 `export options = (none)` + `::error::use-profiles 没有生成导出配置…` + rc≠0，**这是期望的确定性失败，不算静默兜底、不算 CI 回归** | **已按当前 tip 的实现重跑核对**：该 yaml 在注释里明确写死**不从仓库内的 `ios/ExportOptions.plist` 取配置**。静态核对通过（`grep -cE '^[^#]*FCI_BUILD_DIR' codemagic.yaml` = 0；`grep -c 'TZX5LG7ZW5' codemagic.yaml` = 0；`grep -n 'ios/ExportOptions.plist' codemagic.yaml` 命中 1 行且在注释里）。我在当前 tip 的 `Build IPA` 脚本上桩测五态（本轮重跑）：(a) 有 `$HOME/export_options.plist` → rc=0，`flutter build ipa --export-options-plist=<该文件>`；(b) 仅有 `$CM_BUILD_DIR/export_options.plist` → rc=0 且用该文件；(c) 两者皆无 + `XCODE_TEAM_ID=ABCDE12345` + pbxproj 有 `PROVISIONING_PROFILE_SPECIFIER` → rc=0，生成 `export_options.generated.plist` 含 `method`/`teamID`/`signingStyle`/`provisioningProfiles`/`uploadSymbols`/`uploadBitcode`/`stripSwiftSymbols`，`grep -c destination` = 0；(d) 两者皆无 + `XCODE_TEAM_ID` 未设 → `export options = (none)` + `::error::use-profiles 没有生成导出配置…` + rc=1、flutter 未被调用；(e) 两者皆无 + Team ID 合法但已装描述文件里无匹配 bundle id → 同样 `(none)` + rc=1（不猜描述文件名）。首次构建日志待验 |
| R2 | 自动签名是否被显式建立 | 日志中应依次出现 `Override Xcode development team`、`Set up code signing settings on Xcode project`（`xcode-project use-profiles`）、`Install pods`，且实际导出用的配置含 `provisioningProfiles`（`use-profiles` 生成的 plist 或运行时生成的那份） | **已在 `ci/codemagic-gitea-source` 修复**（沿革见文首，现行 yaml 沿用）：`environment.ios_signing{distribution_type: app_store, bundle_identifier: top.headwindx.localmind}`、`xcode-project use-profiles`、`pod install` 三步均在触发时 tip 的 yaml 中（本轮实测 `scripts` 顺序逐字一致：Override → use-profiles → Get Flutter packages → Install pods → Build IPA）。**更正上一版说法**：仓库内 `ios/ExportOptions.plist` 不再作兜底，CI 已不读它（yaml 注释明确写死；定位：`grep -n 'ios/ExportOptions.plist' codemagic.yaml` 命中 1 行、以 `#` 开头且含「不退回」）。静态核对通过，首次构建日志待验 |
| R3 | `submit_to_testflight` 语义 | 当只面向内部组时，是否必须提交 beta 审核 | 该字段在 Codemagic 语义为“提交 TestFlight beta 审核”，内部组本身不需要审核；若因此延长可见时间或产生审核状态，记录为体验问题而非阻塞 |
| R4 | `max_build_duration` 余量 | 当首次构建（无缓存）耗时应 ≤ 75 分钟，留 15 分钟余量给 90 分钟上限 | 首次 pod install + 全量编译无缓存，需实测 |
| R5 | 构建源与触发链是否真的由 Gitea 触发 | 当向触发分支 **`docs/gitea-repo-live`** 推一次 commit 时，应由 Gitea webhook 触发构建，且 `CM_COMMIT` 等于该次 push 的 commit（= 触发时该分支的 SHA）；GitHub 侧**不参与 CI 构建**（同步策略待定，见附录 A）；**`main` 已冻结在 `bb564e6`，向 main 推送不得作为触发测试对象** | **已实证（2026-11-24）**：Deploy Key id=1（只读）+ push webhook id=4（events=push、active）已配；向 `docs/gitea-repo-live` 推 `8ecf9c2` → Gitea 投递 → Codemagic 收到 `X-Gitea-Event: push`（兼容头 `X-GitHub-Event: push`）、`ref=refs/heads/docs/gitea-repo-live`、`after=8ecf9c21` → 响应 **202「Started builds: ios-testflight」** → 生成构建 **#3**；Gitea Webhooks → Recent deliveries 存有该次请求头与 payload；GitHub 侧 0 hook（不会重复触发）。构建 #3 在**配置校验阶段**即停（0 机器分钟），原因是缺 `top.headwindx.localmind` 的 App Store 描述文件 → **触发链本身已通**，失败点在 Apple 侧签名资产（G2 / G4），不是触发问题 |
| R6 | 构建号兜底 | `PROJECT_BUILD_NUMBER` 为空时不得产出空号 | **构建号兜底已修复**（沿革见文首，现行 yaml 沿用）：`BUILD_NUMBER="${PROJECT_BUILD_NUMBER:-0}"`，且号 < 1 时 `::error::构建号无法确定…` 退出 1。当前 tip 上本轮重测（mock `app-store-connect`，7 例）：PROJECT 为空 + pubspec 96 + ASC 三命令全返回 300 → `CFBundleVersion = 301`；PROJECT 为空 + pubspec 被改坏 → `::error::构建号无法确定…` rc=1、无空号；PROJECT 为空 + 三命令全失败 → notice×3 + warning + 97 |
| R7 | 导出配置不得含 `destination=upload` | 当 `Build IPA` 步骤输出 `export options = <路径>` 后，**该导出配置内不得出现 `destination` 键为 `upload`**。取证：`/usr/libexec/PlistBuddy -c 'Print :destination' <该路径>` 应报键不存在；或 `grep -c 'destination' <该路径>` = 0。**出现 `destination=upload` → T3 判失败**，根因指向该导出配置由旧路径或手改产生（运行时生成的配置实测不含 `destination`；`docs/codemagic-testflight.md` 的 §3.5 与 §5 排错表同一口径：先上传一次、Codemagic 再传同一 build 号会被 Apple 拒收；定位：`grep -n 'destination=upload' docs/codemagic-testflight.md` 命中 2 行 —— §3.5 正文 1 行、§5 排错表 1 行）。`dest=upload` 的仓库旧文件 `ios/ExportOptions.plist` 仍在仓库（三键：method/signingStyle/destination），CI 不读它，故不构成风险 | **已核（本轮在当前 tip 重跑）**：运行时生成的 plist `grep -c destination` = 0；yaml 内 `destination` 只出现在注释里（`grep -n 'destination' codemagic.yaml` 命中 2 行、均以 `#` 开头，说明为何不用旧文件）。首次构建日志待验 |

---

## 执行顺序与回收格式

1. G1、G2、G4 就绪后（G3 建议就绪）→ T6（最快，只需日志）→ T1 → T2 → T3 → T4 → T5；T7 在 T1 之后任意时点用临时分支做；R5（自动触发）已实证通过，无需再单独验。
2. 每次构建回收这 5 个值，写进回收表：`CM_BUILD_ID` / `CM_COMMIT` / `PROJECT_BUILD_NUMBER` / `CFBundleVersion` / `CFBundleShortVersionString`。
3. 任一用例失败：给出“失败步骤名 + 日志原文 3–10 行 + 归属（Indigo 配置 / Henry 控制台 / Apple 侧 / Gitea 侧）”，不要只写“构建失败”。

## 当前判定

| 用例 | 覆盖点 | 现在能执行吗 | 阻塞原因 |
| --- | --- | --- | --- |
| T1 | ① 日志全绿、签名无错 | 否 | G1 已就绪（应用已接 Gitea）；仍缺 G2、G4 —— Apple 侧没有 `top.headwindx.localmind` 的 App Store 描述文件（构建 #3 即因此在配置校验阶段停止）；自动触发已实证（R5），不再阻塞 |
| T2 | ② Team ID 覆盖生效、产物无 TZX5LG7ZW5 | 否 | 依赖 T1 产物 |
| T3 | ③ IPA 上传、内部组收到 | 否 | G1、G2、G4；依赖 T1/T2 |
| T4 | ④ build 号递增不冲突 | 否 | G1、G2、G4；需 2 次构建；G3.2/G3.3 缺失时只能部分验证 |
| T5 | ⑤ 真机安装并启动 | 否 | G5 + T3/T4 |
| T6 | ⑥ Flutter 版本与 .fvmrc 一致 | 否 | 需 G1 才能拿到构建日志（版本不一致**已在基线修复**为 3.44.2，待日志确认） |
| T7 | ⑦ 版本不一致时失败且可定位 | 否 | G1；需临时分支与一次 macOS 构建分钟 |
| R5 | 触发源是否由 Gitea webhook 触发（推 `docs/gitea-repo-live` 能否自动触发） | **是** | 已实证（2026-11-24）：Deploy Key id=1（只读）+ push webhook id=4（Active）已配；推 `8ecf9c2` → 202「Started builds: ios-testflight」→ 构建 #3；`main` 已冻结、不作触发测试对象 |

结论：计划已就绪，7 项覆盖点全部落成可执行用例；**所有用例均需 Henry 完成 G1、G2、G4（G3 建议）后才能执行**，其中 T2–T5 还需要 Indigo 跑通首次构建、T5 还需要一台真机与内部组账号；**自动触发（R5）已实证通过**（G6.1 / G6.2 已就绪），当前阻塞集中在 Apple 侧签名资产（G2 / G4）。

原先静态核对出的 3 个风险（R1 `FCI_BUILD_DIR`、R2 缺 `use-profiles`/导出选项、T6 版本不一致）**已在 `ci/codemagic-gitea-source` 修复**（沿革见文首，现行 yaml 沿用）；此后导出配置又收紧一层：由「只用 use-profiles 产物、拿不到就以仓库内那份 plist 兜底」改为**四级确定性选取**，且明确写死不读仓库 `ios/ExportOptions.plist`（逐版沿革见文首）。**本文不再用「某一版相对前一版改了什么」来支撑结论**（那套论述会随 tip 漂，而判据从来不依赖它）：证据一律改成在当前 tip 上重跑命令 —— 我在当前 tip 上**把 yaml 层静态项与全部桩测重跑了一遍**（`yaml.safe_load` 解析、行数、三处 grep、5 段 `bash -n`；导出选项五态 + Team 覆盖四态 + 构建号九例，桩测 18 项全过）：`yaml.safe_load` 解析通过、`environment.xcode` 为 `'26.6'`（str）；`triggering` / `scripts`（5 段）/ `integrations` / `environment.flutter` / `cocoapods` / `groups` / `ios_signing` / `artifacts` / `publishing` 的逐字段核对见附录 A；5 个 `script` 块 `bash -n` 全过（rc=0）；Team 覆盖脚本在当前 tip 的 pbxproj 副本（3 处空占位、`TZX5LG7ZW5` 残留 0）上四态实测；构建号逻辑用 mock `app-store-connect` 跑 9 例；导出选项四级选取跑 5 路桩测（含运行时生成、Team ID 未设 → rc≠0、无匹配描述文件 → rc≠0）。**entitlement 侧实测**：当前 tip 的 `Runner.entitlements` 经 `plistlib` 解析 `keys=[]`、`aps-environment` 计数 0；开启 Push 的三步顺序见 T1 失败根因「签名类」。**唯一尚未验证的环节是首次真实构建日志**（G6.1 / G6.2 已就绪，等 Henry 补齐 G2 / G4 的 Apple 侧签名资产）与一台真机。

## 附录 A｜`codemagic.yaml` 逐字段核对（触发时 `docs/gitea-repo-live` tip；2026-11-23 本地执行）

逐字段对照来源：直接读触发时 tip 的 yaml 原文 —— `git show docs/gitea-repo-live:codemagic.yaml`（2026-11-23 本地执行）。**本附录「实际值」列是内容事实**：不随分支 tip 漂，只有 yaml 内容真的改了才会变；复核方式是照抄各行的复核命令，输出须与「实测」一致，不一致即被测物已变、本附录需重核（各版本改了什么见文首沿革链，这里不再逐版论述）。当前 yaml 规模：`git show docs/gitea-repo-live:codemagic.yaml | wc -l` = 272 行、`git cat-file -s $(git rev-parse docs/gitea-repo-live:codemagic.yaml)` = 15908 字节、`yaml.safe_load` 解析通过。与判定直接相关的字段固化如下（值全部取自 tip 上的 yaml 原文，非转述）：

| 字段 | 触发时 tip 的实际值（复核命令见格内） |
| --- | --- |
| **构建源** | **Gitea 源码仓库** `one/localmind-x`（`git@git.headwindx.top:one/localmind-x.git`，HTTPS `https://code.headwindx.top/one/localmind-x.git`；Add application → HTTPS/SSH + Deploy Key **只读** + 手配 webhook，Target URL 从 **Codemagic 应用 → Webhooks 复制 payload URL**，不手拼）。GitHub `dehengxu/localmind-x` **不参与 CI 构建**（yaml 注释里的 GitHub 角色：当前值是「角色待定：镜像 / 归档 / 删除，三选一」，旧措辞原文为「降级为只读备份」—— 此处只作原文引述、不作判据，措辞沿革见文首。注意这只是**文案**变化，不影响任何判定）。**仓库侧现状：已创建（私有，默认分支 main）并全量推送**（2026-11-23 实测 `ls-remote`：`main`=`bb564e6`、`docs/gitea-repo-live` = 触发时 tip、30 tag；分支数以 `git ls-remote --heads` 实时为准，实测 11）；**Codemagic 控制台已重接该仓库、Deploy Key 已加（只读 id=1）、push webhook 已配（id=4，Active）**（2026-11-24 实测）。该段注释改过两处措辞（「构建源：自建 Gitea（正本）」→「（源码仓库）」、GitHub 那行改「不作为构建源（角色待定）」），webhook 口径本身未变 |
| `workflows.ios-testflight.name` | `iOS TestFlight` |
| `instance_type` / `max_build_duration` | `mac_mini_m2` / `90` |
| `triggering` | `events: [push]`；`branch_patterns: [{pattern: docs/gitea-repo-live, include: true, source: true}]` —— **唯一条目即 `docs/gitea-repo-live`，不再是 `main`**（`yaml.safe_load` 实测：`{"events": ["push"], "branch_patterns": [{"pattern": "docs/gitea-repo-live", "include": true, "source": true}]}`）；注释明确“触发源 = Gitea 的 push webhook（§2.3），Gitea 侧未配 webhook 时控制台 Start build”。**Codemagic 官方口径：webhook 触发时用的是被推分支上那份 `codemagic.yaml`**，即触发时 tip 这一份（判据：`grep -c 'pattern:' codemagic.yaml` = 1 且 `grep -n 'pattern: docs/gitea-repo-live' codemagic.yaml` 命中 1 行 —— 唯一条目即该分支） |
| **触发分支** | **`docs/gitea-repo-live`**（main 冻结在 `bb564e6`，不作触发源）。`main` 两端恒为 `bb564e6`、其 yaml 是旧版（`LocalMind ASC API Key` / `$FCI_BUILD_DIR` / flutter 3.44.0、无 `use-profiles` / `ios_signing`）→ **按 main 建的构建一律记无效构建，不计入 T1–T7**。将来 Henry 指定长期主干分支时，只需改 `branch_patterns` 一行 |
| `integrations.app_store_connect` | `App Manager`（注释含三条路径与“两边同名是硬要求”） |
| `environment.flutter` / `xcode` / `cocoapods` | `3.44.2` / **`"26.6"`（字符串）** / `default`。**`xcode` 在 yaml 原文里写成带引号形式 `xcode: "26.6"`** —— 不带引号时 YAML 会把 `26.6` 解析成浮点数，而该字段语义是版本号字符串，故必须显式加引号（解析值仍是 26.6、只有类型 float→str；这一层的沿革见文首）。**`xcode` 钉 26.6 的理由**：Codemagic 官方《Xcode update policy》把 `latest` 定义为「最新 *release* 版本」——发新 major 时不立刻切、但**会漂**（`edge` 才是最新可用）；现官方镜像页上 `Xcode 26.6.x (default)` = macOS 26.5.1 / Xcode 26.6 / iOS 26.5 runtime（Mac mini M2 与 M4 都有），另有 `Xcode 27.2.x` 镜像。仓库根 `.fvmrc` 锁 `flutter 3.44.2`，该 SDK 源码 `xcodeRequiredVersion = 15` / `xcodeRecommendedVersion = 16` → 未针对 Xcode 27 验证，故取默认档 26.6 而不用会漂到 27.2 的 `latest`。实测（当前 tip）：`yaml.safe_load` 解析后 `xcode` 为字符串 `'26.6'`，`grep -c 'xcode: latest' codemagic.yaml` = 0、`grep -c 'xcode: "26.6"' codemagic.yaml` = 1、`grep -c 'xcode: 26.6' codemagic.yaml` = 0（引号是 yaml 原文的一部分，不带引号的 grep 在现行内容上会漏判） |
| `environment.ios_signing` | `distribution_type: app_store`，`bundle_identifier: top.headwindx.localmind` |
| `environment.groups` | `[app_store]` |
| `scripts`（顺序） | `Override Xcode development team` → `Set up code signing settings on Xcode project` → `Get Flutter packages` → `Install pods` → `Build IPA` |
| `Build IPA` 构建号起手 | `BUILD_NUMBER="${PROJECT_BUILD_NUMBER:-0}"`（当前 tip 上本轮实测 9 例）；pubspec build 号更大则取 `pubspec+1`；号 < 1 → `::error::构建号无法确定（PROJECT_BUILD_NUMBER 未设置，且 pubspec version 不可用）` 退出 1 |
| `Build IPA` ASC 查询 | 三条**全部执行**：`get-latest-app-store-build-number`、`get-latest-build-number`、`get-latest-testflight-build-number`；每条成功打 `ASC <cmd> = <n>`；非纯数字/失败打 `::notice::命令 <cmd> 未返回纯数字构建号（输出：…）`；三条全失败才打 `::warning::取 ASC 最新构建号失败（app=<id>），沿用当前号 <n>`；最终 `CFBundleVersion = max(PROJECT, ASC 最大号+1, pubspec+1)` |
| `Build IPA` 导出选项 | **四级确定性选取，不读仓库 `ios/ExportOptions.plist`**：① `$HOME/export_options.plist`（`use-profiles` 生成，实测日志 `export options = <路径> (来源：xcode-project use-profiles 生成)`）→ ② `$CM_BUILD_DIR/export_options.plist` → ③ 运行时生成 `$CM_BUILD_DIR/export_options.generated.plist`（内容实测：`method=app-store`、`teamID=$XCODE_TEAM_ID`、`signingStyle=manual`、`provisioningProfiles→top.headwindx.localmind`、`uploadSymbols=true`、`uploadBitcode=false`、`stripSwiftSymbols=true`，**无 `destination`**；日志 `… (运行时生成：teamID=…, provisioningProfile=…)`）→ ④ 均无则 `export options = (none)` + `::error::use-profiles 没有生成导出配置…` + rc≠0。候选 ① ② 只接受含 `<key>method</key>` 的文件 |
| `artifacts` | `build/ios/ipa/*.ipa`、`/tmp/xcodebuild_logs/*.log` |
| `publishing.app_store_connect` | `auth: integration`、`submit_to_testflight: true`、`beta_groups: [Internal Testers]` |

仓库侧对照（当前 tip）：`cat .fvmrc` = `{"flutter": "3.44.2"}`、`grep -c '^version: 1.9.1+96' pubspec.yaml` = 1（`environment.sdk ">=3.12.0 <4.0.0"`、`environment.flutter ">=3.44.0"`）、`ios/Podfile` 存在（`platform :ios, '17.6'`）、**`ios/Runner/Runner.entitlements` 已无 `aps-environment`**（`plistlib` 实测 `keys=[]`、`grep -c 'aps-environment' ios/Runner/Runner.entitlements` = 0）、`grep -c 'PRODUCT_BUNDLE_IDENTIFIER = top.headwindx.localmind' <pbxproj>` = 6、`grep -c 'DEVELOPMENT_TEAM = "";' <pbxproj>` = 3、`grep -c 'TZX5LG7ZW5' <pbxproj>` = 0。**`ios/ExportOptions.plist` 仍在仓库**（实测三键：`method=app-store`、`signingStyle=automatic`、`destination=upload`）—— **CI 已不读它**（该文件是否删/改待 Henry 决定，不影响 CI 判定）。

我独立执行的核对（不依赖 Indigo 的自测结论）：

- `yaml.safe_load` 解析通过；5 个 `script` 块 `bash -n` 全过（rc=0）。
- Team 覆盖脚本在当前 tip 的 pbxproj 副本上四态：不配 → warning 且 3 处仍空（`TZX5LG7ZW5` 残留 0）；配 `ABCDE12345` → 3 处替换、残留空占位 0；配 `ABC` → `::error::`、退出 1、工程未改；配小写 `abcde12345` → 同样 `::error::`、退出 1（正则只认 `^[A-Z0-9]{10}$`，Apple Team ID 为大写）。
- 构建号逻辑用 mock `app-store-connect` 在当前 tip 的 `Build IPA` 脚本上跑 **9 例**，逐例比对实际输出（下表为本轮重跑，非沿用旧结论）：

| 场景 | 预期 | 实测（当前 tip） |
| --- | --- | --- |
| PROJECT 未设，pubspec 96，ASC 三命令全返回 300（c1） | 301 | `CFBundleVersion = 301` ✅ |
| PROJECT=50，pubspec 96，ASC 300（c2） | 301 | `CFBundleVersion = 301` ✅ |
| PROJECT 未设，pubspec 400，ASC 300（c3） | 401 | `CFBundleVersion = 401` ✅ |
| PROJECT 未设，pubspec 96，ASC 三命令**全失败**（c4） | notice×3 + `::warning::取 ASC 最新构建号失败`，97 | `CFBundleVersion = 97` ✅（三条 notice + warning 原文一致） |
| PROJECT 未设，pubspec 96，ASC 三命令全返回非数字（c5） | 同上，97 | `CFBundleVersion = 97` ✅ |
| PROJECT 未设，pubspec version 被改坏（c6） | `::error::构建号无法确定…`、rc=1、无空号 | 实测 `::error::构建号无法确定（PROJECT_BUILD_NUMBER 未设置，且 pubspec version 不可用）`，rc=1 ✅ |
| PROJECT=10，pubspec 5，ASC 300（c7） | 301 | `CFBundleVersion = 301` ✅ |
| 未配 `APP_STORE_APPLE_ID`（c8） | `::warning::未设置 APP_STORE_APPLE_ID…`，97 | `CFBundleVersion = 97` ✅（无任何 `ASC … = ` 行） |
| 三条中只有 `get-latest-build-number` 可用并返回 410（c9） | notice×2 + 取最大值 410 → 411 | `CFBundleVersion = 411` ✅（两条 notice 原文一致） |
