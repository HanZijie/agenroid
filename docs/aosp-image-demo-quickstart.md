# AOSP 镜像打包与 Demo 验证

这条路径用于把 AgentOS 和三个 Demo App 打进 Android 15 Cuttlefish
`userdebug` 镜像，并在启动后的设备上做最小可重复验证。AOSP 源码、镜像、
构建日志和密钥都放在仓库外；仓库只保存 overlay、脚本和本教程。

## 当前结论

以下是仓库已有证据支持的结论：

- `android-15.0.0_r34` 的
  `aosp_cf_x86_64_only_phone-trunk_staging-userdebug` 已完成过完整 `droid`
  构建，生成了 Cuttlefish 镜像和 host package。
- 自定义 Cuttlefish 实例已启动到 `sys.boot_completed=1`；
  `sideagentd`、`agentos` 和 `agentos.sideagentd` 已注册，
  `cmd agentos health` 返回 `state=ready`。
- `AgentOsPluginProbe` 的发现、启用、握手、禁用、重新绑定和用户清理路径
  已有 Cuttlefish 验证记录；三个 Demo APK 也可以通过本教程的参数一起进入
  product 镜像。
- `NotesPluginCrossAppTest` 已在 stock Cuttlefish 上通过。这个结果证明
  Android App 级跨进程 Demo 路径可运行，不等同于 native `sideagentd` 的
  MiniMax/Jev 真实请求验收。

Pixel 8 真机、干净主机从零复现、完整 Plugin tool/resource 数据面以及带真实
MiniMax/Jev 凭据的 runtime 验收不在这份快速教程内。需要真实模型请求时，使用
[`cuttlefish-runtime-acceptance.md`](cuttlefish-runtime-acceptance.md)；平台剩余
缺口以 [`platform/aosp-integration/aosp-todo.md`](../platform/aosp-integration/aosp-todo.md)
为准。

## 1. 准备环境

需要一台 Linux x86_64 构建机，以及 JDK 17、`repo`、Android SDK/ADB、可用的
`/dev/kvm` 和 Cuttlefish host。AOSP checkout 必须是仓库接线脚本要求的
`android-15.0.0_r34`，并放在本仓库之外。例如：

```bash
cd /path/to/agenriod
AOSP=/mnt/aosp-src/aosp-r34
OUT=/mnt/aosp-out/agentos-demo
EVIDENCE=/mnt/aosp-evidence/agentos-demo

test -f "$AOSP/.repo/manifests/default.xml"
```

第一次准备 AOSP 时，按 AOSP 官方方式同步 `android-15.0.0_r34`，再让
`wire-platform.py` 检查 manifest 和参与接线项目的 revision。不要把 AOSP 源码
或 `OUT` 放进 Git，也不要把密钥写入命令行、APK 或镜像归档。

## 2. 构建前端和 Demo APK

先构建 AgentOS 前端、Notes Plugin 和三个独立 Demo。所有 Android Gradle 命令
都经过仓库脚本：

```bash
./tools/android/android.sh gradle \
  :frontends:agenriod:assembleDebug \
  :plugins:notes:assembleDebug

./tools/android/android.sh gradle -p demo-apps \
  :meeting-records:assembleDebug \
  :calendar:assembleDebug \
  :alarm:assembleDebug
```

确认 APK 已生成：

```bash
test -f frontends/agenriod/build/outputs/apk/debug/agenriod-debug.apk
test -f plugins/notes/build/outputs/apk/debug/notes-debug.apk
test -f demo-apps/alarm/build/outputs/apk/debug/alarm-debug.apk
test -f demo-apps/calendar/build/outputs/apk/debug/calendar-debug.apk
test -f demo-apps/meeting-records/build/outputs/apk/debug/meeting-records-debug.apk
```

## 3. 接线并打包 Cuttlefish 镜像

先预览接线差异。这个命令只读 AOSP checkout，不会修改它：

```bash
python3 tools/aosp/wire-platform.py "$AOSP" --target cuttlefish
```

确认差异后，用统一构建入口一次完成接线、Soong 分析、平台构建和 runtime
packaging gate。传入五个 APK 后，前端、Notes 和三个 Demo 都会被 staged 到
product 镜像；三个 Demo 参数必须同时传入。

```bash
python3 tools/aosp/build-agentos.py "$AOSP" \
  --target cuttlefish \
  --product aosp_cf_x86_64_only_phone \
  --release trunk_staging \
  --variant userdebug \
  --out-dir "$OUT" \
  --evidence-dir "$EVIDENCE" \
  --frontend-apk frontends/agenriod/build/outputs/apk/debug/agenriod-debug.apk \
  --notes-apk plugins/notes/build/outputs/apk/debug/notes-debug.apk \
  --demo-alarm-apk demo-apps/alarm/build/outputs/apk/debug/alarm-debug.apk \
  --demo-calendar-apk demo-apps/calendar/build/outputs/apk/debug/calendar-debug.apk \
  --demo-meeting-records-apk demo-apps/meeting-records/build/outputs/apk/debug/meeting-records-debug.apk \
  --stage all \
  --jobs "$(nproc)"
```

构建成功后，重点产物在下面两个目录：

```text
$OUT/target/product/aosp_cf_x86_64_only_phone/  # boot/system/vendor/super 等镜像
$OUT/host/linux-x86/                            # Cuttlefish host package
$EVIDENCE/build-agentos.log                     # 分阶段构建日志
$EVIDENCE/build-agentos.report.json             # 结构化结果
```

`build-agentos.py` 会为同一个 `--out-dir` 加锁，避免两个构建互相覆盖。构建失败
时先看 `build-agentos.log` 和 report 中的 `failed_stage`，不要把失败的输出当作
可启动镜像。

## 4. 启动并检查设备

启动方式取决于 host package 是直接运行还是放在 Cuttlefish Docker 容器里。直接
运行时，把 `$OUT/host/linux-x86` 和 `$OUT/target/product/...` 交给同一套
`launch_cvd`；容器、KVM 设备映射和 SwiftShader 参数见
[`cuttlefish-host.md`](../platform/aosp-integration/cuttlefish-host.md)。启动成功
后确认 ADB 序列号，例如 `127.0.0.1:6521`。

```bash
SERIAL=127.0.0.1:6521
adb connect "$SERIAL"
adb -s "$SERIAL" wait-for-device
test "$(adb -s "$SERIAL" shell getprop sys.boot_completed | tr -d '\r')" = 1
adb -s "$SERIAL" shell getprop ro.build.fingerprint
adb -s "$SERIAL" shell service check agentos
adb -s "$SERIAL" shell service check agentos.sideagentd
adb -s "$SERIAL" shell cmd agentos health
```

最后一条应返回包含 `"state":"ready"` 的 JSON。也可以把同样的只读检查保存
成报告；远程 host/container 场景使用 `tools/aosp/check-cuttlefish.py` 的
`--mode agentos`。

## 5. 做 Demo 烟测

镜像内的三个 Demo 是平台签名的 product APK。先检查包存在，再检查
`AgentManagerService` 已发现它们：

```bash
for package in \
  com.example.agentos.demo.alarm \
  com.example.agentos.demo.calendar \
  com.example.agentos.demo.records; do
  adb -s "$SERIAL" shell pm path "$package"
done

adb -s "$SERIAL" shell cmd agentos plugins --user 0
```

在返回结果中应能看到三个 `pluginId`。如果需要重新触发某个 Demo 的发现，
可以显式启用它：

```bash
adb -s "$SERIAL" shell cmd agentos enable --user 0 com.example.agentos.demo.records
adb -s "$SERIAL" shell cmd agentos enable --user 0 com.example.agentos.demo.calendar
adb -s "$SERIAL" shell cmd agentos enable --user 0 com.example.agentos.demo.alarm
adb -s "$SERIAL" shell cmd agentos plugins --user 0
```

再分别打开三个界面，完成一次最小用户操作：

```bash
adb -s "$SERIAL" shell am start -W -n com.example.agentos.demo.records/.MainActivity
adb -s "$SERIAL" shell am start -W -n com.example.agentos.demo.calendar/.MainActivity
adb -s "$SERIAL" shell am start -W -n com.example.agentos.demo.alarm/.MainActivity
```

手工验收可以记录为：记录 App 新建并保存一条会议记录，日历 App 新建一条
日程，闹钟 App 设置并启用一次性闹钟。它们验证的是 APK、Activity、私有数据和
Plugin descriptor 的基本路径；当前 AOSP overlay 仍未宣称完整的 tool/resource
调用、lease 和 cancel 数据面已经完成。

需要复现已有的 App 级跨进程测试时，可参考
[`cuttlefish-host.md`](../platform/aosp-integration/cuttlefish-host.md) 中的
`NotesPluginCrossAppTest` 记录。需要验证 native Jev/MiniMax 请求和重启恢复时，
不要把上述 Demo 烟测当作替代品，改用
[`tools/aosp/test-agentos-runtime.py`](../tools/aosp/test-agentos-runtime.py)。

## 6. 保留结果

每次构建至少保留以下信息：Git revision、AOSP resolved manifest、lunch target、
构建日志、镜像 SHA-256、Cuttlefish 启动日志、ADB 序列号、`cmd agentos health`
结果和 Demo 烟测报告。建议把它们放在 `$EVIDENCE` 或
`../.local/aosp-artifacts/<date>/`，不要提交到仓库。完成后可用：

```bash
sha256sum "$OUT"/target/product/aosp_cf_x86_64_only_phone/*.img \
  > "$EVIDENCE/image-sha256.txt"
```

这份教程默认只覆盖 Cuttlefish；`--target pixel8` 是另一条需要单独检查设备树、
vendor、AVB 和真机启动的路线。
