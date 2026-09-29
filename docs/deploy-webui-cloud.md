# 云服务器 Web 界面访问指南

如果你已经把项目部署到云服务器，但不知道在浏览器里输入什么地址才能打开 Web 管理界面，这篇教程就是为你准备的。

> 其实就两步：让服务监听外网，再在浏览器里输入地址。

---

## 目录

- [方式一：直接部署（pip + python）](#方式一直接部署pip--python)
- [方式二：Docker Compose](#方式二docker-compose)
- [如何在浏览器里打开界面](#如何在浏览器里打开界面)
- [如何确认 Docker 重建已生效](#如何确认-docker-重建已生效)
- [访问不了？先检查这几项](#访问不了先检查这几项)
- [可选：Nginx 反向代理（绑定域名 / 80 端口）](#可选nginx-反向代理绑定域名--80-端口)
- [安全建议](#安全建议)

---

## 方式一：直接部署（pip + python）

### 第一步：修改 .env 中的监听地址

用编辑器打开 `.env`（在项目根目录，即包含 `main.py` 的目录），找到这一行：

```env
WEBUI_HOST=127.0.0.1
```

把 `127.0.0.1` 改成 `0.0.0.0`：

```env
WEBUI_HOST=0.0.0.0
```

> `127.0.0.1` 表示只有本机能访问，`0.0.0.0` 表示允许任何来源访问。云服务器必须改成 `0.0.0.0` 才能从外网打开界面。

> **注意**：`.env` 里的 `WEBUI_HOST` 优先级高于命令行参数。所以即使你在命令里加了 `--host 0.0.0.0`，如果 `.env` 里还是 `127.0.0.1`，外网照样访问不了。请务必先改 `.env`。

### 第二步：启动服务

在项目根目录执行：

```bash
# 只启动 Web 界面（不自动执行分析）
python main.py --webui-only

# 或者：启动 Web 界面（启动时执行一次分析；需每日定时分析请加 --schedule 或设 SCHEDULE_ENABLED=true）
python main.py --webui
```

启动成功后，终端会输出类似：

```
FastAPI 服务已启动: http://0.0.0.0:8000
```

如果你想让服务在退出终端后继续运行，可以用 `nohup`：

```bash
nohup python main.py --webui-only > /dev/null 2>&1 &
```

> 日志文件会由程序自动写入 `logs/` 目录，用 `tail -f logs/stock_analysis_*.log` 查看。

### 修改端口（可选）

默认端口是 8000。如果想改用其他端口，在 `.env` 里设置：

```env
WEBUI_PORT=8888
```

然后重启服务。

---

## 方式二：Docker Compose

### 第一步：确认已有 .env 配置

项目的 `docker/docker-compose.yml` 在容器内部已经自动设置了 `WEBUI_HOST=0.0.0.0`，你不需要在 `.env` 里再改监听地址，Docker 会自动处理。

### 第二步：启动服务

在项目根目录执行：

```bash
# 同时启动定时分析 + Web 界面（推荐）
docker-compose -f ./docker/docker-compose.yml up -d

# 或者只启动 Web 界面服务
docker-compose -f ./docker/docker-compose.yml up -d server
```

启动后查看状态：

```bash
docker-compose -f ./docker/docker-compose.yml ps
```

看到 `server` 服务状态为 `running` 就说明 Web 界面已经在运行了。

### 修改端口（可选）

默认端口是 8000。如果想改用其他端口，在 `.env` 里设置：

```env
API_PORT=8888
```

然后重新启动容器：

```bash
docker-compose -f ./docker/docker-compose.yml down
docker-compose -f ./docker/docker-compose.yml up -d
```

---

## 如何在浏览器里打开界面

服务启动后，在浏览器地址栏输入：

```
http://你的服务器公网IP:8000
```

例如，如果你的服务器 IP 是 `1.2.3.4`，就输入：

```
http://1.2.3.4:8000
```

如果你的域名已经解析到这台服务器，也可以直接用域名访问：

```
http://your-domain.com:8000
```

> **在哪里查公网 IP？** 登录你的云服务器控制台（阿里云/腾讯云/AWS 等），在实例列表里可以看到「公网 IP」或「弹性 IP」。

---

## 如何确认 Docker 重建已生效

WebUI 现在会在“系统设置”页展示只读的“版本信息”卡片，包含：

- `WebUI 版本`
- `构建标识`
- `构建时间`

如果 `apps/dsa-web/package.json` 里的版本号仍是占位值 `0.0.0`，页面会自动回退展示本次前端构建生成的 `构建标识`，避免你误把占位版本当成真实发布版本。

当你重新执行 `docker-compose -f ./docker/docker-compose.yml up -d --build`，或者单独重新执行前端 `npm run build` 后，可以刷新浏览器并进入“系统设置”，优先确认“构建时间”是否已经变化；若变化，通常就说明当前加载的静态资源已经切换到最新构建。

在确认本地前端打包链路时，建议执行以下命令用于本次改动的最小验证闭环：

```bash
cd apps/dsa-web
npm ci
npm run lint
npm run build
```

其中 `build` 成功后，`static` 下生成的 `index.html`/JS/CSS 资源会包含本次构建时间与构建版本信息；刷新后在“版本信息”卡片中应能见到变化。

---

## 访问不了？先检查这几项

### 1. 安全组 / 防火墙没有放行端口

这是最常见的原因。云服务器默认只开放 22（SSH）端口，需要手动放行 8000（或你改的端口）。

**操作方法**（以阿里云为例）：
1. 登录阿里云控制台 → 云服务器 ECS → 找到你的实例
2. 点击「安全组」→「配置规则」→「添加安全组规则」
3. 方向选「入方向」，端口范围填 `8000/8000`，授权对象填 `0.0.0.0/0`，点击「确定」

腾讯云、AWS 等云厂商操作类似，找到「安全组」或「防火墙规则」，新增一条允许 TCP 8000 端口的入站规则即可。

### 2. 服务器系统防火墙拦截了

如果你的系统开启了 `ufw` 或 `firewalld`，也需要放行端口：

```bash
# Ubuntu / Debian（ufw）
sudo ufw allow 8000

# CentOS / RHEL（firewalld）
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --reload
```

### 3. 直接部署时 .env 里的 WEBUI_HOST 没改

这是第二常见原因。`.env` 里默认是 `WEBUI_HOST=127.0.0.1`，这样服务只监听本机，外网根本连不上。

改法：打开 `.env`，把 `WEBUI_HOST=127.0.0.1` 改成 `WEBUI_HOST=0.0.0.0`，然后重启服务。

> Docker 方式不需要改这个，可以跳过。

### 4. 端口号对不上

检查访问地址里的端口是否和 `.env` / 启动命令里设置的端口一致。

- 直接部署：默认 8000，可通过 `WEBUI_PORT=xxxx` 修改
- Docker：默认 8000，可通过 `API_PORT=xxxx` 修改

### 5. 页面能打开，但 UI 元素异常变大 / 布局错乱

**症状**：浏览器能访问到 8000 端口，页面有内容，但文字、按钮、卡片尺寸异常大，没有正常布局与配色。

**根因**：`static/index.html` 存在但 CSS/JS 资源缺失（`static/assets/` 为空或不存在），浏览器加载了 HTML 框架但无法拿到样式与脚本，退化为裸 HTML 渲染。

可先用浏览器开发者工具（F12 → Network 标签页）检查是否有 `/assets/index-*.js`、`/assets/index-*.css` 的 **404** 错误。若有，按以下方式修复：

**Docker 用户**：

```bash
docker-compose -f ./docker/docker-compose.yml down
docker-compose -f ./docker/docker-compose.yml build --no-cache
docker-compose -f ./docker/docker-compose.yml up -d
```

重建完成后，用 `Ctrl+Shift+R` 强制刷新浏览器缓存，再访问页面。

**直接部署用户**：先确保已安装 Node.js 18+（推荐 20+），然后手动构建前端：

```bash
cd apps/dsa-web
npm ci
npm run build
cd ../..
python main.py --webui-only
```

---

## 可选：Nginx 反向代理与安全加固（配合 Cloudflare / 隐藏真实 IP 与端口）

如果你有域名，或者配置了 Cloudflare（CF）等 CDN/代理，建议通过 Nginx 做反向代理，并严格阻断公网通过 IP 和端口直接访问源站。

### 1. 避免通过 IP:8000 端口直接访问

在默认配置下，Docker 可能会直接在宿主机公网 `0.0.0.0:8000` 监听。由于 Docker 直接操作 Linux iptables，常规 UFW 规则可能被绕过。

**正确做法**：
1. **绑定本地回环**：在 `.env` 中设置 `API_BIND_IP=127.0.0.1`（或者直接修改 `docker/docker-compose.yml` 中的 ports 为 `127.0.0.1:8000:8000`），然后重新启动容器：
   ```bash
   docker-compose -f ./docker/docker-compose.yml down
   docker-compose -f ./docker/docker-compose.yml up -d
   ```
2. **云服务器安全组**：在阿里云/腾讯云/AWS 等控制台的安全组规则中，**彻底删除或关闭 8000 端口**的入方向放行规则。

这样外部网络将完全无法连接 8000 端口，仅宿主机本地的 Nginx 可以访问该端口。

### 2. Nginx 配置示例（禁止 IP 直接访问 + 支持 WebSocket + 还原 CF 真实 IP）

新建或编辑 `/etc/nginx/conf.d/stock-analyzer.conf`：

```nginx
# ----------------------------------------------------
# 1. 默认服务器：拦截所有直接使用 IP 或未授权域名访问的请求
# ----------------------------------------------------
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444; # 444 会直接关闭 TCP 连接且不返回任何数据，有效防御恶意扫描
}

server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;
    # 需要提供证书（可使用自签名哑证书），直接丢弃连接
    ssl_certificate /etc/nginx/ssl/dummy.crt;
    ssl_certificate_key /etc/nginx/ssl/dummy.key;
    return 444;
}

# ----------------------------------------------------
# 2. 正常业务服务：仅响应指定域名
# ----------------------------------------------------
server {
    listen 80;
    # 若在源站配置了 SSL 证书（如 Cloudflare Origin CA），可开启 443：
    # listen 443 ssl http2;
    server_name your-domain.com; # 替换为你自己的域名

    # 若使用 HTTPS：
    # ssl_certificate /path/to/origin-cert.pem;
    # ssl_certificate_key /path/to/origin-key.key;

    # 还原 Cloudflare 代理的访客真实 IP
    real_ip_header CF-Connecting-IP;
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 141.101.64.0/18;
    set_real_ip_from 108.162.192.0/18;
    set_real_ip_from 190.93.240.0/20;
    set_real_ip_from 188.114.96.0/20;
    set_real_ip_from 197.234.240.0/22;
    set_real_ip_from 198.41.128.0/17;
    set_real_ip_from 162.158.0.0/15;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 131.0.72.0/22;
    set_real_ip_from 2400:cb00::/32;
    set_real_ip_from 2606:4700::/32;
    set_real_ip_from 2803:f800::/32;
    set_real_ip_from 2405:b500::/32;
    set_real_ip_from 2405:8100::/32;
    set_real_ip_from 2a06:98c0::/29;
    set_real_ip_from 2c0f:f248::/32;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 支持 WebSocket（Agent 对话与实时事件流）
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # 延长分析超时时间（防止长耗时分析或 SSE 流被 Nginx 中断）
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}
```

### 3. 启用配置并测试

```bash
sudo nginx -t            # 检查配置语法
sudo systemctl reload nginx
```

### 4. 推荐方案：Cloudflare Tunnel（零开放入方向端口）

如果追求极致安全，推荐使用 **Cloudflare Tunnel（cloudflared）**：
- 云服务器**不需要在安全组中开放 80、443、8000 中的任何一个入方向端口**（可实现全关）。
- 本地启动 `cloudflared` 守护进程与 Cloudflare 边缘建立安全出站连接，流量直接内网转发到 `http://127.0.0.1:8000`。
- 彻底隐匿源站 IP，从物理网络层面免疫针对源站 IP 的扫描和直接攻击。

> **注意事项**：
> - 配合 Nginx / CF 代理并启用 Web 登录认证时，务必在 `.env` 中设置 `TRUST_X_FORWARDED_FOR=true`，以确保防暴力破解限流能正确拿到访客真实 IP。
> - Cloudflare 控制台中的 SSL/TLS 模式建议设置为 **Full** 或 **Full (strict)**，并在「边缘证书」中开启「始终使用 HTTPS」。

---

## 安全建议

把 Web 界面暴露到公网之前，强烈建议开启登录密码保护：

在 `.env` 中设置：

```env
ADMIN_AUTH_ENABLED=true
```

重启服务后，第一次访问网页时会要求设置初始密码。设置完成后，每次打开设置页面都需要输入密码，可以防止 API Key 等敏感配置被他人看到。

> 如果忘了密码，可以在服务器上执行：`python -m src.auth reset_password`

---

遇到其他问题？欢迎 [提交 Issue](https://github.com/ZhuLinsen/daily_stock_analysis/issues)。
