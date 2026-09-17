# 如何解决 Codex 控制浏览器时的 nodeRepl.fetch request failed 报错

本仓库分享 Windows 上的实际排障过程、修改方法与可直接复制给 AI 的提示词。配套中文视频已制作完成，上传中；下载链接将在发布后补充。

## 适用范围

2026-09-14 的本机案例：能发现 Chrome，但读取标签页约 21 秒后出现 `nodeRepl.fetch request failed`；新会话、重装扩展仍失败。显式配置控制启动器的代理并完全重启后，成功读取标签页、打开 Google 并读取页面内容。

这是本机验证的临时修复，相同报错不一定来自同一原因。代理端口、插件版本、运行时目录必须以实际配置为准。更新可能切换运行时并使补丁失效。

## 一、检查代理与网络路径

本例 HTTP 代理为 `http://127.0.0.1:10120`。先检查监听，再比较同一地址的直连和代理路径：

```powershell
Get-NetTCPConnection -State Listen -LocalPort 10120 | Select-Object LocalAddress,LocalPort,OwningProcess

curl.exe --noproxy '*' --connect-timeout 8 --max-time 12 -sS -o NUL -w 'direct HTTP=%{http_code} time=%{time_total}' https://chatgpt.com/backend-api/me

curl.exe --proxy http://127.0.0.1:10120 --connect-timeout 8 --max-time 12 -sS -o NUL -w 'proxy HTTP=%{http_code} time=%{time_total}' https://chatgpt.com/backend-api/me
```

本次直连超时，代理收到 HTTP 403。403 仅说明收到 HTTP 响应，不代表认证、接口功能或浏览器控制成功。测试不需要添加账号令牌。端口应替换成你自己的实际端口。

## 二、找到当前真正生效的文件

检查当前插件配置：

```text
%USERPROFILE%\.codex\plugins\cache\openai-bundled\unified-computer-use\<当前版本>\.mcp.json
```

读取 `mcpServers.cua_repl`，其中 `command` 是实际自带 Node 程序，`args` 指向实际启动入口。本例入口结构如下：

```text
%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<运行时标识>\bin\node_modules\@oai\cua-repl\bin\cua-repl.mjs
```

不要把占位符当成真实路径。旧版可能使用 `launch.mjs`，应从当前配置定位，检查启动逻辑、子进程环境继承及 Node 的环境代理支持，再决定修改位置。不要用旧版整文件覆盖新版。

## 三、备份原文件

将下列路径替换为上一步查到的实际值：

```powershell
$entry = '实际 cua-repl.mjs 完整路径'
$node = '实际 node.exe 完整路径'
$backup = $entry + '.bak-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
Copy-Item -LiteralPath $entry -Destination $backup
Get-Item -LiteralPath $entry,$backup
```

记录备份路径，不要覆盖已有备份。

## 四、最小化添加代理配置

本例在入口脚本的 `await cua_repl.launch()` 之前加入以下代码，保留原有导入、异常处理与其他逻辑：

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

如果已有 `NO_PROXY` 或 `no_proxy`，应合并并保留必要条目，不能直接丢弃。该方案要求实际 Node 运行时支持对应环境代理机制，并且相关子进程继承环境。本例对此进行了检查和独立网络验证。

该修改作用于启动器及继承其环境的子进程，不需要修改系统全局代理、网站授权或安全审批逻辑。

## 五、语法检查与重启

```powershell
& $node --check $entry
```

检查通过后保存其他任务，完全退出 Codex 再重新打开，并保持代理运行。

**为什么要重新打开？** 已经运行的浏览器控制进程仍使用启动时载入的代码和环境。改磁盘文件不会自动替换其内存中的程序，新建会话也可能继续复用旧进程。重新启动相关控制进程才能加载修改；完全退出并重开 Codex 是本案例采用的方式，不要求重启整台电脑。

## 六、真实浏览器验收

复制给 AI：

```text
测试 Chrome：先列出现有标签页，再新开 Google 首页或公开测试页，并读取页面里的搜索框、按钮等内容。报告每一步的真实工具结果；不要只根据扩展显示已连接就宣布成功。
```

验收应覆盖标签页列表、打开页面和读取内容三个步骤。本案例通过这些检查，不代表所有网站功能均已验证。

## 七、回滚与更新后处理

确保 `$backup` 对应当前同版本原文件。新开的 PowerShell 需要重新设置 `$entry` 和 `$backup`。先保留修改版本，再恢复备份：

```powershell
$modifiedCopy = $entry + '.before-rollback-' + (Get-Date -Format 'yyyyMMdd-HHmmss')
Copy-Item -LiteralPath $entry -Destination $modifiedCopy
Copy-Item -LiteralPath $backup -Destination $entry -Force
```

恢复后完全重启 Codex。更新可能覆盖补丁或切换运行时；重新定位当前入口，先验证新版本是否已修复，再决定是否重新添加补丁。不要向新版本直接覆盖旧版本备份。

## 八、可直接复制给 AI 的完整提示词

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

## 九、配套视频

8 分 29 秒、1080p 横屏、中文合成配音、逐句字幕、20 个章节及 Remotion 动画。画面是案例重现，不是实时录屏。视频和可编辑工程将在上传完成后添加下载入口。

## 参考资料

- [OpenAI 浏览器扩展说明](https://learn.chatgpt.com/zh-Hans/docs/chrome-extension)
- [公开报告 #44364：代理临时修复](https://github.com/openai/codex/issues/44364)
- [公开报告 #44697：发现成功但标签页请求超时](https://github.com/openai/codex/issues/44697)

公开报告属于用户经验，不等于官方确认的通用根因。视频中的成功结果来自 2026-09-14 本机验收记录。
