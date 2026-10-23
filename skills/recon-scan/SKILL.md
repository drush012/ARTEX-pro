---
name: recon-scan
description: 对已授权目标做资产发现与已知漏洞检测时调用该 skill（subfinder / httpx / katana / nuclei）。
---

# Recon & Detection（资产发现 + 已知漏洞检测）

在**已授权**前提下，用标准开源工具完成：子域/资产发现、存活与指纹探测、端点爬取、
**已知漏洞 / 暴露 / 错误配置的检测**。

本 skill **只做发现与检测，不做利用（exploit）**。利用、后渗透、内网等阶段不在本 skill 范围，
由人工在授权环境处置。

---

## 边界与禁止（Agent 必读 · 违反即越界）

| 范围 | 允许 | 禁止 |
|---|---|---|
| **目标** | 仅对**任务 scope 内、已授权**的资产 | 越界资产、未授权目标、无差别批量打击 |
| **动作** | 枚举、探测、爬取、模板化**检测** | 投递利用 payload 拿权限、改状态、破坏性操作 |
| **nuclei** | `-severity` 过滤的检测/信息类模板 | 侵入性利用类模板 → 须经审批门（intercept/guard）人工确认后再跑 |
| **速率** | 遵守目标速率、尊重 scope | 爆破、DoS、高频压测 |

- **所有 HTTP 必须走录制代理**（`-proxy http://127.0.0.1:8788`，或设 `HTTP_PROXY`），
  否则命中没有流量证据，等于白打。
- 发现的服务/端点一律 `insert_assets`；确认的漏洞一律落 finding，**不得遗漏**。
- 工具只做授权检测；拿到“疑似可利用”的点，**记录并交人工**，不要在本 skill 里自动利用。

---

## 工具与用法（按序，前一步喂后一步）

### 1. 子域 / 资产发现 — subfinder
```bash
subfinder -d <domain> -silent -o OUTDIR/subs.txt
```
被动为主，产出喂资产图（补 enrich 的被动 DNS）。

### 2. 存活 / 指纹 — httpx
```bash
httpx -l OUTDIR/subs.txt -silent -title -tech-detect -status-code \
      -proxy http://127.0.0.1:8788 -o OUTDIR/live.txt
```
分流存活资产、识别技术栈，缩小检测面。

### 3. 端点爬取 — katana
```bash
katana -u <url> -jc -proxy http://127.0.0.1:8788 -o OUTDIR/endpoints.txt
```
JS 感知爬取，补充 `api-recon` 的端点发现。

### 3b. 被动历史 URL — gau
```bash
gau <domain> --threads 5 | sort -u > OUTDIR/gau-urls.txt
```
从 wayback / CommonCrawl / CT 拉历史 URL（**纯被动，不碰目标**），补端点与参数面。

### 4. 已知漏洞 / 暴露检测 — nuclei
```bash
nuclei -l OUTDIR/live.txt -severity low,medium,high,critical \
       -proxy http://127.0.0.1:8788 -o OUTDIR/nuclei.txt
```
模板化检测 CVE / 暴露 / 错误配置。**侵入性模板经审批门确认后再跑**；
默认优先信息性与检测类模板。

### 4b. XSS 检测 — dalfox
```bash
dalfox url "<带参数的 url>" --proxy http://127.0.0.1:8788 -o OUTDIR/xss.txt
# 批量: dalfox file OUTDIR/endpoints.txt --proxy http://127.0.0.1:8788
```
参数化 XSS **检测**（确认漏洞是否存在，附 PoC 证明）。命中落 finding + 流量证据；
**不做进一步利用 / 持久化 / 窃取**——确认即止,交人工。

### 5. 密钥 / 暴露扫描 — trufflehog
对**已下载的**前端 bundle / source map / 配置文件做密钥检测（自动化手工翻 `env.js`、`.map` 找 key 的活）：
```bash
trufflehog filesystem OUTDIR/js --only-verified --json > OUTDIR/secrets.json
```
**判性质（吃过的教训）**：扫出的 key 先分类 —— **公开客户端 key**（Sentry DSN / PostHog / Mapbox / 分析 key）绝大多数**不计分**；只有**带权限、能横向利用**的硬编码凭证才有价值。
命中只**记录位置（文件+变量名）与类型**，**不实际使用、不验证有效性、不外传**（凭证红线）。

### 6. 隐藏参数发现 — arjun
```bash
arjun -u "<endpoint>" -oJ OUTDIR/params.json
```
对已知端点挖隐藏/未文档化的参数，补 `api-recon` 的参数面。**只发现参数名，不做注入/漏洞测试**。

### 7. 被动情报 — shodan（FOFA/Censys 同理）
```bash
shodan host <ip>            # 某 IP 的开放服务/历史（需 SHODAN_API_KEY）
shodan search "ssl:<域名>"   # 关联资产
```
**纯被动查第三方情报库，不碰目标本身** —— 扩资产面、定位边缘资产。FOFA/Censys 走各自 API+key，同理。
**JWT 分析**：遇到 Bearer/JWT 直接**内联 base64 解码**看 claim/alg（我们 Wolt 那样），不需额外工具。

---

## 产出

- 发现的服务 / 端点资产 → `insert_assets`（不漏）
- 确认的漏洞 → finding（附 **nuclei 模板 id + 原始请求 + 流量记录 id**，三者齐全才算证据完整）
- 每条结论标注：检测工具、置信度、是否需人工复核
- 与 `api-recon` 衔接：katana/httpx 的端点可回灌给 `api-recon` 做参数面深挖

---

## 与现有能力的关系

| 已有 | 本 skill 补什么 |
|---|---|
| nmap（端口/服务） | 子域与存活面（subfinder/httpx）大幅扩侦察面 |
| api-recon（前端接口逆向） | katana JS 爬取补端点；httpx 指纹定位该用哪套方法 |
| 浏览器/手搓 curl | nuclei 模板化**已知漏洞检测**，覆盖 curl 手测覆盖不到的面 |

**一句话**：本 skill 把“资产发现 + 已知漏洞检测”标准化、规模化，但严守
“只检测、不利用、只打授权 scope、全程留证”的边界。
