# GM Pay（epusdt）与 new-api 同机部署

本文记录本项目在 `usdt.tokenaigate.com` 的独立部署与 new-api 钱包充值对接。示例假设 Ubuntu/Debian、宿主机 Nginx、Docker Compose，new-api 已占用宿主机 `3000`，项目位于 `/data/epusdt`。实际路径不同时替换示例值。**只使用本仓库的 `compose.production.yaml`；仓库旧的 `docker-compose.yaml` 把 `8000` 暴露到所有网卡，并挂载另一份 `./env`，不适合照搬到这一部署。**

## 1. 准备代码、端口和配置

确认域名 `usdt.tokenaigate.com` 的 A 记录指向本机、Docker Compose 与 Nginx 可用，并确认宿主机 `8000` 未被占用：

```bash
docker compose version
nginx -v
ss -lntp | grep -E ':3000|:8000' || true
cd /data/epusdt
```

已有部署目录时保留原有 `.env` 和 `data/`，不要重新初始化或覆盖。首次部署且 `.env` 不存在时：

```bash
cd /data/epusdt
if [ ! -f .env ]; then cp .env.example .env; fi
mkdir -p data
chmod 600 .env
```

本仓库 `.env.example` 已按 `compose.production.yaml` 设置 `app_uri=https://usdt.tokenaigate.com`、`http_listen=:8000`、`db_type=sqlite`、`sqlite_database_filename=/app/data/epusdt.db`、`runtime_root_path=/app/data/runtime` 和 `install=false`。上线前核对这些值。`./data:/app/data` 是持久化挂载：容器里的 `/app/data/epusdt.db` 对应宿主机 `/data/epusdt/data/epusdt.db`。配置文件是 `./.env:/app/.env:ro`；不要编辑 `src/.env.example` 来代替实际 `.env`，也不要把宿主机路径填进容器内路径字段。

```bash
docker compose -f compose.production.yaml config
docker compose -f compose.production.yaml up -d --build
docker compose -f compose.production.yaml ps
docker compose -f compose.production.yaml logs --tail=80 epusdt
curl -fsS http://127.0.0.1:8000/payments/gmpay/v1/config
```

生产 Compose 只把容器 `8000` 映射到宿主机 `127.0.0.1:8000`，不与 new-api 的 `3000` 冲突。公网只能经 Nginx 访问。公开配置接口返回可用链和收银台设置；若请求失败，先看容器日志，不要继续开启钱包充值。

## 2. Nginx、证书与 Cloudflare

宿主机 Nginx 可为 USDT 使用独立站点 `/etc/nginx/sites-available/usdt.tokenaigate.com`，不改 new-api 或 AliMPay 的站点。先建立 HTTP 站点供证书签发；已有同名站点时编辑原文件：

```nginx
server {
    listen 80;
    server_name usdt.tokenaigate.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_read_timeout 30s;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/usdt.tokenaigate.com /etc/nginx/sites-enabled/usdt.tokenaigate.com
nginx -t
systemctl reload nginx
```

若使用 Certbot 的 Nginx 插件，确认 HTTP 可达后签发并启用 HTTPS：

```bash
certbot --nginx -d usdt.tokenaigate.com
nginx -t
curl -fsS https://usdt.tokenaigate.com/payments/gmpay/v1/config
```

也可使用服务器已有的证书方案。Cloudflare 的 `usdt.tokenaigate.com` 记录可保持橙云代理；**SSL/TLS → 概述**应使用 **Full (strict)**，源站证书需覆盖该域名。不要让“缓存所有内容”规则缓存收银台、订单状态或 API 响应。支付回调若遭遇人机验证或 WAF 阻断，仅针对确切回调路径排查规则，不能凭 DNS 已解析就认定回调正常。若 Nginx 后面是 Cloudflare，真实客户端 IP 只能从可信 Cloudflare 来源还原；未配置真实 IP 前，不要按看到的代理 IP 贸然设置商户 Key 的 IP 白名单。

## 3. GM Pay 后台配置

完成管理员首次初始化并立即修改密码。不要把管理员密码、商户密钥、私钥或助记词写进本仓库。

1. **链与代币**：启用 `TRON` 与其下的 `USDT`，核对代币合约和精度。仅选 EPay 默认网络并不能自动启用链或资产。
2. **地址管理**：添加并启用你实际控制的 TRON 收款地址。历史沟通中提供的地址以 `TT1F...` 开头，后来后台截图中的地址以 `TLTa...` 开头；两者不同。必须先在自己的钱包核对归属，确认无误后再收款。只填写公开地址，绝不导入私钥或助记词。
3. **RPC 节点**：确认 TRON 的监听节点健康、链上扫描正常。能创建订单不代表链上到账已经可以自动识别。
4. **系统配置 → 支付配置**：法币使用 CNY，设置能取得 `USDT/CNY` 的汇率。自动模式下，汇率 API 地址应是接口目录；程序会请求其 `/cny.json`。如果状态为红色“需要关注”，点击刷新并查明最近一次错误，确认 USDT 汇率可用后再开放充值。金额精度 `2` 是项目默认值，单地址并发较高时可评估提高；已有待支付订单时不要随意修改匹配精度。
5. **系统配置 → EPay 配置**：默认网络 `TRON`、默认代币 `USDT`、默认法币 `CNY`，保存。new-api 发送 `type=usdt.tron` 时会直接选择该资产，但默认值也应正确。
6. **支付管理**：给 new-api 使用单独的商户 Key。当前截图中的 `tokenapigate` PID 为 `1001`；以实际后台显示值为准，将其密钥只填入 new-api。商户 Key 卡片的固定回调地址可留空，new-api 会在每笔订单提供 `notify_url`。未使用的 `default` Key 确认无其他依赖后再停用。

**Telegram、基础配置中的 Logo/背景和 OkPay 都不是自有钱包 TRON-USDT 收款所必需。** OkPay 保持关闭，避免把独立链上收款误接成托管收款。

## 4. new-api 连接与钱包展示

在 new-api **系统设置 → 计费与支付 → 支付网关 → epusdt** 填写：

| 字段 | 本部署的值 |
| --- | --- |
| 启用 epusdt USDT-TRC20 | 打开 |
| epusdt EPay 端点 | `https://usdt.tokenaigate.com/payments/epay/v1/order/create-transaction` |
| epusdt 商户 PID | `1001`，或实际专用 Key 的数字 PID |
| epusdt EPay 密钥 | 该 PID 对应的商户密钥 |

点 **保存所有设置**。端点只填到 `create-transaction`，**不要加 `/submit.php`**；new-api 使用的 EPay 客户端会补这个路径。new-api 的回调站点根地址应是 `https://tokenaigate.com`，而非 USDT 网关域名；本次回调路由为 `https://tokenaigate.com/api/user/epusdt/notify`。回调域名应可从 GM Pay 服务器通过 HTTPS 访问。

**“常规 → 付款方式”列表只需保留 AliMPay 的支付宝 `alipay`。** new-api 在合规确认有效、epusdt 开关打开、端点路径合法、数字 PID 和密钥均已配置时，会自动把 `USDT-TRC20` 加进钱包支付方式。手动在 `PayMethods` 添加 `usdt.tron` 不会替代这些条件；程序会过滤该手工项。注意“USDT”手动转账页签是另一套管理员核账流程，不是这次 GM Pay 自动监听与回调流程。

## 5. 上线验收与排错

用一笔可控的小额充值逐步核对：new-api 钱包出现 `USDT-TRC20` → 创建订单后 GM Pay 显示**自己的**收款地址、TRON 网络和准确的 USDT 数量 → 实际链上付款 → GM Pay 订单变为已支付 → EPay 异步通知成功 → new-api 钱包只入账一次。**浏览器跳转或链上转账成功都不能单独证明 new-api 已入账。** 确认整个流程成功后再对普通用户开放。

| 现象 | 优先核对 |
| --- | --- |
| 钱包只有支付宝 | new-api 的 epusdt 页签是否启用并保存；端点是否为指定 HTTPS 路径；PID 是否为数字；密钥是否已配置；合规确认是否有效 |
| 下单提示链未启用、资产不支持或无钱包 | GM Pay 的 TRON 链、USDT 资产、钱包地址是否已启用且合约正确 |
| 下单提示汇率错误 | `USDT/CNY` 是否有可用值；自动汇率最近一次刷新结果和日志 |
| 订单已支付而 new-api 未入账 | 订单通知投递记录、主站回调地址、Cloudflare/WAF、Nginx 转发与签名密钥是否一致 |
| 订单一直未支付 | TRON 节点及扫描状态、转账网络/代币/收款地址/实付金额是否与订单一致 |

EPay 请求与回调字段详见 [`wiki/API.md`](wiki/API.md)；new-api 的 epusdt 端点与钱包显示条件以对应版本代码为准。

## 6. 备份与更新

本部署的 SQLite 数据库、运行状态和日志在 `data/`，实际配置在 `.env`。更新前做一致性备份：

```bash
cd /data/epusdt
docker compose -f compose.production.yaml stop epusdt
umask 077
EPUSDT_BACKUP="/root/epusdt-backup-$(date +%F-%H%M%S).tgz"
tar -czf "$EPUSDT_BACKUP" data .env
chmod 600 "$EPUSDT_BACKUP"
docker compose -f compose.production.yaml up -d
```

确认备份可恢复后再更新代码并重建容器；保留现有 `.env` 和 `data/`。检查容器日志、本机公开配置接口、HTTPS 接口，并重新跑一笔小额支付的回调验收。备份包含数据库与配置，限制文件访问；不要提交到 Git。
