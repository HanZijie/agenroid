# AOSP integration

这里记录将 AgentOS 接入 AOSP 的位置和约束。源码构建基线固定为 **`android-15.0.0_r34`**，先验证 Cuttlefish，再处理 Pixel 8（`shiba`）。AOSP checkout、工具链缓存和镜像放在仓库外；只提交 overlay、接线/备份脚本和可重现的版本记录。

```text
system/agent/                         系统契约与 Node 数据面参考实现
platform/aosp-integration/overlay/    待集成的 native daemon、Java 控制面、AIDL、init 和 domain
platform/framework/agent-manager/     AgentManagerService 设计
platform/product/                     产品包、权限和 system app 配置
tools/aosp/                           接线、备份及其测试
```

所有 AOSP 侧改动（sideagentd 落地、SELinux、AgentManagerService、Plugin 权限、freezer 与 phantom process killer 调优、系统测试）以 [AOSP 变更全局 TODO](aosp-todo.md) 为唯一事实来源，按状态和验证证据维护。

默认 PR CI 检查 overlay 结构与已有运行时契约，不构建 AOSP。完整平台构建和系统验证仍需单独执行，专用平台 workflow 待添加。需要实际打包并验证 Demo 时，按 [`AOSP 镜像打包与 Demo 验证`](../../docs/aosp-image-demo-quickstart.md) 操作。

## 当前证据（2026-09-23）

- `62223a7` 已保存自动接线和真实测试 APK `AgentOsPluginProbe`；后续 `bfde761`、`1c163f3` 等提交补充了 Cuttlefish-only 接线、版本校验、仓库外备份及脚本测试。
- 新主机的 SSH 指纹、两块数据盘挂载、CPU/内存、`/dev/kvm` 与主机 JDK 已记录。frameworks/base、Go、clang 的 archive/tree 核对已通过；本轮自定义镜像构建使用了已保存的 r34 checkout 和临时构建兼容修正，完整 resolved manifest、干净主机复现和第二份独立镜像仍待补齐。
- 官方 Cuttlefish **stock build `16373615`** 的 image zip 与 host package 已下载到本地并通过传输前后及本地 SHA-256 一致性校验。证据目录为仓库外的 `../.local/aosp-artifacts/2026-09-22-rebuild/fallback/`，其中 `index.json` 和 `SHA256SUMS` 记录校验结果。
- 19:17 已验证 stock Cuttlefish 通过 QEMU 启动：ADB `127.0.0.1:6520` 为 `device`，`sys.boot_completed=1`，系统 build ID 为 `CP2A.260605.016`。19:19 本地备份 index 记录 93 个已校验文件、0 error、0 pending。环境、恢复参数和日志采集见 [stock Cuttlefish runbook](stock-cuttlefish.md)。
- `android-15.0.0_r34` 的 AgentOS 自定义 Cuttlefish 镜像已经完成构建和启动验证；证据中的 `m droid -j96` 返回 `BUILD_RC=0`，自定义实例 `127.0.0.1:6521` 达到 `sys.boot_completed=1`，并通过 `sideagentd` UID、Binder 服务、health 和 Plugin 发现/握手检查。stock 包不含本仓库 overlay，不能替代这份自定义镜像证据。
- 三个 `demo-apps` APK 已提供独立构建和镜像 staging 参数；已有 `NotesPluginCrossAppTest` 在 stock Cuttlefish 上通过。该 App 级结果与自定义系统服务验证的范围不同，完整 Plugin tool/resource 数据面和带真实凭据的 native runtime 仍需单独验收。Pixel 8 真机和干净主机从零复现也尚未完成。

## Overlay 与自动接线

`overlay/` 已包含 stable AIDL、native `sideagentd` runtime Worker、MiniMax/Jev
请求、受保护 secret 文件、持久化事件和重启恢复边界，以及
`AgentManagerService`。控制面实现了 manifest 发现、包身份校验、按用户保存启用状态、用户生命周期、绑定/握手、断连重试，以及 `cmd agentos`、`runtime-test` 和 `dumpsys agentos` 的基础诊断。代码存在不代表这些行为已在 Android 系统上验证。

在完成所选项目同步并保存 checkout 状态后，从仓库根目录执行：

```bash
python3 tools/aosp/wire-platform.py /path/to/aosp
python3 tools/aosp/wire-platform.py /path/to/aosp --apply
```

默认 target 是 `cuttlefish`，不要求同步 Pixel 的 shusky 项目；Pixel 路线需显式 `--target pixel8`。脚本检查 manifest 默认 revision 及参与接线项目的 HEAD 是否对应固定 tag，复制 overlay，并修改 AID、产品包、SystemServer、Java 依赖、权限与平台 SELinux 接线。它不会拒绝所有工作区改动；先检查 dry-run 差异。旧文件保存在 **AOSP 根目录的同级** `agentos-wiring-backups/<timestamp>/`，避免 Soong 扫描备份中的重复模块。重复应用应无变更。

`prepare-overlay.sh` 仍可用于仅复制 overlay，不执行平台接线。接线行为测试使用临时 Git fixture：`python3 tools/aosp/test_wire_platform.py`；它不能替代真实 AOSP 编译或启动验证。

## 构建与备份顺序

1. 只同步 Cuttlefish 所需项目，不执行 `repo sync -g all`。保存 `repo manifest -r`、工具链版本、磁盘记录和 patch；原始 `upstream-default.xml` 不能代替 resolved manifest。
2. 应用接线后，先通过 Soong 分析，再构建 `aosp_cf_x86_64_only_phone-trunk_staging-userdebug`。Trusty 单独处理；90 分钟仍未进入有效编译时，转官方预构建镜像验证主机与 Cuttlefish 启动。
3. 构建开始即在保留副本的本地电脑运行备份：

   ```bash
   python3 tools/aosp/backup-artifacts.py --watch \
     --remote-out /mnt/aosp-out/aosp-out \
     --evidence-root /mnt/aosp-out/evidence \
     --evidence-root /mnt/aosp-out/logs \
     --local-dir ../.local/aosp-artifacts/2026-09-22-rebuild/custom
   ```

   脚本用 `rsync --partial --append-verify` 获取镜像/包和指定证据目录，只有远端传输前后与本地 SHA-256 相符才发布已验证副本。它不负责生成 manifest、日志或 patch；构建流程必须先保存这些文件。stock 与自定义产物使用不同备份目录。
4. 依次验证 Cuttlefish/ADB、`sideagentd`、`agentos`、Plugin 发现/启用/握手/禁用/删除用户、Binder death 和 freezer。停机前再执行 `--once`，检查本地文件、`index.json`、`SHA256SUMS`、日志和 manifest，并确认最新代码已推送。

当前同步握手仍可能被不返回的 Plugin 耗尽两个工作线程；完整 MCP、tool/resource 调用、capability lease、前端迁移和 freezer 矩阵也未移植或验证。详细缺口以 [全局 TODO](aosp-todo.md) 为准。当前结论是 Cuttlefish 镜像路径可构建并完成已列出的设备验证；Pixel 8 和完整产品级系统仍不能标为 Ready。
