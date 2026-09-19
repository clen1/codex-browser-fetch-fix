---
name: codex-browser-proxy-repair
description: 排查并修复 Windows 上 Codex 浏览器控制的 nodeRepl.fetch request failed，尤其是配置过本机代理后成功、应用更新后再次失效的情况。定位当前运行时，备份并最小补回代理配置，重启后使用真实浏览器工具验收。
---

# Codex 浏览器代理修复

目标是恢复用户原本使用的浏览器控制工具。相同报错可能有不同原因；本技能记录的是本机验证过的临时修复，不是官方永久配置方案。仅测试连接的请求先做只读验证；用户要求修复时，在证据支持后完成备份和最小修改。

## 先确认失败位置

使用当前提供的官方浏览器工具，遵循其初始化与浏览器选择说明。区分浏览器发现、标签页枚举、页面内容读取分别是否成功。记下报错和耗时；不要只根据扩展显示“已连接”判断恢复。

本案例报错为 `nodeRepl.fetch request failed`，约 21 秒后失败。新会话、重装扩展、单独重启浏览器未解决。不要反复执行这些无证据的操作，也不要改用另一套浏览器工具后宣布原工具已修好。

## 定位当前生效入口

Windows 的候选配置路径：

`%USERPROFILE%\.codex\plugins\cache\openai-bundled\unified-computer-use\<当前版本>\.mcp.json`

读取实际生效配置的 `mcpServers.cua_repl.command` 和 `args`，得到自带 Node 与启动入口。多版本并存时结合当前加载的插件或进程信息核对，不能仅按文件夹日期猜测。只读取需要的字段，不打印令牌、完整环境或无关配置。

入口常见结构：

`%LOCALAPPDATA%\OpenAI\Codex\runtimes\cua_node\<运行时标识>\bin\node_modules\@oai\cua-repl\bin\cua-repl.mjs`

路径及文件名都可能变化。检查入口、启动逻辑和子进程是否继承 `process.env`。不要硬编码历史运行时标识，不要用旧版文件覆盖新版。

## 验证代理与适用性

使用用户提供或已确认的本机 HTTP/mixed 代理地址。历史示例是 `http://127.0.0.1:10120`，不是通用默认值。

1. 检查相应端口是否监听。
2. 对同一个公开地址比较直连和显式代理请求，设置约 8–15 秒超时，仅输出状态码、耗时与错误类型，不输出 Cookie 或认证信息。
3. 使用配置中的实际 Node 核对版本、环境代理支持，并做相同网络验证。历史案例为 Node v24.21.0，支持 `NODE_USE_ENV_PROXY=1`。
4. HTTP 403 只说明收到响应，不代表登录、接口或浏览器控制成功。若代理不可用，先解决代理可达性；若补丁已存在或没有证据表明代理路径异常，继续定位其他原因，不叠加补丁。

## 最小修复

仅在当前入口缺少所需配置、运行时支持且代理验证支持该路线时应用。先建立带时间戳、不覆盖已有文件的同版本备份，记录绝对路径。保留原有导入、启动和异常处理，在启动子进程之前添加下列逻辑。将示例地址换为实际地址：

```js
// Local proxy repair: preserve loopback access for the browser bridge.
const proxyAddress = "http://127.0.0.1:10120";
const bypassEntries = [process.env.NO_PROXY, process.env.no_proxy]
  .filter(Boolean).flatMap(v => v.split(",")).map(v => v.trim()).filter(Boolean);
const mergedBypass = [...new Set([...bypassEntries, "localhost", "127.0.0.1", "::1"])].join(",");
Object.assign(process.env, {
  NODE_USE_ENV_PROXY: "1",
  HTTP_PROXY: proxyAddress, HTTPS_PROXY: proxyAddress,
  http_proxy: proxyAddress, https_proxy: proxyAddress,
  NO_PROXY: mergedBypass, no_proxy: mergedBypass
});
```

检查变量名是否与新版代码冲突。已存在的等价配置不重复插入；保留已有必要 NO_PROXY 条目。不要关闭 TLS 校验、修改安全审批逻辑、关闭更新，或无关地修改系统全局代理。

使用实际 Node 执行 `--check <入口完整路径>`，核对差异只有目标配置。失败时用本次同版本备份恢复并再次检查。向用户报告修改文件、备份路径和验收进度。

## 重启与验收

修改磁盘文件不会更新已运行进程的代码和环境；新聊天也可能复用控制进程。完成修改与检查后，说明需要用户保存工作、彻底退出再打开 Codex，保持代理运行。不要突然终止应用或浏览器。

用户重启后实际执行：

- 列出 Chrome 标签页。
- 读取一个相关公开页面的正文或控件。
- 在需要完整操作验收时新建公开测试页面，确认打开并读到内容，不提交表单或修改用户资料。

按实际完成的检查报告结果。语法通过、HTTP 响应、发现浏览器都不能单独等同于控制恢复。如果仍失败，读取工具排障说明并基于新证据继续；不反复加补丁或无效重启。

## 更新复发、替代路线与回滚

应用更新可能切换运行目录或替换入口；每次重新从生效配置定位，检查新版是否仍需要修复。2026-09-19 本机观察到新的运行目录不含代理配置，旧目录已不存在；补回后重启，真实标签页列表与 GitHub 正文读取成功。这是个案证据，不应推断所有同名错误都由更新造成。

用户希望采用 TUN 模式时，可说明它是让流量经过代理的另一条路线，是否适用取决于客户端、服务、路由及规则；先核对实际客户端再操作，不能保证开启即解决，也不能把未测试的 TUN 路线写成本机已验证方案。

回滚只使用当前同版本原文件备份：先另存修改后的文件，再恢复备份、检查语法、重启并验收。不要删除备份，不要将旧运行时备份覆盖新运行时。

浏览器已可控制但上传出现 `Not allowed` 是另一类问题：检查工具上传文档及扩展“允许访问文件网址”要求，不应重新应用网络补丁。

## 调用示例

> 使用 $codex-browser-proxy-repair 修复 Codex 控制 Chrome 的 nodeRepl.fetch request failed。我的本机代理是 http://127.0.0.1:10120。请定位当前生效入口，验证后备份并最小修复；需要重启时先完成修改与检查，重启后用真实浏览器工具验收。
