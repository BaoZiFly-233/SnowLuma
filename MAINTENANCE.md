# SnowLuma 维护手册（接手版）

> 面向对象：接手 `SnowLuma` 的维护者。
> 本文件由接手时对 `f2ea3354 chore(release): v2.4.2` 的全量勘察写成，包含**工作原理**、**维护清单**、**风险登记**、**运行手册**与**路线图**。
> 所有 `路径:行号` 引用均针对本次勘察的提交，改动后请同步更新引用。
>
> **脱敏说明**：本文不含任何环境私有信息——账号与仓库归属、网络地址与可达性结论、
> 密钥路径、工具链版本、绝对路径、口令、机器指纹，一律不写（需要时用 `<占位符>` 指代）。
> 真实取值请写在**不随仓库分发**的本地笔记里（例如工作区之外的 `private/env.local.md`），
> 不要写回本文件。

---

## 0. 一页速览

### 这是什么

SnowLuma 是一个"协议运行时"：它把 Windows/Linux 上的 **QQ NT 客户端**变成一台可编程的机器人宿主。

```
QQ.exe (腾讯官方客户端，已登录)
   │  ① 原生 DLL 被手动映射注入（manual map，不落盘、不改文件）
   ▼
nnphook (C++，闭源 .dll/.so) ── 拦截 QQ 的 IPC 收发 ──┐
   │  ② 每进程两条命名管道/Unix socket (mojo.<pid>.control / .recv)
   ▼                                                  │
packages/bridge  ── 管道帧协议(v1) → QQ 原始包          │
   ▼
packages/protocol ── 解包/解码/身份解析/协议请求 ───────┘
   ▼
packages/onebot ── 事件转换(23 类) + 230 个可执行 action + 消息持久化
   ▼
WebSocket / HTTP / HTTP-POST / SSE  ←→  OneBot v11 客户端（NoneBot、Koishi…）
   ▲
   └── WebUI (Hono + React, :5099) / TS SDK / MCP server
```

一句话概括分工：**bridge 负责"连上 QQ"，protocol 负责"看懂 QQ"，onebot 负责"说 OneBot 的话"，core 负责"把一切装起来并管生命周期"，webui/mcp/sdk 是三种使用面。**

### 体检结论（Windows 工作站 / Node `<LOCAL_NODE>` / pnpm `<LOCAL_PNPM>`）

| 项目 | 命令 | 结果 |
| --- | --- | --- |
| 依赖安装 | `pnpm install --frozen-lockfile` | ✅ 通过（23s；需绕过 engine-strict，见 §6-P0-1） |
| 类型检查 | `pnpm typecheck` | ✅ 14/14 包通过 |
| 单元测试 | `pnpm test` | ⚠️ 8/9 包通过；core 1500 过 / 4 失败，**全部是 Windows 环境差异**，非回归 |
| 构建 | `pnpm run build:all` | ✅ 通过，`dist/` 25 MB |
| 产物校验 | `node tools/check-release-layout.mjs` | ✅ `Release layout verified` |
| 冒烟启动 | `node dist/index.mjs` | ✅ 启动 → 原生完整性 `process=High (RID=<RID>)` → 打印初始口令 → `listening http://127.0.0.1:5099` → `curl /` = 200 |

4 个失败测试（属环境，非代码）：

1. `packages/core/tests/webui-restore.test.ts:84` —— 断言 `0o600`，Windows 返回 `0o666`。
2. `packages/core/tests/webui-restore.test.ts:259` —— 同上，`0o700` 目录权限。
3. `packages/core/tests/update-entry.test.ts:159` —— 期望长路径，实得 8.3 短名（`<TEMP_SHORT>~1`）。
4. `packages/core/tests/highway/highway-parallel.test.ts:2092` —— 5000ms 超时（并发上传测试对机器性能敏感；性能不足的机器会超时，别据此调高超时值）。

### 接手后最该先做的三件事

1. **把更新链路指向自己的 release 仓库**（否则"检查更新"会 404，且可能把用户导向已消失的官方 org）。见 §6-P0-2。
2. **确认许可边界**：Source-Available 非商用许可 + 禁止公开分发修改版 + 原生件是独立专有组件。见 §6-P0-3。
3. **建立自己的原生件供应链策略**：仓库里没有原生源码，QQ 一更新就可能全体失效。见 §6-P0-4。

### 风险 Top 5（细节见 §7）

| # | 风险 | 位置 |
| --- | --- | --- |
| 1 | 管道帧协议在 TS 与 C++ 两侧手工重复定义，无版本协商，一处改动即全部会话静默失效 | `packages/bridge/src/qq-hook-client.ts:17-44` |
| 2 | `login.db` SQLCipher 口令与页格式硬编码，QQ 一换即"账号识别全灭" | `packages/bridge/src/qq-account/login-db.ts:9`、`sqlcipher.ts:24-54` |
| 3 | 自更新只校验同源 sha256，无签名，且镜像可由用户配置 | `packages/core/src/update/{release,download}.ts` |
| 4 | Windows 更新替换非原子，失败只能等下次启动自愈 | `packages/core/src/update/files.ts:241-269` |
| 5 | 原生错误串参与控制流（正则匹配注入失败信息），glibc/QQ 改词即永久失败 | `packages/bridge/src/hook-manager.ts:655-657` |

---

## 1. 仓库与分支现状

### 1.1 上游已消失

- `SnowLuma/SnowLuma`（GitHub 组织仓库）：API 返回 `Not Found`，`https://snowluma.github.io/…` 404 —— 属实已下架。
- 上游实际存活节点是个人 fork：**`TomyJan/SnowLuma`**（142 star / 353 fork，`main` 与 `dev` 均在推进，`v2.4.2` release 于 2026-11-09 发布）。
- 最初参照的那个 fork 属**另一个 GitHub 账号**（下文记作 `<FORK_OWNER>`，id `<FORK_ACCOUNT_ID>`）；用于维护的登录账号（下文记作 `<MAINTAINER>`，id `<MAINTAINER_ID>`）对它只有 `pull`、没有 `push`。
  ⇒ 已在维护账号下另建 fork **`<MAINTAINER>/SnowLuma`**，并把 `origin` 指向它，`git push` 已实跑通过（见 §1.4）。

### 1.2 代码规模与作者

- 1319 commits，起点 `b3ffbae0`（2026-05-02，"Initial commit"，作者 手瓜一十雪）。
- 作者分布：DrSmoothl 1066、手瓜一十雪 101、H3CoF6 33、snowluma-bot[bot] 32、墨梓柒 13、时瑾 9、Airline233 8、stla 6、MliKiowa 5、Qiao 5。
  ⇒ **单一核心作者占 80%**，接手时最大的知识断层在 `protocol` 与原生层。
- 15 个 workspace 项目（14 包 + 根），1220 个 TS 文件，约 31.6 万行（含测试）：

| 包 | 源码行 | 测试行 |
| --- | --- | --- |
| `onebot` | 34268 | 40144 |
| `webui` | 34397 | 2481 |
| `protocol` | 32080 | 26014 |
| `mcp` | 26986 | 6251 |
| `core` | 22382 | 32734 |
| `proton` | 8871 | 0 |
| `sdk` | 7622 | 7073 |
| `bridge` | 5590 | 8139 |
| `proto-defs` | 5365 | 0 |
| `common` | 3330 | 3066 |
| `websocket` | 2200 | 4712 |
| `ui` | 1253 | 0 |
| `sqlite` | 570 | 389 |
| `runtime` | 0（只有二进制与启动器） | 0 |

### 1.3 版本、许可、分支模型

- 版本：root 与全部包统一 `2.4.2`，由 `tools/bump-version.mjs` 批量改写。
- 许可：`SnowLuma Source-Available Non-Commercial License v1.0`（2026-06-19，Copyright SnowLumaDevs）。
  - 非 OSI 开源；**禁止商业使用**；**禁止公开发布修改版**；贡献者需授予可再许可的永久许可。
  - Section 5 明确：`snowluma-*.node/.dll/.so` 等 **Native Addon 是独立专有组件，不在该许可内**。
  - ⇒ fork 要公开发布/分发（含 Docker 镜像、lite 包）在许可上是**未授权**的；私下自用、内部部署或取得书面授权后才谈分发。
- 分支模型（`CONTRIBUTING.md`）：`dev` 日常开发；`main` 只是发布指针，不接受 PR；`native/auto-update-*` 由 SnowLuma Bot 从外部 `SnowLumaNative` 仓库自动生成，**不在本仓维护**（本仓也没有原生源码）。
- 已拉取的远端分支 21 个，含 `dev`、`dev2`、`codex/*`、`native/*`、`fix-media-url-retain`、`patch-template-issue`。

### 1.4 远端与取用方式（环境无关）

本文件**不记录任何网络可达性结论**——不写哪条链路通、哪个入口可用、哪种协议认证成功。这些属于部署环境私有信息，写进仓库即等于泄露。下面只给**通用要求**，真实取值与实测结果留在本地。

需要三个远端（角色固定，地址自填）：

| 名称 | 角色 | 要求 |
| --- | --- | --- |
| `upstream` | 上游真身 `TomyJan/SnowLuma` | 能 `fetch` 到全部分支与 tag |
| `origin` | 你自己的**可写** fork | 能 `push`；只读 fork 不够 |
| `mirror`（可选） | 参照用的第三方 fork | 只读，用于对照差异 |

配置与验收，只看**行为**不看猜测：

1. 用哪种协议、走哪条路径、是否用离线介质，全部由环境决定；仓库里不写死任何一种地址或域名。
2. `git push --dry-run origin HEAD` 退出码为 0 才算"推送可用"；`git fetch upstream --tags` 能拉出新 tag 才算"同步可用"。把结果记在本地笔记，不写进本文件。
3. 需要统一改写协议时用 `url.<base>.insteadOf`，避免把地址散落在脚本里。
4. 完全离线时用 `git bundle`：`create --all` → 传输 → `verify` → `clone`，可在不接触任何外部端点的情况下完成同步。

与通道无关的日常命令：

```bash
git fetch upstream '+refs/heads/*:refs/remotes/upstream/*' --tags
git push origin <branch>
gh repo fork TomyJan/SnowLuma --clone=false --remote=false   # 需要新建可写 fork 时
```

> 注意：clone 时 `core.symlinks=false`，`CLAUDE.md`（git mode `120000` → `AGENTS.md`）被检出为 9 字节文本文件。**不要**为了改符号链接而破坏工作树干净状态；要读规范就读 `AGENTS.md`。

---

## 2. 工作原理

### 2.1 进程模型

单进程 Node 服务（`index.mjs`）+ 一个"更新启动器"外壳：

- **发行包**里 `index.mjs` 会先当"父亲"：`updateRoot && !SNOWLUMA_UPDATE_ROOT` 时走 `runApplication(updateRoot)`（`packages/core/src/index.ts:251-264`），先 `applyPendingUpdate()` 应用上次下载好的补丁，再带 `SNOWLUMA_UPDATE_ROOT` 环境变量 spawn 子进程；子进程以退出码 `75`（`RESTART_EXIT_CODE`）表示"我要重启以完成更新"，父亲循环 `do { applyPendingUpdate } while (code === 75)`（`packages/core/src/update/startup.ts:5,45-53`）。
- **开发态**（无 `__PACKAGED__`）直接 `main()`，没有更新外壳。
- 注入目标进程是外部进程（QQ.exe / linux QQ），SnowLuma 只是宿主；DLL 被 manual map 进 QQ 后即使 SnowLuma 重启也会残留，靠 `isPipeLive` 快路径"接管"（`packages/bridge/src/hook-session.ts:415-423`）。

### 2.2 启动时序（`packages/core/src/index.ts:52-129`）

```
loadRuntimeConfig()                     # config/runtime.json + SNOWLUMA_* env 覆盖
await configureFileTransport(...)       # 必须在任何日志之前：确定日志落盘方式
log.info('SnowLuma starting')
logRuntimeIntegrity(await getRuntimeIntegrity())   # 原生 addon：进程完整性级别(RID)
migrateGlobalSettings()                 # 全局配置迁移
cleanupInvalidPerUinConfigs()           # 清理 onebot_<垃圾>.json 幽灵账号（issue #162）
new BridgeManager() + bindSystemFaceCatalog()
new OneBotManager(); oneBotManager.bind(bridgeManager)
resolveAutoLoad()                       # SNOWLUMA_HOOK_AUTOLOAD 覆盖配置
createStateWiring(bridgeManager)        # WebUI 状态总线接线
new HookManager({bridgeManager, autoLoadOnDiscovery, onSessionsChanged})
createNotificationManager(...) + bind
initWebUI(port=5099, ...)               # __BUILD_WEBUI__ 或 SNOWLUMA_DEV_WEBUI=1
```

- WebUI 初始化失败**只记录不致命**（`index.ts:126-128`）——这是有意的：QQ 侧可用性优先于控制台。
- 端口占用时 `findAvailablePort()` 逐个 +1，最多 50 次（`packages/core/src/webui/port.ts:37-56`）。
- 关闭顺序固定且每步独立 try/catch：`oneBotManager.dispose()` → `notificationManager.dispose()` → `stopWebUI()` → `stateWiring.dispose()` → `hookManager.dispose()` → `bridgeManager.dispose()` → `closeLogger()`。注释（`index.ts:176-178`）说明必须**先静默状态观察者**再拆 hook，否则真实 `disconnected` 边沿会污染 UI。

### 2.3 注入与管道协议（原生耦合的核心）

- 加载方式：`process.dlopen({exports:{}}, addonPath)`，不是 `require()`（`packages/bridge/src/injector.ts:35-39`）；文件名 `snowluma-<platform>-<arch>.node`，搜索顺序 `<moduleDir>/native` → `../../runtime/native`（`packages/common/src/native-path.ts:7-21`）。**懒加载**是刚性设计：更新程序必须在打开任何原生文件前完成替换。
- 必需导出：`getRuntimeIntegrityAsync`、`getAllMainProcessAsync`、`getFileHoldersAsync`、`loadModuleManualAsync(pid, dylibPath)`、`unloadModuleManualAsync(pid, handle)`；缺符号直接硬失败 `Hook native addon lacks getAllMainProcessAsync`（`injector.ts:100-103`）。
- TS **从不解析 `QQ.exe` 路径**：主进程 PID 由原生枚举给出，TS 只合成 `name` 并置 `path: ''`（`injector.ts:106-109`、`hook-manager.ts:651-653`）。注入返回手动映射句柄 `{base, entry, exceptionTable, size}`。
- 传输：每 PID 两条管道。Windows `\\.\pipe\mojo.<pid>.control` / `.recv`；POSIX `<runtimeDir>/mojo.<pid>.control.sock`（`packages/bridge/src/qq-hook-client.ts:172-182`）。存活探测为单次 `readdir('\\\\.\\pipe\\')` + `/^mojo\.(\d+)\.control$/i`（`:659-671`）。
- 帧格式 v1，全小端，40 字节头（`qq-hook-client.ts:17-44`，`encodeFrame :408-419`，`FrameReader :474-486`）：

| 偏移 | 字段 |
| --- | --- |
| 0 | magic `0x31504851`（"QHP1"） |
| 4 | version = 1 |
| 6 | op（hello=1, sendRequest=2, sendAck=3, sendReply=4, error=5, recvPacket=6, loginState=7, loginIdentityHint=17, clientMessageReceipt=21, clientMessageState=22） |
| 8 | requestId |
| 12 | status |
| 16 | flags（`1<<0` wantReply、`1<<2` loggedIn） |
| 20/24/28 | cmdLen / msgLen / bodyLen |
| 32 | value0（u64） |

  上限：cmd 4 KiB、msg 64 KiB、body 16 MiB（`:23-25`，注释要求与 `nnphook` 的 `pipe_protocol.hpp` 手工保持同步）。
- 登录语义**直接读帧**：`loggedIn = flaggedLoggedIn || frame.status !== 0`；`uin = frame.msg || value0.toString()`（`:1297-1304`）。状态约定是隐式契约。
- 心跳靠命令名识别：`trpc.qq_new_tech.status_svc.StatusService.SsoHeartBeat`，首见即启动，90s 判陈旧、15s 确认（`hook-session.ts:34-40`）。
- 连接预算：**跨所有重试共享 60 秒**（`hook-session.ts:27,680-681,790-803`），并以 `Promise.race([connectAll, deadline])` 防止握手卡死阻塞串行操作链（`:818-821`）。

### 2.4 账号识别（不依赖 QQ 告诉你 UIN）

QQ 的 `login.db` 是 SQLCipher v4：

- 口令硬编码 `BD156D6710D54D8782F4`，字段 `1000`(uin)/`1001`(uid)/`1007`(nickname)，表 `login_table`（`packages/bridge/src/qq-account/login-db.ts:9-12,130`）。
- 纯 TS 重实现页格式：1024 字节客户端头、4096 页、页尾 `IV(16)||HMAC` 补到 AES 块、PBKDF2 4000 轮、页 HMAC 密钥经 2 轮对 `salt^0x3a` 派生；算法对在 `{none,sha1,sha256,sha512}×{sha1,sha256,sha512}` 上**探测**（`sqlcipher.ts:24-38,52-54`），`-wal` 帧合并。代码自己承认"算法跨客户端构建不固定，所以探测"（`sqlcipher.ts:17`）。
- **只读**：解密写进 `mkdtemp` 副本再打开，QQ 原文件绝不以写方式打开（`login-db.ts:118-125`）；结果按 `mtimeMs:size` 缓存。
- PID↔UIN 绑定靠"谁持有这个文件"：Windows 调 `getFileHoldersAsync`，Linux 查 POSIX 锁表按 device+inode 匹配（`database-holders.ts:17-60`）。
- 路径：Windows `nt_qq/global/nt_db/login.db`、根目录取自 `.../Tencent/QQ/UserDataInfo.ini` 的 `[UserDataSet] UserDataSavePath`；`nt_msg.db` 在 `<root>/<uin>/nt_qq/nt_db/`。Linux 走 `nt_qq_<md5(hex(md5(uid))+"nt_kernel")>/nt_db/` 这一 md5 链（`paths.ts:74-96,103-161`）。
- 因为原生侧存在竞态，才加了 UIN 合理性闸门 `/^\d{5,10}$/`（`packages/common/src/uin.ts:9-11`），否则 13 位时间戳会被当成 UIN。
- **全仓没有任何 QQ 客户端版本检查**：兼容性是被假设的，不是被验证的。

### 2.5 一条 QQ 消息 → OneBot 事件

**先澄清一个容易误解的点：JS 侧不做任何 QQ 协议解密（SSO 层）。** 解密、解包 SSO 都在注入的 C++ 里完成，管道里送过来的已经是"明文+已解压"的包体；JS 只做二次整形、protobuf 解码与业务语义解析。唯一的 JS 侧解密是**账号识别**读 `login.db`（`bridge/qq-account/sqlcipher.ts:254` 的 `createDecipheriv('aes-256-cbc')`，见 §2.4），与 SSO 链路无关。**唯一的压缩**是消息体内部：`msg-push/helpers.ts:65` 的 `decompressData`，首字节 `0x01` → `inflateSync`（上限 4/8 MiB），`0x00` → 直接 UTF-8。

```
管道 recvPacket(op=6) 帧
  → hook-session.ts:1245-1253 整形为 PacketInfo{pid, uin, serviceCmd, seqId, retCode, fromClient:false, body}
  → core/src/bridge/bridge.ts:204-219 闸门（identity.initialization；source==='client-message' 走另一条分支）
  → packet-pipeline.process(pkt)  按 serviceCmd 取已注册 parser（:87-89）
       ├─ MSG_PUSH_CMD  → parseMsgPushOrThrow（bridge.ts:80 注册）
       └─ KICK_NT_CMD   → parseKickNT
  → msg-push/index.ts:105-236  上下文重建 → sysmsg 去重闸门(:128) → registry 解码 → 存活过滤(:161)
  → protobuf_decode<PushMsg>(pkt.body)（context.ts:48-77；proton 编译期生成的 codec，运行时无 schema）
  → 归一化为 QQEventVariant（events.ts:612 起，23 种 kind）
  → packet-pipeline.ts:408-412 `events.emit(event)`（BridgeEventBus，异步 Promise.allSettled）
  → IdentityService 补 uid/uin/群成员身份
  → onebot: CONVERTERS[kind] → OneBot v11 事件 JSON（envelope 统一注入 self_id + time）
  → EventPipeline → 存储(messages.db) + 广播(WS/HTTP-POST/SSE)
```

要点：

- `CONVERTERS` 是对 `QQEventVariant` 全集的**映射类型**：新增 QQ 事件不写转换器会直接编译失败（`packages/onebot/src/event-converter/index.ts:68-135`）。这是本项目最好的防回归设计之一。
- 事件字段中有相当多是**合成值**，改它们会改变下游语义：`message_id` 由 `sha1(sequence:sessionId:eventName)` 取前 4 字节（含负值，`message-id.ts`）、`self_sent` 用 `post_type=message_sent` + 合成 `target_id`、`sender.sex='unknown'`/`age=0`/`font=0`、群 `sub_type='normal'`/`role=role||'member'`/`anonymous=null`、通知 `invitor_id=0`/`invitor_status='unknown'`。
- `message_id` 是**不可变契约**：非零有符号 int32，负数合法（测试固定 `-585543322`）。改动会破坏所有历史消息引用。

#### 2.5.1 富文本元素解码（`packages/protocol/src/msg-push/rich-body-decoder.ts`）

- 入口 `decodeRichBody:645` / `convertElements:667`，内部联合类型 **16 种**（`events.ts:202-231`）。
- 映射主干：`text → text|at`（`attr6Buf>11` 时取 targetUin，否则走 `MentionExtra` type 1/2）、`face → face`、`marketFace → mface`、`notOnlineImage → image`（`fileId = filePath ?? md5Hex`，附 size/w/h/subType，摘要 `[图片]`/`[动画表情]`）、`customFace → image`、`videoFile → video`（保留 mediaNode）、`groupFile`/`transElem(24) → file`、`richMsg|lightApp → json|xml|forward`（先 `decompressData`；`MultiMsg_<uuid>` 则是 forward 的 resId）、`srcMsg → reply`（递归 elemsRaw）、`commonElem → poke(2)/flash_file(3)/image|record|video(48 且 biz 10~22)/face(33,37)/markdown(45)/inline_keyboard`。
- **失败开放（fail-open）**：未知 tag 只打一条 `wire element ignored unknownTag=…`（`:463`、`:717`），`DECODED_WIRE_FIELDS` 是白名单（`:45-49`），未匹配的 `commonElem` 只记录 serviceType/businessType。⇒ QQ 加新元素时消息不会崩，但**会被静默丢内容**，这是排障时第一个要开 trace 看的地方。
- 图片 URL 三套兜底（`helpers.ts:14-56`）：rkey/fileid → `https://multimedia.nt.qq.com.cn`，否则 `http://gchat.qpic.cn`，再退化到 `gchatpic_new/0/0-0-${MD5}/0`；`ntImageUrlFromFileId` 按 appid 1406(私聊)/1407(群) 区分。
- 媒体上传四条链：图片 `0x11c4/0x11c5`（cmdId 1004/1003）、PTT `0x126e/0x126d`（silk + 波形合并，requestId 1 群 / 4 私）、视频 `0x11ea/0x11e9`（requestId 3，缩略图 1002/1006，`compatQmsgSceneType` 硬编码 2）、群文件 `0x6d6_0`（uinForm）再由 `MessageSvc.PbSendMsg` 的 cmdType 71/95 发出。

### 2.6 一条 OneBot action → QQ 请求

```
WS/HTTP 收到 {action, params, echo}
  → api-handler.handle → handleInContext（实例正在关闭则 ACTION_FAILED）
  → ACTION_REGISTRY 查表（未知 action → UNKNOWN_ACTION）
  → 参数 parse：类型强制 + 跨字段校验，首个 Err 即返回 BAD_REQUEST "field: reason"
  → action.run(ctx) → ctx.bridge.apis.<area>.method()（Core 的 13 个 Api 类）
  → protocol 组包 → 管道 sendRequest 帧 → QQ
  → 帧回 reply/ack → 组装 OneBot 响应信封 {status, retcode, data}
```

- Action 定义是"单一定义双用途"：`defineAction({name, summary, returns, returnsSchema, examples, readOnly, params, rules, run})`（`packages/onebot/src/action-kit.ts:749-762`）。`returnsSchema`/`examples` 运行时不校验，只被 MCP 目录生成器与 CI 漂移检查强制（`packages/mcp/scripts/gen-catalog.test.ts:21-22`）。
- 规模：**222 个声明式 action + 6 个别名 + 2 个 raw 名 = 230 个可执行名**，与 `packages/sdk/src/actions.ts` 的 `SNOWLUMA_ACTIONS` 一一对应（覆盖率由 `packages/onebot/tests/sdk-actions.test.ts` 校验）。
- 分区：info 5、message 6、friend 3、group-info 7、group-admin 24、group-file 12、request 2、extended 115、group-album 9、qzone 12、qzone-public 17、system-face 4、stream-file 2、stream-download 4。
- retcode 集合 `{100, 1200, 1400, 1404}`；流式 action 用 `STREAM_MARK='stream-action'` 包装帧（`packages/onebot/src/streaming.ts:19,62,88`）。
- 别名举例：`fetch_ptt_text|get_ptt_text|get_record_text`、`ocr_image|.ocr_image`、`send_packet|.send_packet`、`.get_word_slices`。

#### 2.6.1 协议请求是怎么发出去的

- 统一信封（`packages/protocol/src/bridge-oidb.ts:1-27`）：`{command, subCommand, errorCode:0, body, errorMsg:'', reserved: uinForm ? 1 : 0}`。
- 分发器 `oidb-service.ts:134 invokeOidb`：wireName 拼成 `OidbSvcTrpcTcp.0x<cmd>_<sub>` → `ctx.sendRawPacket(cmd, spec.encode(env), timeoutMs)`；`gotResponse=false` 即 Error；返回码不在 `acceptedEnvelopeCodes` 里抛 `OIDB error ${code} on 0x${command}_${subCommand}: ${serverMsg}`。
- 发消息走 `MessageSvc.PbSendMsg`（`packages/core/src/bridge/apis/message.ts:41`），群路径 `:443`；失败文案是排障关键词：`send group message failed:`、`failed to decode SendMessageResponse`、`send group message rejected: result=… err=…`、`group message delivery was not confirmed…`。
- 序列号：`nextClientSequence` 自增、`nextMessageRandom` 用黄金比例 `0x9e3779b9 & 0x7fffffff`（`bridge.ts:313-318`）。
- 超时：默认 **15s**，可用 `SNOWLUMA_PACKET_TIMEOUT_MS` 调（`packet-timeout.ts:4`）；未登录时返回 `{success:false, gotResponse:false, errorCode:-1}` 而不是抛错。
- 管道 `requestId` 是 uint32 循环，**跳过 0**（`qq-hook-client.ts:558,1074-1089`）——0 在原生侧另有含义（`clientMessageReceipt` 契约要求 `requestId===0`），改这里会连累回执识别。

### 2.7 传输层（OneBot 侧）

| 传输 | 默认 | 关键行为 |
| --- | --- | --- |
| WebSocket 服务端 | `127.0.0.1:3001`，role `Universal`（同样有 `?? '0.0.0.0'` 回落） | 路径不匹配 → 400 `Bad path for WebSocket`；鉴权失败 → 401；事件只发给 `Event|Universal`；30s 心跳、2 次丢失≈90s 判定 |
| WebSocket 客户端 | 无 | 反向连接，签名含 `url/role/token/maxPayload`；默认 5000ms 重连 |
| HTTP 服务端 | `127.0.0.1:3000`，路径 `/` | 前缀匹配 + `isAuthorized(..., allowRawToken=true)`；`_events` 需 `enableSse`（否则 404）；只接受 GET（405）；裸 GET 返回 `{online:true}`；默认 `config.host='127.0.0.1'`，但**代码回落是** `config.host ?? '0.0.0.0'`（`http-server-adapter.ts:316,356`）⇒ 配置里删掉 host 就会变成全网卡监听 |
| HTTP POST 上报 | 无 | `X-Self-ID` + `X-Signature=HMAC-SHA1(token,payload)`，5s 超时，响应体当作 quick operation 执行 |
| SSE | 无 | 重放 256 事件 / 8 MiB，单客户端 8 MiB；`Last-Event-ID` 重放，无效 400、过期/越权 409；15s ping；重启或 token 轮换即失效 |

认证工具 `packages/onebot/src/network/utils.ts:12-38`：`Authorization: Bearer`、可选裸 token、`?access_token=`。**空 token 等于关闭认证**（`http-server-adapter.ts:410`）——公开部署前必须显式设 token。

### 2.8 存储模型

```
config/
  runtime.json              # 进程级配置
  webui.json                # WebUI 认证（含 mustChangePassword、TOTP、会话快照）
  onebot.json               # 全局 OneBot 覆盖
  onebot_<uin>.json         # 每账号权威配置
  cert.pem / key.pem        # WebUI TLS
data/<uin>/
  snowluma_identity.db      # uid/uin/群成员/好友身份缓存
  messages.db               # 消息 + 媒体索引（同一个文件，两个 writer，见 §7 第 6 条）
  reactions.db              # 表态
logs/                       # 按日滚动，<uin>/ 子目录可选，总量/天数受限
tmp/update/                 # 更新暂存（存在则优先，否则落到系统临时目录）
```

- 消息表 `messages(message_hash PK, is_group, session_id, sequence, sequence_authoritative, event_name, client_sequence, private_direction, random, timestamp, data, media_data, classification_version)` + 4 个索引 + 迁移账本（`message-store-migration.ts:320-334,418-427,462`）。
- 旧 `.json` 会迁到 `.db`；`media.db` 只复制不删除（`media-store-migration.ts:9-19`）。
- **持久数据不做自动删除**（AGENTS.md 硬规则）；只有撤回才写墓碑。
- 身份库 `snowluma_identity.db`（`packages/protocol/src/identity-service.ts`）：**内存才是权威**，SQLite 只是快照/可选（`openForUin` 传 `null` 即纯内存）。表：`users(id,uid UNIQUE,uin UNIQUE,nickname,remark,is_friend,source,updated_at)`、`groups(group_id PK,…member_count,member_max,active,updated_at)`、`group_members(…role default 'member',is_robot,active,updated_at, UNIQUE(group_id,uid), UNIQUE(group_id,uin))`，另有热缓存、**上限 256 的待写队列**、重试阶梯 `[100,500,2000,10000,30000]ms` 与 `persistenceStatus`。
  - 迁移是**临时的**：`migrateSchema :1206` 用 `PRAGMA table_info` 判断后 `ALTER TABLE`，**没有 `user_version`**。加字段时请同时补这一处，否则老库缺列。
  - `resolveUid :525` 会抛（缓存→群成员列表→资料），`resolveUin :572` **永不抛**；查找顺序 numeric-uid 快捷 → 群成员 → 内存 → 持久化成员键 `${groupId}:${uid}` → 全局。

### 2.9 WebUI（后端 = Hono，前端 = Vite/React）

- 后端 `packages/core/src/webui/server.ts`（~2300 行单文件，本身就是一个维护热点）。
- 认证：`x-snowluma-token` 头（兼容 `Authorization: Bearer`）；Cookie **只作用域到 `/avatar`**、`httpOnly`、`SameSite=Strict`（`request-security.ts:44-62`），好让 `<img src="/avatar/...">` 带上凭证。
- 会话是**内存 Map**，TTL 24h，每 IP 5 次失败锁 15 分钟（`server.ts:125-144`）⇒ 每次重启所有运维会话失效。
- 初始口令：`randomBytes(8).hex`，只打印一次（`auth.ts:240-321`、`server.ts:686-694`），可用 `SNOWLUMA_WEBUI_BOOTSTRAP_PASSWORD` 播种，强制改密 + EULA/PRIVACY 同意闸门在服务端强制执行。
- 流式一律 **SSE，没有 WS**：`/api/state/stream`（50ms 去抖，背压丢帧并补 `{kind:'dropped'}`）、`/api/logs/stream`、`/api/debug/stream`。
- 前端构建到 `dist/client`（`packages/webui/vite.config.ts:38-41`），core 每次请求 `readFileSync(index.html)`（`static-frontend.ts`）⇒ 改前端必须重新构建；dev 模式 `:5178` 代理 `/api`、`/avatar` 到 5099。

### 2.10 自更新

- 版本源：GitHub Releases 页面/API（`SnowLuma/SnowLuma` 硬编码 4 处，见 §6-P0-2），镜像列表可配（内置若干公共镜像，见 `update/sources.ts` 的 `DEFAULT_UPDATE_SOURCE`：`mode` 取 `auto`/`direct`/`mirror`）。
- 校验：**只有 sha256**，且摘要来自同一渠道（releases HTML 的 `<clipboard-copy value="sha256:…">` 或 API JSON）；镜像只是明文传输层，无独立认证。**无签名**。
- 落盘：暂存目录 → `transaction.json.tmp` + `rename` 提交 → 各条目拷入 `<root>/.update-install` 再 `rename` → 删除旧版独有条目 → `rename(pending, completed)` 作为提交点；失败回滚备份，二次失败抛 `AggregateError('更新失败，恢复未完成；下次启动将继续恢复')`（`files.ts:238-269`）。
- 闸门：仅发行包可用（`在线更新仅支持发行包`）、容器内拒绝（`请更新容器镜像`）、必须更高且非预发布版本。

### 2.11 SDK / MCP / UI

- SDK：9 个入口（`.`、`client`、`messages`、`events`、`types`、`errors`、`actions`、`node`、`package.json`）；`node` 入口提供本地文件流与**反向 WS 服务端**（默认路径 `/onebot/v11/ws`——注意与 OneBot 默认 `/` 不一致，`REVERSE_WEBSOCKET.md` 有说明）。
- SDK 版本策略：`VERSIONING.md` 明确 Major/Minor/Patch 判据，OIDC 可信发布。
- MCP：`pnpm --filter @snowluma/mcp gen` 生成 `packages/mcp/src/generated/catalog.ts` + `catalog.json`；`tools/check-action-catalog.mjs` 用 `git status --porcelain` 卡漂移；server 用 `SNOWLUMA_MCP_ENDPOINT/MODE(docs|read|write)/TOKEN/EVENT*` 等 env 门控，默认 `read`。
- UI：`packages/ui` 是发布到 npm 的组件包（`@snowluma/ui`）。

### 2.12 protobuf：没有 `.proto` 文件，只有 TS 接口 + 编译期生成

这是接手后最容易踩坑的一块：

- `packages/proto-defs` 是 **30 个手写 `.ts`**、5365 行、739 个 interface、2809 个字段（133 repeated / 100 optional），**全仓没有任何 `.proto`**。来源是社区逆向（Lagrange/NapCat 的 tag 对照，`oidb-actions/message.ts:137` 有明确注释）。
- `packages/proton` 是**编译期** codec 生成器（vendored 自 SnowLuma/protobuf-dsl，MIT）：你写 `pb<tag,type>` 接口，Vite 插件（`enforce:'pre'`，`src/index.ts:1-74`）在 `prepare(buildEntries)` 阶段扫描并**改写调用点** `protobuf_encode<T>/protobuf_decode<T>`；未被改写的调用会在运行时抛 `…was not transformed by the @snowluma/proton Vite plugin`。生成器只 emit 可达 codec，`sizeBound<=64` 时用定长缓冲。
- 插件已接入：`packages/core/vite.config.ts:139`（import 在 :1）、`packages/core/vitest.config.ts:4`、`packages/onebot/vitest.config.ts:5`、`packages/mcp/vitest.config.ts:5`、`packages/protocol/vitest.config.ts:3`。⇒ **新增 proto 定义后，必须确认这些 vitest 配置也加载了插件**，否则测试里会抛"未转换"。
- 校验手段不是往返编解码，而是 `packages/protocol/tests/oidb-services/_pb-oracle.ts`：用反推出来的 tag 从第一性原理构造期望字节，能抓到"往返都对但 tag 写错"的错。24 个测试文件用它。
- 风险：没有 schema 主源、没有版本号、字段靠逆向。QQ 换 tag 时表现为"字段串位/莫名空值"，不会报错。维护姿势：改 interface → 加/改 oracle 期望字节 → 跑 protocol 测试 + 全量 typecheck。

---

## 3. 包职责矩阵（改哪里先看谁）

| 包 | 职责 | 入口 / 关键文件 |
| --- | --- | --- |
| `core` | 应用入口、账号级 Bridge 编排、WebUI 后端、通知、发行装配 | `src/index.ts`、`src/bridge/manager.ts`、`src/webui/server.ts`、`src/update/*`、`vite.config.ts` |
| `bridge` | QQ 进程发现、账号/登录识别、Hook 会话、原生传输 | `src/hook-manager.ts`、`src/hook-session.ts`、`src/qq-hook-client.ts`、`src/injector.ts`、`src/qq-account/*` |
| `protocol` | 解包解码、类型化 QQ 事件、身份解析、协议请求与媒体传输 | `src/packet-pipeline.ts`、`src/msg-push/index.ts`、`src/identity-service.ts`、`src/highway/*` |
| `onebot` | action 校验与分发、事件转换、消息存储、HTTP/WS/SSE 适配 | `src/actions/index.ts`、`src/action-kit.ts`、`src/instance.ts`、`src/event-converter/*`、`src/network/*` |
| `common` | 日志、配置、路径、版本、共享工具 | `src/runtime.ts`、`src/native-path.ts`、`src/version.ts` |
| `sqlite` | 异步 SQLite 门面、原生加载、预编译选择 | `src/index.ts`、`src/driver.ts`、`src/native.ts`、`src/layout.ts`、`src/vite.ts` |
| `websocket` | WS 实现 + 原生解析器绑定 | `src/native.ts`、`src/websocket.ts` |
| `proto-defs` / `proton` | protobuf schema 与编译期编解码生成 | `packages/proton/README.md`、各包 `vitest.config.ts` |
| `runtime` | 发行清单、启动器、随包原生件 | `check-node-version.cjs`、`launcher.bat/.sh`、`native/*` |
| `webui` | Vite/React 控制台 | `src`、`vite.config.ts`、`INTEGRATIONS.md` |
| `sdk` | 公开 TS 客户端 | `src/index.ts`、`VERSIONING.md`、`STREAMING.md` |
| `mcp` | MCP server + 生成的 action 目录 | `src/server.ts`、`scripts/gen-catalog.test.ts` |
| `ui` | 可发布 UI 组件包 | `src` |
| `tools` / `.github/workflows` | 校验、打包、发布自动化 | `tools/check-release-layout.mjs`、`tools/release.mjs`、`release.yml`、`promote-dev-to-main.yml` |

---

## 4. 构建、测试、发布

### 4.1 命令表（根目录执行）

```bash
pnpm install --frozen-lockfile      # 依赖（Node 版本见 §6-P0-1）
pnpm typecheck                      # 14 包类型检查
pnpm lint                           # ESLint
pnpm test                           # 工作流策略检查 + 全 workspace 测试（会顺带重生成 MCP catalog）
pnpm test:workflows                 # 只跑 PR 策略测试，无 GitHub 写入
pnpm test:core                      # 只跑 core，不代替全量
pnpm test:native                    # 原生覆盖，需要匹配的原生件
pnpm run build:all                  # 发行输入 + 可发布包（core→sdk→mcp→ui→webui）
pnpm format / format:check          # 4 空格缩进、控制语句带花括号
pnpm run ci                         # 本地复刻 CI 全部命令（含安装与 catalog 漂移检查）
```

只跑单包：`pnpm --filter @snowluma/onebot test`、`pnpm --filter webui test`（前端包无 `@snowluma/` 前缀）。

### 4.2 构建产物与布局约束

- `packages/core/vite.config.ts` 打**单 bundle** `index.mjs`（`codeSplitting:false` `:192`），external 只允许 Node 内置模块，`generateBundle` 会拒绝任何非 builtin import（`:95-99`）。**这条约束不能破**：更新程序需要一个能被整体替换的入口文件。
- 构建时校验原生件与法律文件存在（`:66-89`），并把 `check-node-version.cjs`、`launcher.*`、`EULA/PRIVACY`、`native/*`、`ffmpeg` 拷进 `dist/`；`release.json = {version,target,entries}` 只在 `BUILD_WEBUI=true` 时生成（`:107-129`）。
- `tools/check-release-layout.mjs`：`release.json.version` 必须等于 `package.json.version`，`entries` 必须等于根目录 `readdir` 去掉 `node`/`node.exe`，sqlite 清单/target/libc 精确匹配，且**禁止任何 `/snowluma/i` 命名的目录**。
- SQLite 预编译：`packages/sqlite/src/vite.ts:26-66` 产出 `native/sqlite/prebuilds/<target>/@appthreat+sqlite3[.glibc|.musl].node` + `LICENSE.sqlite3` + `manifest.json`（驱动 `@appthreat/sqlite3` 9.1.0，native interface 1，NAPI 10）。
- Lite 包 = `dist/*`；Full 包 = `dist/*` + 内置 `node/node.exe`。Node 版本闸门 `^24.22.1 || >=26.12.1`（`check-node-version.cjs`，25.x 与预发布一律拒绝）。
- ⚠️ `launcher.bat` **缺 `cd /d "%~dp0"`**：从快捷方式（工作目录非安装目录）启动会找错 `config/`、`data/`。`launcher.sh` 有 `cd`。

### 4.3 CI 与发布流水线

| 工作流 | 触发 | fork 内可否运行 |
| --- | --- | --- |
| `ci.yml`（test / typecheck / build / lint） | PR、dispatch | ✅ 可用 |
| `dev-build.yml`（构建 job） | push dev | ✅ 可用 |
| `release.yml`（v* tag → 三平台矩阵 → lite/full 资产） | tag `v*` | ✅ 可用（需 `contents: write`） |
| `dispatch-docker-release.yml` | release 后 | ❌ 依赖 `SNOWLUMA_TOKEN` + org 变量 + 外部仓库 |
| `promote-dev-to-main.yml` | push dev / tag chore.* | ❌ 依赖 GitHub App（`SNOWLUMA_BOT_APP_ID`/`_PRIVATE_KEY`） |
| `sync-docs-catalog.yml` | 生成目录后 | ❌ 依赖 App + `SnowLumaDocs` 仓库 |
| `sdk-npm-publish` / `mcp-npm-publish` / `ui-npm-publish` | tag | ❌ OIDC Trusted Publisher 绑定原 org/repo/workflow |
| issue/stale 策略类 | issue/PR | ✅ 可用 |

发布流程（`tools/release.mjs`）：预检（分支必须 `dev`、干净、不落后）→ 本地全量 CI（`tools/ci-check.mjs` 复刻 6 条命令）→ `bump` → `chore(release): vX.Y.Z` → push dev（8 次重试）→ 轮询 `origin/main == release SHA`（30 分钟 / 15s）→ `git tag vX.Y.Z` + push。promote 工作流用 `force-with-lease push origin/dev:refs/heads/main` 让 main 追上 dev，门禁为提交前缀 `^\[merge\]|^chore\(release\):`，且 main 上不能有独有提交。

> 注释里说"发完切回 dev"（`tools/release.mjs:25-26`）但**没有实现**——这是已知的流程缺口，发版后需人工切回。

### 4.4 测试资产地图（不接 QQ 也能测）

这套测试基建是项目最值钱的部分之一，**改代码前先看能不能复用现成替身**：

| 场景 | 替身 | 文件 |
| --- | --- | --- |
| 管道线协议（帧编解码、分片、长度上限、回执校验） | 临时目录起真实 Unix socket `mojo.<pid>.control.sock`，用真实 40 字节头写帧 | `packages/bridge/tests/qq-hook-client.test.ts` |
| 连接/断开/重连、不上报假登录 | 真实 `PipeWatcher` + `HookManager` + `FakeClient` | `packages/bridge/tests/hook-reconnect-no-login-e2e.test.ts` |
| 注入与原生路径解析 | mock `fs.existsSync` + 重定义 `process.platform/arch` | `packages/bridge/tests/injector.test.ts` |
| `login.db` 解密 | **自实现 SQLCipher 加密侧**造合成库（`encryptImage`/`encryptWal`） | `packages/bridge/tests/qq-account-fixtures.ts` |
| 真实 QQ 数据 | 设 `SNOWLUMA_QQ_DATA_ROOT` 才跑，默认跳过 | `packages/bridge/tests/qq-account-real-data.test.ts` |
| 账号级编排（归属、关闭、订阅） | 假 `PacketSender` + 真实内存 `IdentityService` | `packages/core/tests/bridge-manager-*.test.ts` |
| 更新链路 | `tests/helpers/update-fixture.ts` 造包 | `packages/core/tests/update-*.test.ts` |
| WebUI 认证/TOTP/SSE | 直接打 Hono 实例，无浏览器 | `packages/core/tests/webui*.test.ts` |
| protobuf tag 正确性 | `_pb-oracle.ts` 第一性原理构造期望字节 | `packages/protocol/tests/oidb-services/**`（24 个文件复用） |
| Highway 上传 | 真实 `net.createServer` loopback 回 `packHighwayFrame` | `packages/protocol/tests/highway/upload-transport.test.ts` |
| SQLite 并发/门面 | 真实文件 + 第二连接 `BEGIN IMMEDIATE` 证明事件循环未被阻塞 | `packages/sqlite/tests/sqlite.test.ts` |
| WS 协议 | RFC6455 向量 + loopback 客户端/服务端 | `packages/websocket/tests/*` |
| SDK 契约 | 名字必须等于 `ACTION_REGISTRY.executableNames`（含别名与 raw），参数 fixture 必须与 `describe().params` 对齐；另有临时目录装包验证消费者可用 | `packages/onebot/tests/sdk-actions.test.ts`、`sdk-action-params.test.ts`、`packages/sdk/tests/package.test.ts` |

缺口：**没有一条测试覆盖"真 QQ + 真原生件"**，也没覆盖 QQ 版本升级后的兼容性。这是接手后最该补的一条冒烟链路。

---

## 5. 运行手册

### 5.1 首次启动

```bash
pnpm install --frozen-lockfile
pnpm run build:all
cd dist && node index.mjs          # 或双击 launcher.bat（注意 §4.2 的 cd 缺陷）
```

启动后：

1. 控制台横幅打印 `initial credentials: user=admin password=<16位hex>` —— **只打印一次**，`config/webui.json` 里 `mustChangePassword` 会强制改密。
2. 首次访问 WebUI 需同意 EULA/PRIVACY。
3. 打开 `http://127.0.0.1:5099`，在「进程」页选 QQ 进程 → 注入 → 等 UIN 识别 → 在「连接」页配 OneBot HTTP/WS/SSE。

### 5.2 配置

`config/runtime.json`（`packages/common/src/runtime.ts`）：

| 键 | 默认 | 说明 |
| --- | --- | --- |
| `webuiPort` | `5099` | 占用则 +1 重试最多 50 次 |
| `webuiHost` | `127.0.0.1` | 监听地址，改需重启 |
| `webuiTls.enabled` | `false` | 证书放 `config/cert.pem`、`config/key.pem`；TLS 失败**不得**静默降级 HTTP |
| `trustProxy` | `''` | 反代时设置，否则客户端 IP 全是 127.0.0.1 |
| `hookAutoLoad` | `false` | 发现 QQ 进程即自动注入；Docker 镜像在 supervisord 里打开 |
| `logMaxTotalMb` | `1024` | 日志总量上限 |
| `logRetainDays` | `7` | 0 = 不按日期清理 |
| `logPerUin` | `false` | 额外写 `logs/<uin>/` |
| `updateSource` | 内置镜像列表 | `{mode: auto|direct|mirror, ...}` |

环境变量覆盖（优先于文件）：`SNOWLUMA_WEBUI_PORT`、`SNOWLUMA_WEBUI_HOST`、`SNOWLUMA_WEBUI_TLS`、`SNOWLUMA_WEBUI_TRUST_PROXY`、`SNOWLUMA_WEBUI_BOOTSTRAP_PASSWORD`、`SNOWLUMA_LOG_MAX_TOTAL_MB`、`SNOWLUMA_LOG_RETAIN_DAYS`、`SNOWLUMA_LOG_PER_UIN`、`SNOWLUMA_HOOK_AUTOLOAD`、`SNOWLUMA_UPDATE_ROOT`（内部）。

OneBot 侧默认值：HTTP `127.0.0.1:3000` 路径 `/`；WS `127.0.0.1:3001` role `Universal`（token 随机生成）；每账号 `config/onebot_<uin>.json` 权威、`config/onebot.json` 作覆盖；**配置文件损坏会中止启动**（`config.ts:103-140`）。

### 5.3 Docker

镜像默认 `motricseven7/snowluma`，平台 `linux/amd64,linux/arm64`，dev 分支出 `dev` / `dev-<12sha>` 滚动 tag；容器内更新被禁用（提示改用镜像）。fork 自建镜像需改 `release.yml` 的 `vars.DOCKER_RELEASE_REPOSITORY` 与镜像名，并自带 registry 凭证。

### 5.4 更新与回滚

- 在线更新只对发行包生效；容器内拒绝；只接受更高正式版。
- 更新 = 下载 zip/tar.gz → sha256 校验 → 暂存 → 下次重启由父进程替换；中断后下次启动先 `restore()` 再继续。
- 手动回滚：保留旧版整个目录（更新会删掉"旧版独有条目"，没有版本目录隔离），因此**升级前整目录备份**是唯一可靠回滚手段。

### 5.5 故障排查速查

| 现象 | 先查 |
| --- | --- |
| 进程列表空 | 原生 addon 是否加载（`logRuntimeIntegrity` 有无输出）、QQ 是否 NT 版、Windows 权限是否低于 Medium |
| 注入后一直 `connecting` | 60s 连接预算已耗尽（`hook-session.ts:27`）；管道 `mojo.<pid>.control` 是否存在；DLL 是否残留在旧 PID |
| 有进程无 UIN | `login.db` 解密失败（口令/算法轮换）、`getFileHoldersAsync` 缺符号、UIN 闸门 `/^\d{5,10}$/` 拦掉 |
| `bad frame header magic=… version=…` | TS 与原生管道协议版本不一致（换过原生件） |
| action 返回 `UNKNOWN_ACTION` | 名字不在 `ACTION_REGISTRY`（别名/raw 名见 §2.6） |
| WebUI 登录被登出 | 进程重启（会话在内存）；或 TOTP 启用时其他会话被作废 |
| 更新报 `更新包清单无效` | `release.json` 的 `entries` 与白名单不符（多/少文件） |
| 更新报 `发布页面缺少唯一的文件校验信息` | 上游 release 页面结构变了（HTML 解析依赖 `<clipboard-copy value="sha256:…">`） |

---

## 6. 维护清单

按"必须先做 → 值得做 → 有余力再做"排序。每条给出位置、现状、处置建议。

### P0 — 接手前必须处理

**P0-1 工具链与 Node 版本**

- 现状：`.npmrc` 里 `engine-strict=true`，而 `engines.node = ^24.22.1 || >=26.12.1`；低于该基线的本地 Node（例如 `<LOCAL_NODE>`）会被**拒绝**（勘察时是用 `--config.engine-strict=false` 临时装上的）。
- 处置：装 Node 24.22.1+（或 ≥26.12.1），并让 `check-node-version.cjs`、`.node-version`、各包 `engines` 三者继续对齐。**不要**为图省事放宽闸门——它挡的是已知 CVE 版本。

**P0-2 更新链路全部指向已消失的仓库**

- 位置（4 处硬编码 + 1 组镜像）：
  - `packages/core/src/update/sources.ts:8` `RELEASE_ROOT = 'https://github.com/SnowLuma/SnowLuma/releases'`
  - `packages/core/src/update/release.ts:62` 正则 `/SnowLuma\/SnowLuma\/releases\/tag\/([^/]+)/`
  - `packages/core/src/update/download.ts:42` `https://github.com/SnowLuma/SnowLuma/releases/download/v${version}/${name}`
  - `packages/core/src/update/release.ts:130` `api.github.com/repos/SnowLuma/SnowLuma/releases/...`
  - `DEFAULT_UPDATE_SOURCE` 镜像列表（`sources.ts`）
- 现状：`SnowLuma/SnowLuma` 已 404 ⇒ 检查更新必然失败；若用户开 `mode:'direct'` 还会撞墙。
- 处置：抽出一个 `RELEASE_REPO` 常量（owner/repo）替换 4 处；镜像列表按可控性重排；线上自建 release 仓库时注意 `release.json` 与资产命名 `SnowLuma-v<ver>-<platform>-lite.zip|tar.gz` 必须完全一致（更新器按名字拼 URL）。
- 相关：`check-node-version` 失败提示、EULA/PRIVACY 版本号也在更新器白名单里，改名会触发 `更新包清单无效`。

**P0-3 许可与分发边界**

- 现状：非 OSI 许可；禁止商用；**禁止公开发布修改版**；原生 addon 独立专有、不在许可内（`LICENSE` Section 5）。
- 处置（择一）：
  1. 仅内部/自用部署，不分发二进制与源码修改版 —— 成本最低，风险最低。
  2. 取得 SnowLumaDevs 书面授权（许可文件给了联系方式 `motricseven@foxmail.com`）。
  3. 自研原生层与协议解析（等于重写 `bridge` 的 native 部分 + `pipe_protocol`），法律上干净但工作量以月计。
- 注意：fork 到公开仓库、开 CI、发 Docker 镜像、npm 发包都属于"发布"范畴。

**P0-4 原生件供应链**

- 现状：仓库里只有二进制，无 `.c/.cpp/.rs/CMakeLists`；`native/auto-update-native-*` 分支由外部 `SnowLumaNative` 仓库的 Bot 推送（该仓库同样受 takedown 影响时无法更新）。
- 处置：
  1. 立刻归档当前 `packages/runtime/native/*` 与 `dist/native/*`（按版本 + sha256 清单），这是唯一可用的"已知可用基线"。
  2. 建立"原生件 ↔ QQ 版本"对照记录：拿一台测试机镜像 QQ 版本，记录注入是否成功、UIN 是否识别、心跳是否正常。
  3. 没有源码的前提下，能做的只有：版本指纹白名单/黑名单、失败即降级并给出可读错误、把原生失败暴露到 `/api/processes`。
  4. 长期：`SnowLumaNative` 若也下线，只能自研（见 P0-3 选项 3）。

**P0-5 推送与协作通道**

- 要求（不是现状描述）：维护必须有一个**可写**的远端，否则接手只能读不能改。
- 验收：`git push --dry-run origin HEAD` 退出码 0；`git fetch upstream --tags` 能拉到新 tag。**结果只记在本地**，不写进仓库。
- 拓扑若变化（换了 fork 归属、换了协议或入口），只改本地 `.git/config`；仓库里不放任何地址、账号、可达性结论。
- 需要脱离外部端点时用 `git bundle` 交换：`create --all` → `verify` → `clone`，全程不依赖网络通道。
- CI 里要访问 GitHub 的步骤（release 资产、npm 发布）属**部署侧**决策，不要在仓库文档里假设某条通路存在。

### P1 — 值得尽快修的确定性问题

**P1-1 同一个 `messages.db` 被两个可写连接打开**

- 位置：`packages/onebot/src/instance.ts:145-146` —— `MediaStore` 直接指向 `data/<uin>/messages.db`，`MessageStore` 拿到的是 `data/<uin>/messages.json`，但构造时 `dbPath.replace(/\.json$/, '.db')`（`packages/onebot/src/message-store.ts:69-71`）落到**同一个** `.db`，于是两者各自 `openSqliteDb()` 同一文件。
- 症状：WAL 下短时可见性差异、锁竞争（`busy_timeout 1000` 到点即报错）、迁移期间更危险。
- 处置：共享同一个 `SqliteDatabase` 门面（`packages/sqlite` 已支持 `serial()` 重入），或给 MediaStore 单独的库文件。

**P1-2 `message_id` 截断 + `ON CONFLICT DO UPDATE` 可能覆盖历史**

- 位置：`packages/onebot/src/message-id.ts`（sha1 → 前 4 字节 → 0 换 1）；`message-store.ts:71-110` 使用 `ON CONFLICT(message_hash) DO UPDATE`。
- 症状：int32 空间仅 2^32，历史长了必然碰撞；碰撞时旧消息被静默覆盖。
- 处置：主键改为 `(session_id, sequence, event_name)` 或加 `hash_collision_seq`；`message_id` 对外仍是 int32（契约不能动），但存储层不要用它当唯一键。

**P1-3 WS 服务端事件广播无背压上限**

- 位置：`packages/onebot/src/network/ws-server-connections.ts:130` 用 fire-and-forget `safeSend`，没有 `bufferedAmount` 上限（SSE 侧有 8 MiB）。
- 症状：慢客户端拖爆内存。
- 处置：加 `bufferedAmount` 阈值 + 丢弃/断开策略，和 SSE 保持一致的可见行为（丢帧要能观测）。

**P1-4 HTTP 传输默认放行裸 token 与 `?access_token=`**

- 位置：`packages/onebot/src/network/utils.ts:17-25`、`http-server-adapter.ts:410`（空 token = 关闭认证）。
- 症状：token 出现在 URL/日志里；空 token 时完全裸奔——默认 `config.host='127.0.0.1'` 尚算安全，但**配置里删掉 host 就会回落 `0.0.0.0`**（`http-server-adapter.ts:316,356`）。
- 处置：默认 `allowRawToken=false`；token 为空时**拒绝该传输启动**并给明确错误；把 host 回落值改成 `127.0.0.1`，别让"没写 host"等于"监听全网卡"。

**P1-5 更新替换的非原子窗口**

- 位置：`packages/core/src/update/files.ts:241-269`。
- 症状：Windows 上杀软/文件占用会让 `rename` 失败，留下 `AggregateError` 与半装目录，只能靠下次启动 `restore()` 自愈。
- 处置：短期加"更新前整目录快照 + 失败即自动回滚"；中期改成"版本目录 + 切换指针"（`current` 符号链接/清单），彻底消除半装状态。

**P1-6 `launcher.bat` 缺 `cd /d "%~dp0"`**

- 位置：`packages/runtime/launcher.bat`（7 行）。
- 症状：从开始菜单/快捷方式启动时 cwd 不对 ⇒ `config/`、`data/`、`logs/` 落到别处（配置看起来"丢失"）。
- 处置：补 `cd /d "%~dp0"`（与 `launcher.sh` 对齐），并在 `check-release-layout.mjs` 或测试里断言。

**P1-7 文档与配置残留漂移**

- `README.md` 引用 `CONTEXT.md`（不存在）、指向已消失的 `snowluma.github.io`。
- `compat/qq.json`（19 行，version 1 / updatedAt 2026-05-16 / `policy{allowUnknown:true,warnUnknown:true}` / 空 knownGood、knownBad / `minVersion:null`）**全仓无任何代码引用**：唯一消费者是外部项目 **SnowLumaDesktop**，它轮询 `raw.githubusercontent.com/SnowLuma/SnowLuma/main/compat/qq.json`（3 天缓存兜底）。该 URL 现在既解析不了、仓库也已消失 ⇒ 这个兼容清单事实上已经"断供"。处置：要么删掉并在 README 注明，要么改成自己仓库的 URL（注意别把"QQ 版本黑名单"这个安全能力一起丢掉——它原本用于挡已知会崩的客户端版本）。
  真正该补的是**本仓内的 QQ 版本校验**：目前 `bridge`/`core` 里一行版本检查都没有（见 §2.4 末）。
- `RoadMap.md` 提到 `better-sqlite3`，实际已换成 `@appthreat/sqlite3` 9.1.0。
- 处置：一次"文档对齐提交"，之后把"README 概览 + 包内 README 细节"的分工写进 `AGENTS.md` 已有条款即可。

### P2 — 结构性改进（有余力再做）

**P2-1 管道协议加版本协商**：hello 帧携带 `protocolVersion + abiVersion`，TS 侧在登录前断言，不匹配给可读错误，而不是 `bad frame header`。这是 native 与 TS 之间唯一的防错手段（`qq-hook-client.ts:17-44`）。

**P2-2 账号识别自检**：`login.db` 解密失败时不要只记日志，把"口令/算法/字段是否匹配"的结构化结果暴露到 `/api/processes`，并支持按 QQ 版本的 passphrase 表。避免 QQ 一升级就"静默无账号"。

**P2-3 更新包签名**：固定公钥 + 分离签名（minisign/cosign 皆可），校验顺序：签名 → sha256 → 解压。镜像永远只当传输层。

**P2-4 原生错误串去耦合**：`hook-manager.ts:655-657` 用正则匹配原生英文错误；改为原生返回结构化错误码（`{code, message}`），TS 只判 code。

**P2-5 WebUI 会话持久化**：把 session 落到 `config/webui.json`（已有快照机制），重启不再全员登出；顺带给 token 加轮换。

**P2-6 WebUI 后端拆分**：`server.ts` 已 2300+ 行，路由/中间件/SSE 混在一个文件；按 route 分文件但不改 HTTP 契约（前端 `packages/webui` 与 `INTEGRATIONS.md` 是契约）。

**P2-7 e2e 与真机矩阵**：现有测试极好（SQLCipher 加密侧重实现做 fixture、假管道做真实 40 字节帧、假 injector 等），但**没有一条测试覆盖"真 QQ + 真原生件"**；`qq-account-real-data.test.ts` 靠 `SNOWLUMA_QQ_DATA_ROOT` 手工触发。建议加一个"本地真机冒烟脚本"，记录 QQ 版本、原生件 sha256、注入结果。

**P2-8 版本/发布脚本收尾**：`tools/release.mjs` 发版后切回 `dev` 未实现；tag 需 push 两次（先 tag 后 push）；本地 clone 默认不带 tag（本次已 `--tags` 补齐）。

---

## 7. 风险登记

| # | 严重度 | 风险 | 位置 | 触发条件 | 影响 | 缓解 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 高 | 管道帧协议双份定义、无版本协商 | `packages/bridge/src/qq-hook-client.ts:17-44` | 原生件更新改头/标志位 | 全部会话静默失效，仅剩晦涩错误 | P2-1 |
| 2 | 高 | SQLCipher 口令/页格式硬编码 | `qq-account/login-db.ts:9`、`sqlcipher.ts:24-54` | QQ 客户端更新 | 账号识别全灭 ⇒ 无 UIN、无自动加载 | P2-2 |
| 3 | 高 | 更新无签名、镜像可配 | `update/release.ts`、`download.ts` | 上游/镜像被投毒 | 任意代码执行（更新即执行） | P2-3 |
| 4 | 中高 | Windows 替换非原子 | `update/files.ts:241-269` | 杀软/占用 | 半装目录，需下次启动自愈 | P1-5 |
| 5 | 中高 | 原生错误串参与控制流 + Electron/libc 启发式 | `hook-manager.ts:655-657,672-701` | glibc/QQ 改文案 | 自动加载重试变永久失败 | P2-4 |
| 6 | 中 | 同一 `messages.db` 两个写连接 | `onebot/src/instance.ts:145-146` | 高并发写 | 锁竞争/可见性异常 | P1-1 |
| 7 | 中 | `message_id` int32 截断 + upsert 覆盖 | `message-id.ts`、`message-store.ts:71-110` | 历史量增长 | 静默覆盖历史消息 | P1-2 |
| 8 | 中 | WS 广播无背压上限 | `ws-server-connections.ts:130` | 慢消费者 | 内存膨胀 | P1-3 |
| 9 | 中 | HTTP 裸 token/空 token 关认证 | `network/utils.ts:17-25`、`http-server-adapter.ts:410` | 误配置/公网暴露 | 未授权调用 | P1-4 |
| 10 | 中 | WebUI 会话在内存 | `webui/server.ts:125-144` | 每次重启 | 运维全部登出 | P2-5 |
| 11 | 中低 | 60s 连接预算共享 + 1.5s 轮询 | `hook-session.ts:27`、`pipe-watcher.ts` | QQ 良性重启 | UI 卡在 connecting 一分钟 | 缩短预算/把 tickNow 做成硬触发 |
| 12 | 中低 | Linux 账号目录 md5 链 | `qq-account/paths.ts:93-96` | QQ 改混淆方案 | Linux 静默 `pid: null`，Windows 正常 ⇒ 掩盖回归 | 加启动自检日志 |
| 13 | 低 | `getFileHoldersAsync` 缺失即全崩 | `database-holders.ts:20-22` | 原生件回滚 | 所有账号抛错而非降级 | 降级为 `pid: null` |
| 14 | 低 | SQLite musl/glibc 由"缺失 glibc 版本"推断 | `sqlite/src/layout.ts` | 非常规发行版 | 选错预编译 | 启动自检 + 明确错误 |
| 15 | 低 | 115 个 `extended` action 包裹不稳定 QQ 内部接口 | `packages/onebot/src/actions/extended.ts` | QQ 内部 API 改名 | 单点 action 失效 | 用 `readOnly`/文档标注稳定性等级 |
| 16 | 中高 | protobuf 无 `.proto` 主源、无 schema 版本，字段全靠逆向 | `packages/proto-defs/src/**`、`packages/protocol/tests/oidb-services/_pb-oracle.ts` | QQ 换 tag | 字段串位、静默空值，不报错 | 改定义必带 oracle 期望字节（§2.12） |
| 17 | 中 | 元素解码 fail-open，未知 tag 只打 trace | `msg-push/rich-body-decoder.ts:463` | QQ 新增元素 | 消息内容静默丢失 | 未知 tag 计数上指标 + 按版本统计 |
| 18 | 中 | 身份库迁移是 `PRAGMA table_info` + `ALTER`，无 `user_version` | `identity-service.ts:1206` | 后续加字段/改结构 | 老库缺列，运行期 SQL 失败 | 引入 `user_version` 迁移账本（对齐 messages.db 做法） |
| 19 | 中低 | sysmsg 去重是静态路由表 | `msg-push/sysmsg-dedup.ts:9-31` | QQ 改推送结构 | 重复系统消息或误去重 | 表与真实夹具同步（`tests/msg-push/fixtures/`） |
| 20 | 中低 | Highway 时序/阈值是逆向常数值 | `highway/highway-client.ts:31-49`、`upload-nodes.ts:33` | 上传策略变更 | 上传变慢或失败率上升 | 常量集中配置化 + 保留 `SNOWLUMA_HIGHWAY_UPLOAD_TIMEOUT_MS` |

---

## 8. 接手路线图

**第 1 周（把地基踩实）**

- [ ] 装 Node 24.22.1+，`pnpm test` 复跑，把 4 个 Windows 环境失败标成 `skipIf(win32)` 或改断言（不要假装通过）。
- [ ] 归档原生件基线：`packages/runtime/native/*` + `dist/native/*` + sha256 清单 + 对应 QQ 版本号。
- [ ] 决定许可路线（P0-3），据此决定仓库可见性与是否发镜像。
- [ ] 修 `launcher.bat` 的 `cd`（P1-6）；删/标 `compat/qq.json`（P1-7）。
- [ ] 抽 `RELEASE_REPO` 常量，更新链路改向（P0-2）。

**第 2 周（把流程跑通）**

- [ ] 在 fork 上跑通 `ci.yml`（先关掉需要 App secrets 的工作流，或改成 `workflow_dispatch` 手动）。
- [ ] 自建 release：手动打 tag → 生成 lite 包 → 验证 `check-release-layout` → 验证更新器能从自己的仓库检查/下载/校验。
- [ ] 建立"版本 ↔ QQ 版本"验证清单，跑一次真机注入冒烟并记录。

**第 3–4 周（修确定性问题）**

- [ ] P1-1 共享 SQLite 门面；P1-2 存储主键与 `message_id` 解耦；P1-3 WS 背压；P1-4 HTTP 认证收紧。
- [ ] 给每条修完的问题补回归测试（项目测试基建很好，照 `tests/helpers` 的既有风格写）。

**第 2 个月（结构性）**

- [ ] P2-1 管道协议版本协商；P2-2 账号识别自检；P2-3 更新签名。
- [ ] P2-5 WebUI 会话持久化；P2-6 后端拆分。
- [ ] 建立 `docs/` 拓扑：README（概览）→ 包 README（细节）→ 本手册（维护）三者分工固定。

**持续**

- 每次 QQ 大版本更新后：跑真机冒烟 → 记录原生件是否仍工作 → 更新 §7 风险表与 P0-4 清单。

---

## 9. 附录

### 9.1 抓一条真实链路（排障模板）

```bash
# 1) 看进程与管道（Windows）
powershell -c "Get-Process QQ | Select Id,ProcessName"
cmd //c dir "\\.\pipe\" | grep mojo

# 2) 看 SnowLuma 侧日志（默认 logs/）
tail -f logs/*.log

# 3) 看 WebUI 状态流
curl -N -H "x-snowluma-token: <token>" http://127.0.0.1:5099/api/state/stream

# 4) 看 OneBot 侧是否在线
curl http://127.0.0.1:3000/            # 裸 GET → {"online":true}
curl -H "Authorization: Bearer <token>" -X POST http://127.0.0.1:3000/ \
  -d '{"action":"get_login_info","params":{},"echo":"t1"}'
```

### 9.2 术语

| 术语 | 含义 |
| --- | --- |
| Hook / 注入 | 把原生 DLL 手动映射进 QQ 进程，挂钩 IPC |
| 管道帧 | SnowLuma 与注入 DLL 之间的 40 字节头二进制协议 |
| UIN / UID | QQ 号 / 新体系的内部用户 ID；`IdentityService` 负责映射 |
| Bridge | 一个 UIN 的运行实例（协议连接 + Api 层） |
| OneBotInstance | 一个 UIN 的 OneBot 传输/事件分发实例 |
| Action | OneBot v11 的可执行命令，本项目 230 个可执行名 |
| 迁移账本 | `messages.db` 里记录已应用迁移的表，避免重复迁移 |

### 9.3 本手册的维护

- 改动 §2、§6、§7 涉及的路径时，一并更新行号引用。
- 新增 P0 事项请同时写进 §8 路线图，避免清单与计划脱节。
- 本文件不替代 `AGENTS.md`（贡献规范）与各包 README（使用细节）：**规范看 `AGENTS.md`，用法看包 README，维护看这里。**
- **脱敏是新默认**：新增内容不写账号、地址、可达性结论、密钥路径、绝对路径、口令、版本号与机器指纹；只写「要验证什么」，不写「验证结果」。

### 9.4 环境私有信息去哪了

**任何环境信息都不进仓库。** 账号与仓库归属、地址与链路可达性、密钥路径、工具链版本、绝对路径、口令、机器指纹，一律记在工作区之外的私有笔记（或本仓库 `private/` 与 `*.local.md`，已被 `.gitignore` 忽略）。本文只保留「该验证什么」，不保留「验证结果是什么」。

> 分享仓库或截图前，先检查 `.git/config`、`config/*.json`、`logs/*`、`data/<uin>/`：这些在版本控制之外或被 ignore，但**手工打包整个目录**时会一起被带走。