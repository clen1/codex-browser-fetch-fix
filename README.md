# 如何解决 Codex 控制浏览器时的 nodeRepl.fetch request failed 报错

[下载 v1.1 视频及配套资料](https://github.com/clen1/codex-browser-fetch-fix/releases/tag/v1.1.0)：10 分 04 秒、1080p、23 个章节、中文合成配音与逐句字幕。新增更新后恢复指南。

![教程封面](https://github.com/clen1/codex-browser-fetch-fix/releases/download/v1.1.0/cover.png)

## 先明确适用范围

2026-09-14 的本机案例：能发现 Chrome，但读取标签页约 21 秒后出现 `nodeRepl.fetch request failed`；新会话、重装扩展仍失败。显式配置控制启动器的代理，完全重启后，成功读取标签页、打开 Google 并读取页面内容。

这是本机验证的临时修复。相同报错不一定来自同一原因。代理端口、插件版本、运行时目录都要以实际配置为准。

## 一、检查代理

本例本机 HTTP 代理：`http://127.0.0.1:10120`。先检查端口监听：

```powershell
Get-NetTCPConnection -State Listen -LocalPort 10120 | Select-Object LocalAddress,LocalPort,OwningProcess
```

直连测试：

```powershell
curl.exe --noproxy '*' --connect-timeout 8 --max-time 12 -sS -o NUL -w 'direct HTTP=%{http_code} time=%{time_total}' https://chatgpt.com/backend-api/me
```

代理测试：

```powershell
curl.exe --proxy http://127.0.0.1:10120 --connect-timeout 8 --max-time 12 -sS -o NUL -w 'proxy HTTP=%{http_code} time=%{time_total}' https://chatgpt.com/backend-api/me
```

本次直连超时，代理收到 HTTP 403。403 仅说明收到 HTTP 响应，不代表认证、接口功能或浏览器控制成功。测试不需要添加账号令牌。

## 二、找到真正需要修改的文件

打开以下目录，核对当前生效的插件版本：

```text
%USERPROFILE%\.codex\plugins\cache\openai-bundled\unified-computer-use\<当前版本>\.mcp.json
```

读取 `mcpServers.cua_repl`：

- `command`：实际自带 Node 可执行程序。
- `args`：实际启动入口。

本次案例的入口结构如下，**不要把占位符当作真实路径**：

```text
%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<运行时标识>\bin\node_modules\@oai\cua-repl\bin\cua-repl.mjs
```

旧版可能使用 `launch.mjs`。应从当前配置定位，检查其启动逻辑和子进程的环境继承，再决定修改位置。不要用旧版整文件覆盖新版。视频记录的是已验证的版本结构，不保证所有后续版本相同。

## 三、备份原文件

先将两个路径占位符替换为上一步查到的真实路径，再执行：

```powershell
$entry = '实际 cua-repl.mjs 完整路径'
$node = '实际 node.exe 完整路径'
$backup = $entry + '.bak-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
Copy-Item -LiteralPath $entry -Destination $backup
Get-Item -LiteralPath $entry,$backup
```

保留输出中的备份路径，以便回滚。不要覆盖已有备份。

## 四、最小化添加代理配置

本例在入口脚本的 `await cua_repl.launch()` 之前加入下面的代码。保留原有导入、异常处理和全部其他逻辑。

```javascript
Object.assign(process.env, {
  NODE_USE_ENV_PROXY: "1",
  HTTP_PROXY: "http://127.0.0.1:10120",
  HTTPS_PROXY: "http://127.0.0.1:10120",
  http_proxy: "http://127.0.0.1:10120",
  https_proxy: "http://127.0.0.1:10120",
  NO_PROXY: "localhost,127.0.0.1,::1",
  no_proxy: "localhost,127.0.0.1,::1"
});
```

注意：端口需要按实际值替换；若已经设置 `NO_PROXY`，应合并并保留原有必要条目。该方案要求实际 Node 运行时支持相应环境代理机制。本例检查了子进程继承启动器环境，并做了独立网络验证。

这项修改给启动器及继承其环境的子进程配置代理，没有修改系统全局代理，也不需要改网站授权或安全审批逻辑。

## 五、检查语法，完全重启

```powershell
& $node --check $entry
```

语法检查通过后，保存其他任务，完全退出 Codex 再重新打开，保持本机代理运行。单纯新建会话不保证重启控制进程。

## 六、验收必须使用真实浏览器工具

复制给 AI：

```text
测试 Chrome：先列出现有标签页，再新开 Google 首页或公开测试页，并读取页面里的搜索框、按钮等内容。报告每一步的真实工具结果；不要只根据扩展显示已连接就宣布成功。
```

验收标准：标签页列表可读、测试页面可打开、实际页面内容可读。本案例已通过这三项，未据此宣称所有复杂网页功能都已验证。

## 七、回滚

确认 `$backup` 对应**当前同版本原文件**。如果重开了 PowerShell，需重新设置 `$entry` 和 `$backup` 为实际路径。

```powershell
$modifiedCopy = $entry + '.before-rollback-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
Copy-Item -LiteralPath $entry -Destination $modifiedCopy
Copy-Item -LiteralPath $backup -Destination $entry -Force
```

恢复后完全重启 Codex。应用更新可能覆盖补丁或切换运行时，届时重新定位入口；先检查新版本是否已修复，再决定是否重新加补丁。

## 八、直接复制给 AI 的完整提示词

```text
请实际排查并修复我这台 Windows 上 Codex 控制 Chrome 的问题，不要只给通用建议。

症状：能识别 Chrome，但读取标签页时约 21 秒后报 nodeRepl.fetch request failed。新建会话和重装扩展没有解决。

我的本机代理端口是 10120，候选地址为 http://127.0.0.1:10120。请先验证它是否正在监听、是否支持 HTTP 代理，不要直接假定可用。

我授权你进行本机只读诊断，并在证据支持后，备份并最小化修改当前 Codex 浏览器控制启动器，让它使用我的现有本机代理。

执行要求：
1. 用当前官方浏览器工具复现，分清浏览器发现、标签页列表和网页内容读取在哪一步失败。
2. 对相同地址比较直连和代理网络路径。不要使用认证令牌；收到 403 不等于浏览器已经修好。
3. 从当前生效的 unified-computer-use 配置定位 cua_repl 的 command、args 和实际入口，不要照抄旧版本路径。
4. 检查 Node 的代理支持及子进程环境继承。证据支持代理路径异常时，先创建不覆盖已有文件的备份，再添加必要配置。
5. 在合适的启动位置设置 NODE_USE_ENV_PROXY=1、HTTP_PROXY/HTTPS_PROXY 及小写版本，使用实际本机代理地址；NO_PROXY/no_proxy 应包含 localhost,127.0.0.1,::1，并保留已有必要条目。不要用旧版文件覆盖新版，不要改安全审批逻辑或无关系统设置。
6. 使用实际自带 Node 检查语法、核对差异。若需要完全重启，先完成修改与检查，然后告诉我，不要突然退出当前应用。
7. 重启后必须真实读取 Chrome 标签页、打开公开测试页、读到页面内容，才能宣布成功。
8. 报告修改文件的绝对路径、备份位置和回滚方法；如果仍失败，依据证据继续排查，不要认定所有同名错误都是代理问题。

不要输出账号令牌、个人页面内容或其他敏感配置。不要直接删除文件。
参考：https://github.com/openai/codex/issues/44364
```

## 九、发布文案

标题：如何解决 Codex 控制浏览器时的 nodeRepl.fetch request failed 报错

简介：Chrome 明明能被识别，为什么读取标签页还是失败？本期用一个 Windows 实际修复案例，演示代理路径对比、当前启动文件定位、备份与最小修改、重启验收和回滚。附完整代码、命令与 AI 提示词。相同报错可能有不同原因，请先验证再修改。

参考资料：

- [OpenAI 浏览器扩展说明](https://learn.chatgpt.com/zh-Hans/docs/chrome-extension)
- [公开报告 #44364：代理临时修复](https://github.com/openai/codex/issues/44364)
- [公开报告 #44697：发现成功但标签页请求超时](https://github.com/openai/codex/issues/44697)

公开报告属于用户经验，不等于官方确认的通用根因。视频中的成功结果来自 2026-09-14 本机验收记录。


## 更新后再次连接不上怎么办？（2026-09-17 补充）

本机再次遇到同名报错时，当前配置已指向新的运行时目录；新入口没有此前的代理设置，本机 10120 端口仍在监听。已为新入口备份并补回配置，语法检查通过。**这次更新后的连接恢复仍须重启后真实验收，不能仅凭语法检查宣布修好。** 这是本机观察与临时处理，并非官方保证的长期配置方案。

1. **先复现再判断。** 用浏览器工具检查发现浏览器、列标签页、读取页面分别在哪一步失败。如果新版已能正常控制，不需要补丁。
2. **重新核对网络。** 检查实际代理端口仍在监听，对同一地址比较直连与代理。收到 403 只代表有 HTTP 响应，不等于浏览器控制正常。
3. **从当前配置重新找文件。** 读取当前生效的 unified-computer-use 版本下 `.mcp.json`，跟随 `mcpServers.cua_repl.command` 和 `args` 找实际 Node 与入口。不要只按目录名字或修改日期猜哪个版本生效。
4. **检查新入口再最小修改。** 旧补丁可能仍在旧目录，也可能被更新覆盖。核对 Node 的环境代理支持及子进程继承；先备份新版本原文件，再补回缺失配置，保留已有 NO_PROXY 条目。若补丁已存在，继续排查其他原因，不要反复叠加。
5. **检查语法后重启应用。** 用当前配置的 Node 执行 `& $node --check $entry`。保存任务，完全退出 Codex 再打开，代理保持运行；无需重启整台电脑。
6. **做真实验收。** 必须列出 Chrome 标签页、打开公开页面、读到实际页面内容，三项成功才宣布恢复。如果仍失败，保留报错和版本信息，继续定位，必要时恢复同版本备份。

**为什么需要重开？** 修改磁盘文件不会自动替换已运行进程里的代码和环境；新建聊天也可能复用旧控制进程。重启是为了重新加载配置，并不意味着每次连接失败都要重启。

**不要把旧版本的完整脚本或备份覆盖到新版本。** 备份、回滚和补丁都应对应当前版本；不要通过关闭更新来维持旧补丁。

### 更新后复发：直接发给 AI 的提示词

```text
Codex 更新后，浏览器控制再次报 nodeRepl.fetch request failed。之前配置本机代理后成功过，实际代理端口是 10120，请重新核实。
请先复现并比较网络路径，再从当前生效的 unified-computer-use 配置定位 command、args 与实际入口，判断更新是否切换运行时或覆盖补丁。
若证据支持且新版仍需要修复，我授权你备份当前版本原文件，核对 Node 支持和环境继承，最小补回缺失的代理配置，保留已有必要条目。
不要覆盖整个新文件，不要拿旧备份还原新版，也不要重复插入已有补丁。检查语法、报告实际修改和备份路径，完成后告诉我重启要求，不要突然退出应用。
我重开 Codex 后，必须实际列标签页、打开公开页面并读取内容，才可以宣布恢复；若仍失败，继续按证据排查并提供同版本回滚方法。
```

另一个独立问题：浏览器可控制但上传文件提示 `Not allowed`，应检查 Chrome 中 ChatGPT 扩展详情页的「允许访问文件网址」。它与上述网络连接报错不是同一个问题。参见[官方扩展说明](https://learn.chatgpt.com/docs/chrome-extension)。
