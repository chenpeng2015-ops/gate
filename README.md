# VPNGate SSTP 家宽节点（edgetunnel 链式代理） 🚀

自动抓取 [VPN Gate](https://www.vpngate.net/) 的 SSTP 家宽/机房节点，逐个验证可用性，按国家分组、标注住宅/机房，生成可直接通过 **URL 自动轮换** 的节点清单。**每 6 小时自动更新一次。**

> 核心价值：VPN Gate 的 SSTP 节点换得很快，手动测试筛选太痛苦。本仓库把它全自动了——你只需把 `nodes.txt` 的网址填进 edgetunnel 后台一次，之后节点每 6 小时自动换，零手动。

> **本 fork 与上游最大的不同：检测不再走 Cloudflare Worker。** 检测代码（`worker.js`，源自 CheckSocks5）原样放在仓库里，工作流运行时在 GitHub 的临时机器上用 workerd 本地跑起来，主程序调本地地址检测，跑完机器就销毁。**不需要部署检测 Worker，不需要配任何 Secret，也就不耗 Cloudflare 的 CPU 额度。** 代理本身（cfnext、edgetunnel 那些）仍在 Cloudflare 上跑，那是另一回事。

---

## 架构（数据流向）

```text
VPN Gate 官方源（失效时回退 GitHub 镜像）
      │  （每 6 小时，GitHub Actions 定时抓取）
      ▼
筛选 SSTP 节点 → 去重
      │
      ▼
本地检测 Worker（worker.js，workerd 本地运行）
      │  GitHub 临时机器上：wrangler dev → http://127.0.0.1:8787
      │  GET /check?sstp=vpn:vpn@host:port
      │  SSTP 握手 + PPP/PAP 登录 → 隧道内查出口 IP（住宅/机房判定）
      ▼
保留成功节点 → 按国家分组 → 住宅/机房标注 → 延迟排序
      │
      ▼
生成 nodes.txt + data.json（GitHub Pages 发布）
      │
      ▼
edgetunnel 后台「自定义优选IP」框填 https://…/nodes.txt
      │  edgetunnel 每次生成订阅时自动 fetch → 解析 $sstp:// → 套链式代理
      ▼
客户端订阅 edgetunnel 订阅 → 使用 SSTP 家宽节点（每 6 小时自动换）
```

---

## 仓库文件

| 文件 | 作用 |
| :--- | :--- |
| `vpngate.py` | 主程序：抓取、筛选、检测、生成页面数据 |
| `worker.js` | 检测代码（原样，未改动逻辑） |
| `wrangler.toml` | 本地跑检测 Worker 的配置（端口 8787） |
| `.github/workflows/check.yml` | 工作流：装 Python + Node 22，本地起 Worker，跑检测，部署 Pages |
| `web/index.html` | Pages 前端模板 |
| `requirements.txt` | Python 依赖 |

---

## 一、部署教程（从零开始）

### 前置条件

- 一个 GitHub 账号（不需要 Cloudflare 账号来做检测）
- 一个已部署好的 edgetunnel（核心使用端，见下文第 2 步）

### 第 1 步：Fork 本仓库

在 GitHub 上打开本仓库，点 **Fork**，复制到你账号下。

### 第 2 步：改两个地址（fork 后必改）

| 文件 | 位置 | 改成什么 |
| :--- | :--- | :--- |
| `vpngate.py` | `NODES_URL` | 换成你自己的 Pages 地址，形如 `https://你的GitHub用户名.github.io/仓库名/nodes.txt` |
| `.github/workflows/check.yml` | 最后一步输出的站点地址 | 同样换成你自己的 Pages 地址 |

> **不需要**配置检测 Worker 域名，也**不需要**在仓库 Secrets 里加任何东西。旧版的 `DOMAIN` Secret 在本 fork 里已经废弃，删掉也不影响运行。
>
> **不需要**在 `vpngate.py` 里配置 edgetunnel 的 UUID 和节点域名，那是 edgetunnel 自己的配置，`vpngate.py` 只负责生成 `nodes.txt`。

### 第 3 步：开启 GitHub Pages 与 Actions

1. 进你 fork 的仓库 → Settings → Pages，Source 设为 **GitHub Actions**
2. 进 Actions 页，若提示启用 Actions 就点启用
3. 手动触发一次：Actions → VPN Gate Node Check → Run workflow → Run workflow
4. 等它跑完（约 2 分钟），看到绿色 ✓ 即成功

### 第 4 步：确认产物

跑完后，你的站点地址是：

```text
https://你的GitHub用户名.github.io/仓库名/nodes.txt
```

浏览器打开，能看到一堆 `入口地址:443#国家-住宅-XX$sstp://vpn:vpn@节点:端口` 的行，就说明全部打通了。

---

## 二、使用教程（URL 自动轮换，一次配置永久生效）

1. 进 edgetunnel 后台（你的域名/admin），找到「自定义优选IP」文本框
2. 粘贴**一行网址**：

   ```text
   https://你的GitHub用户名.github.io/仓库名/nodes.txt
   ```

3. 点保存
4. 客户端刷新订阅 → 每次刷新 edgetunnel 都重新拉取一次 nodes.txt，节点自动更新

> 原理：`nodes.txt` 是纯节点行版本（无注释头），每行 `入口地址:443#国家-住宅-01$sstp://vpn:vpn@节点:端口`。edgetunnel 下次生成订阅时会 fetch 这个网址、逐行解析成优选入口 + 链式代理指令。你只填一次，之后节点每 6 小时自动换、零手动。

---

## 三、如何更换优选入口

入口地址决定客户端连 Cloudflare 用哪个 IP/域名、稳不稳。程序**不会**自己测入口速度、也不会自动选最优，它是按顺序轮流分配的，所以池子里只放实测过的好地址最重要。

### 方式一：改代码里的静态池（默认池）

- 文件：`vpngate.py`
- 位置：`EDGE_HOSTS = [ ... ]`，默认 31 条
- 改法：用 CloudflareSpeedTest 这类工具在你自己网络上实测，挑「延迟低 + 实际能连通」的，换成 `IP:端口` 或 `域名:端口`，逗号分隔。**端口必须写**，只写 IP 不写端口是不行的。已经死了的入口（比如 `saas.sin.fan:443`）要删掉，不然轮到它的节点就连不上。

### 方式二：用环境变量覆盖（不用改代码）

- 文件：`.github/workflows/check.yml`
- 位置：跑主程序那一步的 `env` 里，在 `CHECK_TIMEOUT` 下面加一行：

  ```yaml
  HOSTS_ENTRY: "104.21.5.6:443,172.67.10.20:443"
  ```

- 注意两点：
  - 格式和方式一完全一样，`IP:端口`，逗号分隔，端口必须写，引号别漏。
  - `HOSTS_ENTRY` 是**整体替换，不是追加**：设了它，代码里默认的 31 条就全被忽略了，只剩你写的这几条 + 优选 API 来的地址。空字符串（`""`）等于没设，会回到默认池。

### 优选 API（自动追加，不用动）

主程序还会自动从优选 API 拉一批地址，追加到静态池后面（目前是 `cf.090227.xyz` 的电信/联通/移动三条，分别返回 6/4/8 条）。这部分在 `vpngate.py` 的 `OPTIMAL_API` 里，API 返回的格式是 `IP:端口#备注`，程序会自动补端口。

---

## 四、配置速查表

| 配置 | 位置 | 说明 |
| :--- | :--- | :--- |
| 定时 | `.github/workflows/check.yml` 的 `schedule` | 每 6 小时跑一次（`0 */6 * * *`），手动触发是 `workflow_dispatch` |
| Node 版本 | 同上，`Setup Node` | **必须是 22 或以上**，最新 wrangler 要求 Node ≥ 22，写 20 会直接失败 |
| `CHECK_WORKER` | 同上，跑主程序的 `env` | 本地地址 `http://127.0.0.1:8787/check?sstp=vpn:vpn@`，不要改成远程 Worker 域名 |
| `CHECK_CONCURRENCY` | 同上 | 检测并发数，默认 32 |
| `CHECK_TIMEOUT` | 同上 | 单次检测超时（秒），默认 90；Worker 内部另有 12 秒上限 |
| `EDGE_HOSTS` | `vpngate.py` | 默认静态入口池（31 条） |
| `HOSTS_ENTRY` | `check.yml` 的 `env` | 高优先级覆盖池，设了就忽略 `EDGE_HOSTS`，空字符串等于没设 |
| `OPTIMAL_API` | `vpngate.py` | 优选 IP 的 API 地址，自动追加 |
| `NODES_URL` | `vpngate.py` | fork 后改成你自己的 Pages 地址 |

---

## 五、常见问题

### 只有几个节点能连

入口优选地址大部分被墙或已死。用 CloudflareSpeedTest 重新实测，把 `EDGE_HOSTS` 或 `HOSTS_ENTRY` 换成实测能通的地址。

### 全部 -1

检查：edgetunnel 是否部署好、域名是否解析到 Cloudflare、UUID 是否填对、传输协议是否对得上；再看 Actions 日志里检测成功数是不是 0。

### 6 小时没更新

到 Actions 页看最近一次运行是否成功、定时任务是否还在（`schedule` 有没有被注释掉）。

### 本地 Worker 没启动（`local worker did not start`）

先看 Node 版本是不是 22 或以上，再看那一步的 `wrangler.log`。Node 20 会直接报 `Wrangler requires at least Node.js v22.0.0`，改成 22 就行。

### 检测成功数和之前差很多

正常。VPN Gate 每天的节点池本身就在变（比如一次是 98 个原始节点、78 个 SSTP、35 个可用，另一次是 87 个、38 个可用），只要国家数、住宅/机房比例大致对得上就没问题。

### 上游更新会不会覆盖我的 fork？

不会。fork 是独立仓库，上游更新只更新上游自己。只有你自己点 `Sync fork` 才会把上游更新合并进来，正常同步是合并不是覆盖，有冲突会让你手动解决；只有主动选“丢弃自己的改动”才会被覆盖。

---

*流水线：GitHub Actions（每 6 小时 cron） → vpngate.py → 本地 workerd 检测 → GitHub Pages*

---

## 引用与致谢

本项目的实现离不开以下开源项目和服务的支持，在此表示衷心的感谢：

| 项目 | 用途 | 链接 |
| :--- | :--- | :--- |
| **cmliu/edgetunnel** | VLESS 代理 + 链式代理，节点最终通过它使用 | https://github.com/cmliu/edgetunnel |
| **CheckSocks5 检测代码** | 检测逻辑来源：验证 SSTP 节点可用性并读取出口 IP，本 fork 将其原样放在仓库里本地运行，不再部署到 Cloudflare | https://github.com/lsh8848/cm-Workers-CheckSocks5 |
| **fdciabdul/Vpngate-Scraper-API** | VPN Gate 节点数据的 GitHub 镜像（官方源失效时回退） | https://github.com/fdciabdul/Vpngate-Scraper-API |
| **VPN Gate** | SSTP 节点数据源 | https://www.vpngate.net/ |
| **Star History** | 提供项目热度曲线图生成服务 | https://star-history.com/ |

---

## 项目热度

[![Star History Chart](https://api.star-history.com/svg?repos=hezhanleiok/gate&type=Date)](https://star-history.com/#hezhanleiok/gate&Date)

---

**特别感谢**：
感谢所有为开源社区做出贡献的开发者们！没有你们的无私奉献，就没有这个项目的诞生。也感谢每一位使用、测试和反馈问题的用户，是你们的支持让这个项目不断完善。
