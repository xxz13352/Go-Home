# Go-Home 部署说明（本机）

部署目录：`<repo 根目录>`（本文中的相对路径都相对它）
上游仓库：<https://github.com/huanchong-99/Go-Home>

## 1. 已完成的部署步骤

| 项目 | 状态 | 位置 |
| --- | --- | --- |
| Python 虚拟环境 | ✅ Python 3.13.12 | `.venv/` |
| Python 依赖 | ✅ customtkinter / tkcalendar / openai / mcp / fastmcp / DrissionPage 等 | `.venv/` |
| 12306 MCP（TypeScript） | ✅ 已编译 | `12306-mcp/build/index.js` |
| FlightTicketMCP（Python） | ✅ 已 editable 安装 | `FlightTicketMCP/` |
| 大模型配置 | ✅ 本地网关 | `config.json` -> `http://127.0.0.1:7863/v1`，模型 `cn:deepseek-v4.1-flash` |
| Node.js | ✅ 系统安装 | `C:\Program Files\nodejs\node.EXE` |

## 2. 启动方式

```bash
cd go-home && .venv/Scripts/python.exe main.py
```

窗口标题为「Go-home - 回家最优路线查询系统」。

## 3. 相对上游仓库的修改

上游代码有几处在本机环境下会直接崩溃，已做了最小改动：

1. **`main.py` — 运行时路径**
   - 原代码硬编码 conda 环境 `G:\conda environment\Go-home`，本机不存在。
   - 改为：优先 `.venv/Scripts/python.exe` → 其次 conda 路径 → 最后 `sys.executable`；Node 用 `shutil.which("node")`。
2. **`main.py` — MCP 子进程环境变量**
   - 原代码启动 MCP 子进程时未传 `env`，导致 `GO_HOME_CHROME_DEBUG_PORT`、代理等变量丢失。
   - 改为 `env=dict(os.environ)`。
3. **`12306-mcp/src/index.ts` — 发车查询路径不再致命**
   - 12306 已把 `/otn/lcQuery/init` 重定向到登录页，原代码取不到就抛异常导致整个 MCP 服务退出。
   - 改为捕获异常、返回空串；仅「中转换乘票」工具不可用，普通余票查询正常（已实测）。
4. **`FlightTicketMCP/.../flight_search_tools.py` — 携程反爬绕过**
   - 携程 whaleguard 会对 DrissionPage 自带启动参数（UA 伪造、`--no-sandbox`、`--enable-automation`）返回 HTTP 432。
   - 改为：用干净的 Chrome 子进程带 `--remote-debugging-port` 启动，再 `ChromiumPage("127.0.0.1:PORT")` 附加；同时清理 Chrome 单实例锁文件。
   - 另补齐了中转时长 `1d45m` / `11h55m` 等格式的解析（原正则只认 `XhYm`）。

## 4. 本机特有的启动方式（受限环境）

如果 Python 子进程无法直接拉起 Chrome，可先手动启动调试端口，再让 MCP 附加：

```bash
"/c/Program Files/Google/Chrome/Application/chrome.exe" --remote-debugging-port=9222 --user-data-dir="$(pwd)/browser_data" about:blank
```

然后设置环境变量再启动：

```bash
GO_HOME_CHROME_DEBUG_PORT=9222 .venv/Scripts/python.exe main.py
```

## 5. 无界面查询

本仓库只提交了上面四处修复。配套的离线分析脚本（批量查价、按「火车最低有票席别、
住宿不计费」的口径排序、以及用去哪儿 H5 作为第二价格源比价）是本地工作副本里的
工具，没有一并提交，这里不再展开。

## 6. 已知限制

- **12306 预售期 15 天**：查询远期日期时，车次时刻与票价会按「今天 + 14 天」的参考日期返回，不是目标日期的真实票价。
- 携程机票为实时抓取价格，页面显示为「起价」，不含机建燃油（约 ¥50 机建 + ¥10~30 燃油）。
- 携程反爬会随浏览器环境变化而失效，若某次查询返回 0 条，重启 Chrome 调试实例即可。
