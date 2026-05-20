# mitmproxy 抓包指南

## 1. mitmproxy 是什么

mitmproxy 是一个中间人代理工具，可以拦截、查看和修改 HTTP/HTTPS 请求和响应。

## 2. 三种工具对比

| 工具 | 界面 | 适用场景 |
|------|------|----------|
| `mitmproxy` | 终端交互式 TUI | 实时查看、编辑、拦截请求 |
| `mitmweb` | 浏览器 Web 界面（默认 `http://127.0.0.1:8081`） | 可视化查看，类似 Chrome DevTools |
| `mitmdump` | 无界面，纯文本输出 | 脚本自动化、日志记录 |

三者共享相同的代理引擎，区别只在展示方式。命令参数通用。

## 3. 正向代理 vs 反向代理

### 核心区别

- **正向代理** — 站在客户端一侧，代替客户端去访问服务。客户端需要知道代理的存在（需配置代理）。
- **反向代理** — 站在服务端一侧，代替服务端接受请求。客户端无感知，以为直接和真实服务通信。

```
正向：  客户端 → [代理] → 服务端
        客户端知道代理，服务端不知道

反向：  客户端 → [代理] → 服务端
        服务端知道代理，客户端不知道
```

**"正/反"指的是代理替谁干活：** 替客户端 = 正向，替服务端 = 反向。

### 选择依据

| | 反向代理 | 正向代理 |
|---|---|---|
| 服务端 | 需改端口 | 不改 |
| 客户端 | 不改 | 需设代理 |
| 适合场景 | 客户端多/不可控 | 服务端不可控 |

## 4. 反向代理用法

假设原服务在 23333 端口，将其改为 23334，mitmproxy 占住 23333：

```bash
# 三种工具任选其一
mitmproxy --mode reverse:http://localhost:23334 --listen-port 23333
mitmweb   --mode reverse:http://localhost:23334 --listen-port 23333
mitmdump  --mode reverse:http://localhost:23334 --listen-port 23333
```

客户端不用改，仍然访问 23333，mitmproxy 透明地记录并转发到 23334。

## 5. 正向代理用法

服务端不改，客户端通过代理发送请求：

```bash
# 三种工具任选其一
mitmproxy --listen-port 8080
mitmweb   --listen-port 8080
mitmdump  --listen-port 8080
```

### 客户端设置代理

#### PowerShell（设置环境变量）

```powershell
$env:http_proxy = "http://localhost:8080"
$env:https_proxy = "http://localhost:8080"
$env:no_proxy = ""
python your_script.py
```

> **重要：** `$env:no_proxy = ""` 必须设置，否则 Python 的 `requests`/`httpx` 会跳过 localhost 流量，不走代理。

#### CMD

```cmd
set http_proxy=http://localhost:8080
set https_proxy=http://localhost:8080
set no_proxy=
python your_script.py
```

> **注意：** PowerShell 中 `set` 是 `Set-Variable` 的别名，设的是 PowerShell 变量而非环境变量，Python 读不到。必须用 `$env:` 语法。

#### 恢复环境变量

关闭终端窗口即可（环境变量只在当前终端生效），或手动清除：

```powershell
# PowerShell
$env:http_proxy = ""
$env:https_proxy = ""
$env:no_proxy = ""
```

## 6. 正向代理原理

客户端代码中仍然写 `localhost:23333`，但设了 `http_proxy` 后，HTTP 库不会直连 23333，而是连接代理（8080），并在请求行中携带完整 URL：

```
GET http://localhost:23333/api HTTP/1.1    ← 完整 URL，代理据此转发
Host: localhost:23333
```

代理根据请求行中的完整 URL 知道该转发到哪里，拿到响应后再返回给客户端。

## 7. mitmweb 图形界面

执行 `mitmweb` 后会自动打开浏览器界面：

- **代理端口**（如 8080）— 处理流量
- **Web 界面端口**（默认 8081）— 查看抓包内容，地址 `http://127.0.0.1:8081`

Web 界面端口可自定义：

```bash
mitmweb --listen-port 8080 --web-port 9090
```

## 8. Windows 上的透明代理

Windows 没有 iptables，无法像 Linux 那样做透明代理（客户端和服务端都不改）。实际方案只有正向代理或反向代理二选一，选改动成本最低的一端。

## 9. 安装注意事项

```bash
pip install mitmproxy
```

mitmproxy 可能会安装不兼容的 `typing_extensions` 版本，导致其他库（如 `anthropic`）报错：

```
ImportError: cannot import name 'List' from 'typing_extensions'
```

解决方法：

```bash
pip install --upgrade typing_extensions
```
