# AgentOS 迁移清单

这是从当前 Android 原型迁移到系统 Agent 的最小顺序。每一步都应能单独验证，默认不构建完整 AOSP 镜像。

2026-09-23 重建基线：源码固定 `android-15.0.0_r34`。`62223a7` 及后续提交已保存 overlay、自动接线、probe 和备份脚本；自定义 Cuttlefish userdebug 镜像已经完成构建、启动、AgentOS health 和 Plugin 发现/握手验证。官方 stock Cuttlefish build `16373615` 仍作为独立 host 基线，stock 包不含 AgentOS。当前没有 Pixel 8 真机验证，也不能把完整 Plugin tool/resource 数据面或真实模型请求写成已完成。

## M0：仓库和契约

- [x] 统一目录：前端、系统 Agent、平台接入、Plugin、库和工具。
- [x] 写下 AgentManager、sideagentd 和输出事件流的职责。
- [x] 把 Agent Bus 和输出事件协议固化成版本化协议文档。
- [x] 定义任务状态、错误分类、取消和重连语义。
- [x] 冻结 Session 状态、调度、公平性、并发、恢复和 Plugin capability lease 契约。
- [x] 冻结 Session 自动选择契约：per-user 30 分钟活跃池、最近 20 个冷候选补足、254 + `new_session` Choice、Brief、Jev token 预算、分阶段选择和安全回退（[session-selection-v1](../system/agent/contracts/session-selection-v1.md)）。
- [x] 冻结 Plugin 运行时注入契约：manifest 发现、per-user 启用、按需绑定/解冻、握手、tool 调用、resource 的 system reminder 注入（[plugin-injection-v1](../system/agent/contracts/plugin-injection-v1.md)）。

## M1：sideagentd 骨架

- [x] 添加独立的 Session Store、Session Scheduler 和 Worker 参考实现（无 Binder、无真实模型）。
- [x] 添加 SessionSelector、Jev Choice HTTP adapter 和 `submitAutoInput` 参考实现；密钥只从未跟踪的运行时配置读取，真实设备 runtime 请求仍待单独验收。
- [~] 添加最小 daemon 可执行文件和健康接口（已随自定义 Cuttlefish 镜像编译并通过 health；正式数据目录和 runtime 请求仍待补齐）。
- [~] 添加 init service 描述和独立 SELinux domain（已在自定义镜像启动；neverallow、数据目录标签和完整 denial 审计仍待完成）。
- [x] 注册稳定 Binder 服务（`agentos.sideagentd` health service 已在 Cuttlefish service manager 中验证）。
- [x] 已实现并在自定义镜像执行 `dumpsys agentos`、`cmd agentos health|plugins|enable|disable` 基础诊断；Session/Task 诊断未实现。

## M2：AgentManagerService

- [~] 控制面已实现 manifest discovery、按用户持久化启用状态、绑定/握手和 health 查询；自定义 Cuttlefish 已验证编译、启动和基础路径，Binder death 恢复与完整 lease 仍待补齐。
- [ ] 处理 sideagentd 的 Binder death、重连和状态恢复。
- [~] 已实现 user start/stop/unlock、删除用户时清理 Plugin 记录和授予状态；Probe 已覆盖部分用户清理路径，Session/lease 生命周期及完整多用户矩阵待完成。
- [~] 现有控制 Binder 只允许 root/system UID，userdebug shell 可用诊断命令；系统签名前端的访问机制与迁移待完成。

## M3：输出管道和存储

- [x] 在 Scheduler 参考实现中验证事件日志、per-session sequence 和 Snapshot + afterSequence 恢复。
- [ ] 将参考实现的 Store 接入系统 daemon 的正式生命周期。
- [ ] 将 Session 自动选择接入稳定 Binder/AIDL，完成真实 Jev secret 注入、超时/回退诊断和多用户系统测试。
- [ ] 把 Task Store 从原型 JSON 文件迁移到 SQLite/WAL。
- [x] 在 sideagentd 参考数据面为副作用工具增加 SQLite `tool_operations`
  记录、幂等键、参数冲突检测和 `unknown` fence；Android native sideagentd
  的正式数据目录与恢复接线仍待完成。

## M4：前端迁移

- [~] `frontends/agenriod` 的主界面和 VoiceInteraction 已通过共享
  `AgentManagerClient` 调用 AgentManagerService；旧 AgentService 仍保留为
  测试兼容代码，未进入正常系统前端路径。
- [~] 新建 Session、提交输入、取消、snapshot 和事件订阅已移到
  `AgentManagerService → sideagentd`；native runtime 的持久化和模型执行仍待
  正式 runtime 包接入。
- [x] Compose、VoiceInteraction 和通知仍作为前端入口；默认 Assistant 和
  长按电源键的产品资源 overlay 已加入 Cuttlefish/Pixel 8 接线。
- [ ] 用一个最小非 Agenriod 前端验证多前端订阅。

## M5：Plugin Broker

按 [plugin-injection-v1](../system/agent/contracts/plugin-injection-v1.md) 实现；AOSP 侧条目见 [AOSP 变更全局 TODO](../platform/aosp-integration/aosp-todo.md)。

- [x] daemon 参考实现 `PluginBroker`：descriptor v3 校验、启用状态、按需绑定/握手、invoke 与 resource/reminder 管道、lease 接线；契约 §13 reference tests 1–12 通过。

- [x] system_server 已实现并在 Probe 上验证包名、UID、签名、版本校验。
- [~] manifest 发现、`BIND_AGENT_PLUGIN` 权限接线与 per-user 启用状态持久化已实现；安装、启用、禁用、删除用户和重绑定已验证，升级与重启矩阵待补齐。
- [~] `BIND_AUTO_CREATE` 绑定、禁用时 unbind 和断连重试已实现并通过部分 Cuttlefish 生命周期测试；按调用需求绑定/空闲释放及 phantom process killer 测试待完成。
- [~] stable AIDL v1 `openPluginSession` 和基础 descriptor 校验已实现；
  `agentos_system_aidl` V2 已冻结握手、capability grant、异步 tool/resource
  callback 和 cancel 的协议骨架，system_server → sideagentd handoff、
  descriptor v3 policy 与运行时调用仍未移植。
- [ ] 修复同步握手无法取消的问题：两个不返回的 Plugin 可耗尽两个握手工作线程；使用可隔离或异步的握手机制并验证后续 Plugin 可恢复。
- [~] 已增加 `AgentPluginSession` 稳定 parcelable，并由
  `AgentManagerService` 注册到 native `sideagentd`；设备编译、Binder death
  清理和正式 lease handoff 仍待验证。
- [~] Node sideagentd 参考实现已覆盖 requestId/幂等键、取消、deadline 和
  `tool_operations` unknown fence；Android sideagentd 的正式调用管道仍未接线。
- [ ] resource 读取与 system reminder 注入管道：turn boundary 拉取、预算与确定性截断、非重放诊断记录。
- [~] Plugin 断连会清理绑定与 session 并退避重试；完整 capability 撤销仍待移植与 Binder death 系统测试。
- [ ] 将 MCP transport、tool 调用、resource 注入和模型运行时从参考实现移植到系统数据面，完成真实端到端验证。
- [ ] 将本地 shell manifest 和推送注册限定为开发模式。

## M6：平台集成

AOSP 侧改动以 [AOSP 变更全局 TODO](../platform/aosp-integration/aosp-todo.md) 为唯一事实来源。

- [x] 固定 `android-15.0.0_r34` 并保存 Cuttlefish-only 自动接线、真实 `AgentOsPluginProbe` 测试源码与接线 fixture tests（`62223a7`、`bfde761`、`1c163f3`）。
- [x] 保存仓库外增量备份工具及校验记录机制（`bfde761`、`90bd0a1`、`941f125`）；运行中的构建仍需持续产出和备份证据。
- [x] 官方 stock Cuttlefish build `16373615` image/host 本地校验、启动和 ADB 已完成；它只证明 host 能运行 stock 系统。
- [x] 在固定源码基线完成 Soong 分析、自定义 Cuttlefish userdebug 构建，并启动 sideagentd 与 `agentos` 服务。
- [~] 系统级 Binder、SELinux 和多用户测试已覆盖 health、Plugin 发现/握手及部分生命周期；完整 tool/resource、lease、恢复和多用户边界仍在补齐。
- [ ] 将平台镜像构建放到手动 workflow，不放进默认 PR CI。
- [ ] 按全局 TODO 的销毁前检查确认自定义镜像、校验、resolved manifest、patch、日志均有本地副本且最新代码已推送，再推进 Pixel 8 真机路线。
