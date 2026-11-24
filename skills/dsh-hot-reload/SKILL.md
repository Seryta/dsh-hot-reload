---
name: dsh-hot-reload
description: 在开发或修复 DSH 插件后热重载使改动生效，或用户要求免重启更新、检查热重载能力时使用。复用 Plugin Manager/HMR 快速路径；没有管理器入口（外部换包）时用 profile 补丁内容变化触发重算，并以探针 + journal 验证，区分 Host 更新、Client 刷新和服务重启。
---

# DSH 插件热重载

本技能的唯一来源是本仓库 `skills/dsh-hot-reload/SKILL.md`；全局 `~/.dsh/skills/dsh-hot-reload` 是指向该目录的符号链接，不复制内容，改仓库即生效。只处理用户已授权的插件变更和生效；纯能力询问或评审保持只读。创造模式提供工具条件，不会自动执行热重载。

## 已验证的快速路径

在 DSH 0.2.2-rc.3 的 Web profile 中已实测：Plugin Manager 可以保持插件启用并触发 profile reconciliation，使已变化的本地链接包 Host 模块生效，无需重启 dsh-web。同一会话、同一 profile 已核实的 entry id 和 HMR 状态可复用，不必每次重扫工具、服务和全部文档。

1. 确认 profile 实际加载目录指向改动后的包。需要构建时先完成构建与测试，并按 dev-resource-manager 使用受控资源；修改另一份副本不会生效。
2. entry 不明时调用 plugin_manager 的 list_plugins，按 nextOffset 翻页取得准确 entryId、enabled、fiberPhase，并确认 @deepseek-ai/dsh-hmr 为 active。仅需安装来源或组合包标识时才查 list_bundles。
3. 结束自己发起的插件业务探测和长连接，不在同一执行链里打开业务请求后立即卸载它的监听器。涉及在途业务时遵循插件排空接口，不强杀用户任务。
4. 对已启用且本次应继续启用的目标调用：

```json
{"action":"set_plugin","target":"<准确 entryId>","enabled":true}
```

目标原本被禁用时不擅自开启。不要用重装包、手改 profile 文件、先关闭再打开或启动第二个 DSH 进程替代这一入口。

5. 读取 application、warnings、错误及工具给出的 nextStep，再验证实际结果。受理不等于成功。

| 返回 | 判断 |
|---|---|
| application: applied | 运行时重新应用成功；继续核查 active 与相关业务。 |
| changed:false 且 applied | 启用状态没变，重组合仍执行。没有源码变化时，不代表强制重建过模块实例。 |
| restart-required | 按具体原因和 nextStep 报告，不自动转为重启服务。 |
| failed / overridden / cancelled | 尚未达到目标，处理实际诊断，不宣称完成。 |

set_plugin 触发 profile 重组合，会检查该 profile 内其它已变化的包，不能承诺只清除一个包的缓存。

## 外部换包：没有"应用"按钮时的手工触发

适用：包已经在 profile 的 `node_modules` 里换好（手工 `pnpm install`、替换 tgz、拷目录），但插件页没有待应用项，且当前 preset 不提供 `plugin_manager`。此时界面本来就没有可点的"应用"，不是界面 bug。

原理（DSH 0.2.2-rc.5 实测）：`@deepseek-ai/dsh-hmr` 在 profile 启动时监听 `profiles/<name>/package.json`（只有 `dsh.profile.bundles` 列表变化才重算）、`profiles/<name>/cordis.patch.yml` 和 `~/.dsh/cordis.patch.yml`；任一文件**内容变化**就 `runExclusive → reconcileProfile`，比较已加载模块与磁盘并替换变化的插件。`root: []` 只表示不监听源码目录，不代表不能重算。

```bash
cd ~/.dsh/profiles/<profile>
cp -a cordis.patch.yml cordis.patch.yml.bak-$(date +%Y%m%d-%H%M%S)
printf '\n# hot-apply trigger %s\n' "$(date +%H:%M:%S)" >> cordis.patch.yml
```

- 必须**内容**变化；`touch` 只改 mtime，不触发。
- "加一行 → 触发 → 删掉 → 再触发"可行，每次内容变化都会重算；但真正决定是否替换的是**插件文件在磁盘上变了没有**，不是这行注释。
- 客户端 `client.js` 仍要整页刷新。

### 验证：探针放对入口

最常见的假阴性是探针写进了宿主根本不 import 的文件。先确认包入口：

```bash
node -e "console.log(require(process.env.HOME + '/.dsh/profiles/<profile>/node_modules/<pkg>/package.json').exports)"
```

`exports['.']` 常指向 `runtime/<gen>/index.js`，仓库根的 `index.js` 可能只是冻结兼容文件。把探针加在真正入口，触发后读日志：

```bash
journalctl --user -u dsh-web --since '-2 min' --no-pager | grep -v "model listing" | tail -20
```

| 观察 | 判断 |
|---|---|
| 出现探针行 | 重算确实重新 import 了该插件，新代码生效 |
| `restart required: ...` | 这一轮热应用不了，只能重启（仍需用户同意） |
| `changed modules keep running their previous code until a restart` | 部分模块替换失败，同上 |
| 什么都没有 | 先确认内容真的变了，再确认该插件在这轮替换范围内 |

不要用 atime 判断是否重新加载：relatime 下读取通常不更新 atime，观测不到等于没有。

收尾：恢复探针文件与补丁原始字节后再触发一次；用 `diff -rq <解包归档>/package <安装目录>` 确认安装目录与交付物逐字节一致。

### 会话与 preset revision

- 旧会话 resume（重新打开）时按**当前** preset revision 绑定，能直接吃到新代码；不需要删会话、也不需要重建。
- 仍在运行的 live Agent 保留自己创建时的 revision（DSH 有意不中途换插件）；把它关掉再打开即可重新绑定。
- 已经开过一轮的会话不能切换 preset id（`agent-preset/locked`），但这不影响同 id 的 resume 绑定。
- fork 的子会话继承父会话的 revision，不是当前 revision：想要"旧上下文 + 新代码"应重新打开原会话，而不是 fork。

## Host、Client 与 shell

- Host 插件需要 HMR/profile reconciliation 加载新模块；仅刷新浏览器不够。
- Client 插件仍须确认浏览器加载了新代码。没有浏览器控制时说明需要刷新页面，不声称已刷新。只有确认同一 DSH checkout 的受管 pnpm run dev:web watcher 正在重建 bundle，且客户端收到更新后，才承诺自动更新。
- Web shell、普通包与宿主静态依赖不保证能按插件热替换；按当前安装方式重建受影响产物并验证原 GUI URL，不新开服务器冒充更新了现有 GUI。

## 最小验证

1. 重新查询目标插件，应为 fiberPhase: active 且无新启动诊断。Inspect Config 的 status: absent 只表示没有暴露 Config schema，不表示插件没加载。
2. 优先通过已有认证页面或正式连接做相关只读业务检查。不搜索登录 token、私人浏览器 profile 或凭据来补验证。
3. 匿名 401 只证明请求被拒，不能单独证明新版业务协议；代理的 302 登录跳转不能证明插件健康。无法认证时明确业务未验证。
4. 报告“服务未重启”须比较前后的 PID 和启动时间。本机 systemd Web 的只读入口：

```bash
systemctl --user --machine="$(id -un)@.host" show dsh-web -p MainPID -p ExecMainStartTimestamp
```

其它部署用已有管理器的等价读数；沙箱里的 ps 不保证看得到宿主进程。非 root 的 journalctl --machine 在本机不支持；如需日志，先用普通 journalctl --user -u dsh-web，并只输出有限相关内容。

5. 交付说明重新应用结果、插件状态、进程是否保持、业务验证范围及 Client 是否仍需刷新，不贴整张插件表。

可记录机器可读证据，未知填 null：

```json
{"target":"<entryId>","application":"applied","warnings":[],"fiberPhase":"active","process":{"pidBefore":null,"pidAfter":null,"startedBefore":null,"startedAfter":null},"business":{"verified":false,"evidence":"<实际结果或未验证原因>"},"client":{"refreshed":false,"refreshRequired":true}}
```

## 仅在快速路径失效时重新发现

加载 dsh-doc，调用 cordis_inspect_list 后定向查询真实 Provider：Host Service.listService 的 hmr、pluginManager，或 Config.listConfigs 的目标包。只有工具可用性不明时才查 Tool.listTools。Inspect 是只读契约接口，不能用于调用业务 Service；不要修改 preset 或临时注入来获取原本没有的执行能力。

按 dsh-doc 定位当前安装版本，只读 @deepseek-ai/dsh-hmr 与 @deepseek-ai/dsh-plugin-manager 的 README 和相关源码，不硬编码安装路径，不假设旧 dev_reload_package/dev_plugin_status 工具存在。

已验证调用链：setPluginEnabled → configure → hmr.runExclusive → reload → hmr.reconcileProfile。对比范围覆盖 profile 模块及本地链接包目标；root:[] 表示没有源码自动监听，不代表不能通过管理器显式重组合。

常见不能全部热替换的原因包括 no-hmr、package-moved、host-module、unmanaged-plugin、module-comparison，以实际返回为准。

## 权限与重启边界

本技能不增加权限。热重载不包含服务重启授权；不要安排延迟重启或把失败转为后台重启。用户明确要求重启后，按当时生效的服务管理规则另行执行。

完全访问的默认预设在本版本绑定 approval:never；仍需审批的工具会在到达界面前被自动拒绝，错误“用户拒绝”不代表用户点击过拒绝。遇到这种情况核对当前策略及审批记录，不凭报错猜测用户操作，也不更改审批控制来绕过拒绝。

## 实测示例

2026-11-16，Web profile 的 dsh-sidebar-triptych 链接本地修复仓库，entry 为 include:sidebar-triptych。重新应用 enabled:true 返回 application:applied、changed:false、warnings:[]；插件为 active，服务 PID 和启动时间未变。此例不是固定配置；没有源码变化的再次应用是幂等重组合，不保证重新执行 apply。

2026-11-23，DSH 0.2.2-rc.5，Web profile 手工 `pnpm install` 换成 `dsh-learn-mode@0.1.2`（本地 tgz），插件页没有可点的应用操作、当前 preset 也无 `plugin_manager`。按本文手工触发重算后，探针在 `runtime/0.1.2/index.js` 打出 `[learn-probe] runtime 0.1.2 host entry loaded`（18:20:43），PID 2908 与启动时间未变；恢复探针与补丁字节后再触发一次，安装目录与归档 `diff -rq` 无差异，`learn_graph` 存储 mtime 与大小未变。
