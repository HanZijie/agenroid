# AgentOS 测试计划

版本：v1.0
编制日期：2026-09-21
适用仓库：`agenriod/`，以及工作区中的 `pi-acp-adapter/`

## 1. 测试对象和当前结论

AgentOS 现在同时包含三类代码，测试结论必须分开记录：

| 范围 | 当前事实 | 本计划的结论口径 |
| --- | --- | --- |
| `frontends/agenriod` | 可编译的 Android Compose 原型，包含 Agent Service、Session Store、MCP、Plugin 和语音入口 | 测试通过只代表 Android 原型行为通过 |
| `system/agent/daemon`、`runtime` | 可在 Node.js 运行的 Scheduler、SQLite Store、Worker、Plugin Broker 和 Pi Worker 参考实现 | 可做离线契约、恢复和故障测试 |
| `platform/`、真正的 `sideagentd`、`system_server`、init、SELinux、AOSP | 已有 `android-15.0.0_r34` Cuttlefish userdebug 镜像构建、启动、health 和 Plugin 发现/握手证据；完整 runtime/tool/resource 和 Pixel 8 仍未完成 | 可以执行 Cuttlefish 镜像与 Demo 烟测，但不能把局部证据写成完整系统级验收 |

Pi ACP 适配器是工作区旁边的独立 npm 包。它有自己的 TypeScript 构建和测试命令，不能因为 AgentOS Runtime 测试通过就视为适配器已经验证。

本计划的目标是验证以下事实：输入顺序和幂等语义正确，事件可按序恢复，取消、超时、进程死亡和重启不会偷偷重放未知副作用，Plugin 权限只能收窄，前端断线不影响任务，敏感信息不进入事件、目录或模型输入，以及 Android 原型的关键用户流程可用。

## 2. 范围

### 2.1 本阶段纳入

- 架构目录、文档链接、Gradle 模块引用、生成资源和 shell 语法。
- Runtime JavaScript、ACP 事件转换、Pi Worker。
- Session Store、Session Scheduler、Process Worker、优先级、公平性、并发、取消、deadline、恢复、事件序列、snapshot 和订阅缓冲。
- Plugin Broker 的 descriptor、身份、per-user enablement、按需绑定、lease、tool 调用、幂等键、超时、死亡撤销和 resource reminder。
- Android JVM 单元测试、Compose/Service/MCP/Plugin 设备测试、Notes 跨 App 流程。
- Pi ACP adapter 的类型检查、构建和 ACP v1 行为。
- 安全、可靠性、性能和可观测性的测试设计；其中部分要等系统组件落地后执行。

### 2.2 当前排除或延期

- 完整 `system_server` capability lease、tool/resource 数据面、恢复语义以及
  sideagentd 的正式生产数据目录策略。
- Pixel 8 真机、干净主机复现、完整 Cuttlefish 多用户系统权限矩阵和真实 native runtime/tool/resource 行为。
- 真实第三方模型、真实外部副作用工具和真实用户凭据的自动化测试。
- 还没有实现的 ACP session load/resume、MCP/Plugin/Hooks 公共配置和 terminal/filesystem delegation。测试应验证它们被明确拒绝，而不是假设已经支持。

## 3. 测试策略和测试层级

测试按由快到慢、由确定到真实的顺序执行。每一层都要保留命令、提交号、环境和日志。

| 层级 | 目标 | 主要工具 | 合入要求 |
| --- | --- | --- | --- |
| L0 静态检查 | 防止目录、引用、生成文件和 shell 回归 | `check-architecture.sh`、`bash -n`、`git diff --check` | 必须通过 |
| L1 单元/契约 | 验证纯逻辑、协议边界、错误分类和安全约束 | Node `node:test`、JUnit、Kotlin 测试 | 受影响组件 100% 通过 |
| L2 组件集成 | 验证 Store + Scheduler + Worker、Broker + lease、ACP + Pi | Node 子进程、SQLite 临时库、fake worker、Pi session fixture | 关键契约全部通过 |
| L3 Android 设备 | 验证 Service、Compose、MCP HTTP、跨 App Binder 和进程恢复 | Pixel_8a AVD、AndroidJUnit4、Compose test、ADB | 关键流程全部通过；无崩溃/ANR |
| L4 系统级（待实现） | 验证 init、Binder、system_server、SELinux、多用户和系统恢复 | Cuttlefish userdebug、dumpsys、`cmd agent`、CTS/自定义测试 | M1-M6 完成后才可执行 |

真实模型只用于少量人工或受控验收；自动化测试使用本地模型 HTTP fixture、MCP SDK fixture 和 fake worker，确保结果可重复且不发送真实请求。

## 4. 环境矩阵

| 环境 | 配置和入口 | 用途 | 目前状态 |
| --- | --- | --- | --- |
| Host/Node | Node.js 22 及以上；`runtime/package-lock.json` | Runtime、Scheduler、Plugin Broker、Pi Worker | 可执行 |
| Pi ACP 包 | Node.js `>=22.19.0`；`pi-acp-adapter/package-lock.json` | TypeScript ACP 适配器 | 可执行 |
| Android Host | Android Studio JDK、Android SDK platform 37/build tools 36、Gradle wrapper | JVM 单元、Lint、APK/Plugin 构建 | 可执行 |
| Android Emulator | 默认 AVD `Pixel_8a`，minSdk 30，target/compile 37 | 设备和跨进程测试 | 本次检查未发现在线设备 |
| 本地服务 fixture | `runtime/mcp-sdk-fixture.mjs`、测试内 `ServerSocket` | OpenAI-compatible/Anthropic/MCP 响应、超时和错误 | 随设备测试启动 |
| AOSP/Cuttlefish | `android-15.0.0_r34` userdebug 镜像、专用 UID、SELinux policy、Plugin probe | 镜像、health、发现/握手和部分生命周期准入 | Cuttlefish 路径可执行；完整 runtime、Pixel 8 和干净主机复现待补 |

测试数据必须使用临时目录和临时 SQLite/WAL 文件。API key 只允许通过 `.env.anthropic.local` 或测试内存配置进入；不得提交、打印或写入事件和截图。任何外部副作用测试都使用幂等测试键和可清理的 fake endpoint。

## 5. 可直接执行的测试清单

所有 Android 命令从 `agenriod/` 目录执行；Android Gradle 命令通过 `tools/android/android.sh`，不要直接调用 `./gradlew`。

### 5.1 L0 架构和仓库检查

```bash
./tools/ci/check-architecture.sh
bash -n tools/android/android.sh tools/ci/check-architecture.sh
git diff --check
```

检查必需文档、模块引用、相对链接、AOSP checkout 和生成文件是否误提交，以及 Gradle 脚本和 shell 是否可解析。

### 5.2 Runtime、Scheduler、Plugin Broker

```bash
npm ci --prefix runtime --no-audit --no-fund
node runtime/build.mjs
npm run test:acp --prefix runtime
npm run test:scheduler --prefix runtime
node runtime/plugin-contract-test.mjs
node runtime/runtime-plugin-test.mjs
node tools/android/anthropic-env-test.mjs
git diff --exit-code -- frontends/agenriod/src/main/assets/agenriod-agent.js
```

其中 `test:acp` 覆盖 ACP v1 初始化、未实现能力不宣告、prompt 内容转换、工具授权、取消和关闭；`test:scheduler` 覆盖 31 个调度/Worker/Pi Worker 用例；两个 Plugin 脚本覆盖 Node Plugin 契约、MCP 工具命名、Plugin 撤回和 MCP 错误；Anthropic 环境测试覆盖解析、边界和安全 staging。

### 5.3 Pi ACP adapter

```bash
npm ci --prefix pi-acp-adapter --no-audit --no-fund
npm run check --prefix pi-acp-adapter
npm run build --prefix pi-acp-adapter
```

`check` 必须同时通过 TypeScript 类型检查、测试配置类型检查和 4 个 ACP 行为测试；`build` 必须生成可执行的 `dist/src/index.js`。

### 5.4 Android Host 检查

```bash
./tools/android/android.sh doctor
./tools/android/android.sh gradle \
  :frontends:agenriod:testDebugUnitTest \
  :frontends:agenriod:lintDebug \
  :plugins:notes:assembleDebug
```

这一步覆盖 Android JVM 单元测试、Lint、主 App 和 Notes Plugin 的 debug 构建。当前 JVM 用例包括 Composer 快捷输入和基线 JUnit；设备测试不由该命令执行。

### 5.5 Android 设备和端到端测试

```bash
./tools/android/android.sh start
./tools/android/android.sh instrumentation
./tools/android/android.sh logs -d
./tools/android/android.sh crashes
./tools/android/android.sh screenshot
./tools/android/android.sh ui
```

`instrumentation` 会构建并安装 Notes Plugin，启动本地 MCP SDK fixture，使用 `connectedDebugAndroidTest` 执行设备测试。需要真实 Anthropic 测试时，先将 `.env.anthropic.local` 配置好，再由脚本导入加密存储；默认测试不需要真实模型。

## 6. 核心测试设计

### 6.1 Session、Store、Scheduler 和 Worker

每次改动调度或存储协议时，至少执行下列场景：

| ID | 场景 | 预期结果 |
| --- | --- | --- |
| SCH-01 | 同一 Session 连续提交多个输入 | 严格 FIFO；一个 Session 同时只有一个 running Task |
| SCH-02 | 多 Session 并发、per-user cap、全局 cap | 不超过配置；释放槽位后按优先级、公平性和 aging 调度 |
| SCH-03 | 相同 `clientRequestId` 重复提交；等价 JSON 键顺序不同 | 返回同一任务；冲突 payload 被拒绝；不产生第二次执行 |
| SCH-04 | queued、running、cancelling、paused、completed、failed 的合法和非法转换 | 合法转换产生对应事件；非法转换返回分类错误且状态不变 |
| SCH-05 | queued cancel、running cancel、cancel grace 超时 | 已确认停止才释放运行槽位；无确认进入 `unknown`/恢复路径，不伪造成功 |
| SCH-06 | queue/execution deadline、晚到的 worker 事件 | 到期触发取消；晚到事件被 generation fence 丢弃 |
| SCH-07 | worker 在启动中、运行中崩溃，daemon 在 running/cancelling 时重启 | 保留队列和 checkpoint；未知 Attempt 不自动重放；需要恢复控制面处理 |
| SCH-08 | Snapshot + `afterSequence` 重连，过旧 cursor，慢读端 buffer overflow | 事件顺序稳定；过旧 cursor 返回 `cursor_too_old`；慢订阅不阻塞 Agent，前端用 snapshot 补齐 |
| SCH-09 | SQLite WAL 事务回滚、独占 owner、单调 sequence | 失败事务无半写状态；sequence 不重复、不倒退；第二 owner 被拒绝 |
| SCH-10 | capability lease 绑定 Session 和 worker generation，lease 过期/撤销 | 新调用被拒；活动 Task 按取消流程结束；旧凭据不能跨 generation 使用 |

Session 自动选择在 [`session-selection-v1.md`](../system/agent/contracts/session-selection-v1.md)
中单独冻结，reference tests 还必须覆盖：

| ID | 场景 | 预期结果 |
| --- | --- | --- |
| SEL-01 | 30 分钟边界、per-user 隔离、254 个已有 Session、最近候选补足 | active pool 严格移除过期项；选择请求最多 254 个已有候选，不足 20 个时补足 `stale` 冷候选，并始终追加 `new_session` |
| SEL-02 | 首轮 query、AI final answer、最近两轮 query/answer | Brief 字段顺序稳定；输入 token 上界生效；Brief 只作为不可信证据 |
| SEL-03 | Jev 返回已有 id、`new_session`、非法 id 或超时；长请求分阶段 | 精确 id 才能路由；固定入口创建新 Session；超预算时批量筛选后再决赛；错误安全回退且不跨用户写入 |
| SEL-04 | API key、错误响应和诊断输出 | key 不进入 request body 以外的持久化数据、事件或日志；错误不回显 secret |

### 6.2 Plugin Broker 和 resource reminder

| ID | 场景 | 预期结果 |
| --- | --- | --- |
| PLG-01 | 协议版本、descriptor schema、`pluginId`、权限、大小和数量非法 | 握手拒绝并分类；系统身份来自 UID/包/签名，不信任 JSON 自称 |
| PLG-02 | 用户未启用、启用后按需 bind、空闲 unbind 后再次使用 | 未启用不 bind；按需拉起；重连产生新 `pluginSessionId` 和新 lease |
| PLG-03 | Binder death、包更新、签名变化、用户禁用 | 所有关联 lease 撤销；新调用失败；in-flight 结果不自动重放 |
| PLG-04 | 重复 `requestId`、external tool 缺少/携带幂等键 | 同一调用去重；缺 key 拒绝；未知终态不自动重试副作用 |
| PLG-05 | deadline、cancel、超时和 attachment 上限 | 绝对 deadline 包含拉起时间；取消是尽力而为；超出内联上限走受控 attachment |
| PLG-06 | `per_turn`、`on_change`、失败/超时、预算和确定性截断 | 只在 turn boundary 拉取；失败省略且记诊断；顺序和预算可复现 |
| PLG-07 | resource 内容带有“忽略系统规则”等指令样文本 | 只作为标注来源的不可信数据进入当轮模型输入，不改变 grant、授权或工具策略 |
| PLG-08 | credentials、Authorization header、完整 descriptor 和 resource 内容查询诊断/事件 | 敏感值不出现在 catalog、事件、snapshot、日志和 prompt；只记录必要元数据 |

### 6.3 Runtime、ACP 和 MCP

- ACP `initialize` 只报告已实现能力；`loadSession`、MCP、terminal 和 filesystem delegation 等未实现功能必须显式拒绝。
- 文本、图片、resource link、embedded text 的输入映射必须保留语义；assistant/thought、tool start/update/end 的事件顺序必须稳定。
- `session/request_permission` 的 allow/reject 结果必须映射为工具成功或错误，不能绕过授权。
- Pi Worker 成功后才保存 checkpoint；重启后上下文可恢复，未确认的 Attempt 不重放。
- MCP Streamable HTTP 覆盖握手、分页、SSE、过期 session、ID 不匹配、服务器错误隐藏和 in-flight close。
- Plugin tool 名称发生冲突时必须稳定改名且不超过模型限制；Plugin 失活后下一轮不得继续暴露旧工具。

### 6.4 Android 原型

设备测试按以下顺序执行，每项失败都要附 logcat、UI XML 或截图：

1. 主页面启动、发送、流式回复、停止和错误提示；Composer 快捷输入、中文 IME 组合输入和按钮状态。
2. 新建/切换/删除 Session；Activity 重建、Agent 进程重启后任务队列、消息和 checkpoint 保持正确。
3. Settings 保存 provider、base URL、model、token 和 hooks；重启后可读，明文 SharedPreferences、日志和事件中不出现 token；非法配置被拒绝。
4. SAF attachment 使用 `content://` URI、持久权限和大小上限；文件路径 URI、超限和不可读资源被拒绝。
5. MCP 设置、Streamable HTTP 工具调用、分页、错误、过期 session 和关闭；服务器错误文本不直接泄露给用户或 prompt。
6. Notes Plugin 未启动时不出现在有效 catalog；启动后可发现和调用；Host 重启不杀 Plugin；停止 Plugin 后能力撤销。
7. 语音入口在设备不支持、无匹配、超时和正常识别时都给出可恢复状态；权限拒绝不导致崩溃。
8. Compose 关键控件可通过 test tag/语义树定位；键盘、旋转和较小屏幕不遮挡发送和停止操作。

当前已有设备用例包括 SessionStore、CompactComposer、Anthropic 配置、MCP HTTP/SDK、AgentService 生命周期、AgentTaskQueue、Notes 跨 App 和设置页；新增功能必须把用例挂到对应契约或用户流程下，不能只增加“能打开页面”的冒烟测试。

## 7. 非功能测试

### 7.1 可靠性和恢复

- 连续提交、取消、重连和重启循环至少 100 次，检查 sequence、任务状态、checkpoint 和 SQLite 文件一致性。
- 模拟 worker 卡死、快速退出、半写响应、Binder death、MCP 断开和磁盘写失败；确认不会重复外部副作用。
- 订阅端分别以快读、慢读、断开重连和过旧 cursor 运行，记录 buffer overflow、恢复延迟和 Agent 是否继续执行。

### 7.2 性能基线

当前仓库没有正式的性能阈值和基准报告，因此第一轮先采集基线：冷启动/热启动、首个事件、首 token、任务吞吐、SQLite commit、snapshot、重连恢复和 Plugin bind/handshake 延迟。记录 p50/p95/p99、峰值 RSS、数据库大小和 CPU；基线稳定后再设版本门槛。禁止以“本机一次运行很快”替代基准。

### 7.3 安全和隐私

- 未授权 UID、未签名包、未保护 service、错误 userId、伪造 pluginId、过期 lease 和未知 tool 全部拒绝。
- 检查命令工具、文件 Broker、MCP header、模型 key、Plugin result、resource reminder 和错误文本的注入边界。
- 检查日志、`dumpsys`、snapshot、事件流、SharedPreferences、APK 资源和构建产物，不得包含凭据、Authorization header 或完整敏感 payload。
- AOSP 阶段增加 SELinux allow/deny、Binder 调用身份、多用户隔离、包更新/签名变化、用户停止/删除和系统恢复测试。

## 8. 通过标准和发布门槛

### 8.0 镜像内置前端验收

先构建 `:frontends:agenriod:assembleDebug` 和 `:plugins:notes:assembleDebug`，再把两个 APK
通过 `wire-platform.py --frontend-apk ... --notes-apk ... --apply` 接入目标 AOSP
产品。刷入镜像后必须确认：

- `com.example.agenriod` 位于产品分区并使用 platform signer；
- `cmd role get-role-holders android.app.role.ASSISTANT --user 0` 返回 Agenriod；
- `settings get secure assistant` 指向 Agenriod 的 VoiceInteraction service；
- 长按电源键打开 VoiceInteraction session，Compose surface 出现并开始监听；
- `cmd agentos plugins --user 0` 发现 Notes，启用后通过系统 endpoint 完成握手；
- `agentos_demo_alarm`、`agentos_demo_calendar` 和 `agentos_demo_meeting_records`
  已安装；meeting-records endpoint 按包身份完成发现；
- Agent session 的输入、事件订阅、snapshot 和取消经由 AgentManagerService 到达 sideagentd。

native sideagentd 已接入 MiniMax-M3 HTTPS Worker、Jev 自动 Session 选择、
受保护 secret 文件和重启恢复 fencing。完整验收必须在匹配的 AOSP 镜像上运行
`python3 tools/aosp/test-agentos-runtime.py --output-dir <new-dir>`；在该脚本
返回 PASS 前，代码编译通过不能算真实系统验收。

### 8.1 Pull Request 门槛

- L0 全部通过，`git diff --check` 无输出，生成的 `agenriod-agent.js` 无未提交差异。
- 受影响的 Node、Pi ACP、Gradle 单元和 Lint 检查全部通过。
- 契约类用例必须 100% 通过；不得用跳过测试掩盖未实现行为。
- P0/P1 缺陷为 0；P2 只有在有明确风险接受记录时才能合入。
- 不引入真实 key、AOSP checkout、`build`/`node_modules` 或本地模型配置。

### 8.2 Android 验收门槛

- `connectedDebugAndroidTest` 关键用例全部通过，连续两次运行结果一致。
- 主发送、停止、Session 恢复、MCP、Plugin 和设置加密流程均通过；无崩溃、ANR、明显泄露或不可恢复卡死。
- 失败必须有可复现步骤和设备/镜像信息；单次偶然通过不算关闭缺陷。

### 8.3 系统版本验收门槛（未来）

- Cuttlefish userdebug 能启动 `sideagentd`，健康检查、`dumpsys agent`、Binder 服务发现和重启恢复通过。
- system_server 只处理控制面；模型循环、网络、QuickJS 和第三方 Plugin 不进入 system_server。
- 多用户、签名/权限、SELinux、Plugin freezer/phantom killer、事件恢复和前端断线全部通过。
- 在达到这些条件前，发布说明必须写“Android 原型/参考实现通过”，不能写“AgentOS 系统服务已验收”。

## 9. 缺陷等级和记录格式

| 等级 | 例子 | 处理要求 |
| --- | --- | --- |
| P0 | 凭据泄露、越权、重复不可逆副作用、数据损坏、系统无法启动 | 立即阻断合入/发布 |
| P1 | 主流程无法发送或恢复、任务永久丢失、关键进程崩溃、取消语义错误 | 修复后才能进入下一阶段 |
| P2 | 边界输入错误、特定设备/网络失败、诊断缺失、偶发 flaky | 记录影响，可有条件合入 |
| P3 | 文案、布局或非关键可访问性问题 | 排期修复 |

每个缺陷至少记录：提交号、测试层级、设备/Node/Java 版本、命令、输入数据、期望和实际结果、日志/事件序列/snapshot、是否可重复、影响范围和修复后的回归用例。

## 10. CI 和执行节奏

当前 CI 已有两个事实：轻量 CI 按变更条件运行架构检查、Runtime 契约测试；Android workflow 运行 Gradle JVM 单元、Lint 和 Notes 构建，但默认不启动模拟器、不构建 AOSP 镜像。`pi-acp-adapter` 目前不在这两个 workflow 的自动检查范围内，应补充独立 workflow 或把其 `npm run check` 纳入工作区 CI。

建议节奏：

1. 每次提交：L0、受影响的 L1/L2、生成资源检查。
2. 每个 Pull Request：完整 Runtime、Pi ACP、Gradle unit/Lint/build；代码触及 Android 设备路径时追加 Pixel_8a instrumentation。
3. 每晚或发布候选：设备测试重复运行、进程杀活/重连循环、性能基线和敏感信息扫描。
4. M1-M6 每个迁移阶段：新增对应的 Cuttlefish/system test，阶段未通过时不把设计文档状态改为已实现。

## 11. 当前基线记录

执行日期：2026-09-21；提交：`ce419af6ecbb944bc244aabad1408b4a764a88bb`。

已通过：

- `./tools/ci/check-architecture.sh`。
- Runtime ACP：4/4；Scheduler/Worker/Pi Worker：31/31。
- Node Plugin contract、Runtime Plugin lifecycle、Anthropic environment：全部通过。
- `npm run check --prefix pi-acp-adapter`：类型检查和 4/4 测试通过。
- `npm run build --prefix pi-acp-adapter`：通过。
- Android `testDebugUnitTest`、`lintDebug`、Notes `assembleDebug`：Gradle 构建成功。
- MiniMax-M3 真实 API 探针：HTTP 200，响应 `TEST_OK`，耗时约 1.4 秒；请求在子进程环境中加载现有本地配置，key 未写入日志。
- Android `connectedDebugAndroidTest`：Pixel_8a 模拟器上 21/21 通过，0 failures，0 errors；包含 Service 生命周期、任务恢复、MCP 和 Notes Plugin 跨 App 测试。
- Android App 真实模型链路：导入本地配置后 UI 显示 `MiniMax-M3`，发送唯一测试串并收到 `VERIFIED7F4`，状态回到 `Ready`；结果截图保存在 `build/codex/minimax-real-result.png`。


平台镜像的后续证据已记录在
[`platform/aosp-integration/cuttlefish-host.md`](../platform/aosp-integration/cuttlefish-host.md)
和 [`platform/aosp-integration/aosp-todo.md`](../platform/aosp-integration/aosp-todo.md)：
自定义 Cuttlefish 镜像已构建并启动，sideagentd、Binder service、health 和
Plugin probe 生命周期已验证；App 级 `NotesPluginCrossAppTest` 也在 stock
Cuttlefish 上通过。native MiniMax/Jev 真实请求、完整重启恢复、tool/resource
调用和 Pixel 8 真机仍需按各自 runbook 单独验收。

这份基线只证明上述命令在当前工作树和当前主机上通过；它不替代设备测试、系统镜像测试或真实部署验收。

## 12. 后续优先级

1. 将 `pi-acp-adapter` 的 `npm run check` 加入 CI，并统一 Node 版本矩阵。
2. 为 Android instrumentation 准备稳定的 Pixel_8a/CI 模拟器，保存 JUnit、logcat、截图和 UI XML 产物。
3. 补充 File Broker、语音、SAF、权限拒绝、可访问性和错误恢复的设备断言。
4. 为 Scheduler/Plugin Broker 增加长时间运行、随机故障注入、性能和敏感信息扫描。
5. 随 M1-M6 实现，把本计划中的 system-level TODO 转为可执行的 Cuttlefish 测试套件，并将每个测试映射到版本化 AIDL/契约条款。
