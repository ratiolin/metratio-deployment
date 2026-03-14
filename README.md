# 部署与运维文档

## 一、文档说明

### 1.1 文档目的
本文档旨在记录当前服务器环境的部署架构、配置信息和运维要点，作为个人技术尝试的总结与备份，方便日后回顾或重建。

### 1.2 安全声明
- 所有敏感信息（密码、API密钥、Token等）已用 `[REDACTED]` 替换，实际值保存在安全位置（如 `.env` 文件）。

## 二、服务器基础环境

### 2.1 操作系统
- **发行版**：OpenCloudOS 5.4.241-30.0017.19（x86_64）
- **内核版本**：Linux VM-0-8-opencloudos 5.4.241-30.0017.19 #1 SMP
- **架构**：x86_64 (64位)
- **主机名**：VM-0-8-opencloudos
- **时区**：CST（中国标准时间）

### 2.2 用户与权限
| 用户名 | UID | GID | 主目录 | Shell | 用途 |
|--------|-----|-----|--------|-------|------|
| root | 0 | 0 | /root | /bin/bash | 超级用户 |
| ratio | 1001 | 1001 | /home/ratio | /bin/bash | 部署与运维用户 |

### 2.3 账号权限明细

| 账号/密钥 | 类型 | 用途 | 权限范围 | 存储位置 |
|-----------|------|------|----------|----------|
| ratio | 系统用户 | 日常运维 | sudo、docker | 仅本人知晓 |
| postgres | 数据库用户 | Mattermost 数据库 | superuser | `/srv/stack/.env` 中 `POSTGRES_PASSWORD` |
| deepseek-api | API密钥 | OpenClaw 模型调用 | 仅限该服务 | `/srv/data/openclaw/config/openclaw.json` |
| auth-server SMTP | 邮箱账号 | 发送认证邮件 | 仅限发件 | `/srv/stack/.env` 中 `SMTP_PASS` |
| SSL证书 | Let's Encrypt | HTTPS | N/A | `/srv/stack/nginx/ssl/` |


### 2.4 Docker环境
- **Docker版本**：20.10.24+dfsg1-1+deb12u1+b3
- **Docker网络**：shared-net (172.24.0.0/16)
- **存储驱动**：overlay2
- **日志驱动**：json-file（默认，建议设置日志轮转）

### 2.5 网络与防火墙
- **开放端口**（firewalld 配置）：
  - 22/tcp（SSH）
  - 80/tcp（HTTP）
  - 443/tcp（HTTPS）
- **iptables**：启用，包含Docker网络规则和腾讯云安全策略
- **SELinux**：禁用

## 三、目录结构

### 3.1 核心部署目录
```
/srv/
├── data/
│   ├── openclaw/
│   │   └── config/              # OpenClaw配置文件（挂载至容器 /home/node/.openclaw）
│   ├── mattermost/
│   │   ├── db/                   # PostgreSQL数据（挂载至容器 /var/lib/postgresql/data）
│   │   ├── plugins/               # Mattermost服务端插件
│   │   ├── client-plugins/        # 客户端插件
│   │   ├── bleve-indexes/         # 搜索索引
│   │   ├── files/                 # 上传文件
│   │   ├── config/                # 应用配置
│   │   └── logs/                  # 应用日志
│   └── grafana/                   # Grafana数据
├── stack/
│   ├── nginx/
│   │   ├── conf.d/                # Nginx站点配置（mattermost.conf, openclaw.conf）
│   │   ├── ssl/                    # SSL证书（fullchain.pem, privkey.pem）
│   │   ├── nginx.conf              # 主配置文件
│   │   └── metratio-static/        # 静态文件
│   └── observability/
│       ├── prometheus.yml          # Prometheus配置
│       ├── alert.rules.yml         # 告警规则
│       └── loki-config.yaml        # Loki配置
├── data/prometheus/                # Prometheus时序数据
└── data/loki/                      # Loki日志存储
```

### 3.2 容器挂载点映射
| 容器名称 | 宿主机路径 | 容器内路径 | 说明 |
|----------|------------|------------|------|
| openclaw-gateway | /var/run/docker.sock | /var/run/docker.sock | Docker套接字 |
| | /proc | /host/proc | 只读主机进程信息 |
| | /usr/local/bin/ops | /usr/local/bin/ops | 运维命令 |
| | /srv/data/openclaw/config | /home/node/.openclaw | OpenClaw配置 |
| | /srv/data/openclaw/workspace | /home/node/.openclaw/workspace | 工作空间 |
| mm-postgres | /srv/data/mattermost/db | /var/lib/postgresql/data | 数据库数据 |
| mattermost | /srv/data/mattermost/plugins | /mattermost/plugins | 服务端插件 |
| | /srv/data/mattermost/client-plugins | /mattermost/client/plugins | 客户端插件 |
| | /srv/data/mattermost/bleve-indexes | /mattermost/bleve-indexes | 搜索索引 |
| | /srv/data/mattermost/files | /mattermost/data | 上传文件 |
| | /srv/data/mattermost/config | /mattermost/config | 配置文件 |
| | /srv/data/mattermost/logs | /mattermost/logs | 应用日志 |
| gateway-nginx | /srv/stack/nginx/nginx.conf | /etc/nginx/nginx.conf | 主配置（只读） |
| | /srv/stack/nginx/conf.d | /etc/nginx/conf.d | 站点配置（只读） |
| | /srv/stack/nginx/ssl | /etc/nginx/ssl | SSL证书（只读） |
| | /srv/stack/nginx/metratio-static | /usr/share/nginx/html | 静态文件 |
| grafana | /srv/data/grafana | /var/lib/grafana | Grafana数据 |
| prometheus | /srv/stack/observability/prometheus.yml | /etc/prometheus/prometheus.yml | Prometheus配置 |
| | /srv/stack/observability/alert.rules.yml | /etc/prometheus/alert.rules.yml | 告警规则 |
| | /srv/data/prometheus | /prometheus | 时序数据 |
| loki | /srv/stack/observability/loki-config.yaml | /etc/loki/loki-config.yaml | Loki配置 |
| | /srv/data/loki | /loki | 日志存储 |

## 四、核心部署架构

### 4.1 服务组成表格
| 服务名称 | 镜像 | 详细版本 | 容器端口 | 宿主机端口 | 健康状态 | 用途 |
|----------|------|----------|----------|------------|----------|------|
| openclaw-gateway | openclaw-gateway:docker | - | 18789 | - | healthy | OpenClaw网关服务 |
| openclaw-cli | openclaw-gateway:docker | - | - | - | healthy | OpenClaw命令行工具 |
| auth-server | auth-auth | Python 3.11.15 | 5000 | - | - | 认证服务（使用SMTP） |
| gateway-nginx | nginx:stable | 1.28.1 | 80, 443 | 80, 443 | - | 反向代理网关 |
| mattermost | mattermost/mattermost-team-edition:11.2.1 | 11.2.1 | 8065, 8067, 8074-8075 | - | healthy | 团队协作平台 |
| mm-postgres | postgres:14 | 14.20 | 5432 | - | healthy | Mattermost数据库 |
| grafana | grafana/grafana:latest | 12.4.0 | 3000 | 3000 | - | 监控仪表板 |
| cadvisor | m.daocloud.io/gcr.io/cadvisor/cadvisor:v0.51.0 | - | 8080 | 8081 | healthy | 容器监控 |
| promtail | grafana/promtail:latest | - | - | - | - | 日志收集 |
| loki | grafana/loki:latest | 3.6.7 | 3100 | 3100 | - | 日志聚合 |
| prometheus | prom/prometheus:latest | 3.10.0 | 9090 | 9090 | - | 指标收集 |
| node-exporter | prom/node-exporter:latest | 1.10.2 | 9100 | 9100 | - | 节点指标 |
| blackbox-exporter | quay.io/prometheus/blackbox-exporter:latest | - | 9115 | 9115 | - | 黑盒监控 |
| alertmanager | prom/alertmanager | - | 9093 | 9093 | - | 告警管理 |
| ops-runner | ops-runner:latest | - | - | - | - | 运维任务执行器 |

### 4.2 访问链路描述
```
外部用户访问流程：
1. 用户访问 https://ai.example.com (OpenClaw) 或 https://chat.example.com (Mattermost)
2. DNS解析到服务器公网IP
3. 请求到达宿主机80/443端口 → gateway-nginx容器
4. Nginx进行SSL终止和认证请求转发至auth-server:5000
5. 认证通过后转发到后端服务：
   - OpenClaw: http://openclaw-gateway:18789
   - Mattermost: http://mattermost:8065
6. 后端服务处理请求并返回响应

内部服务通信：
- 所有服务通过Docker网络 shared-net (172.24.0.0/16) 通信
- 服务间使用容器名称进行DNS解析
```

### 4.3 依赖关系
1. **OpenClaw依赖**：auth-server、gateway-nginx
2. **Mattermost依赖**：mm-postgres、auth-server、gateway-nginx
3. **监控栈依赖**：Grafana依赖Prometheus/Loki，Prometheus依赖各exporter

## 五、关键配置摘要

### 5.1 OpenClaw配置 (openclaw.json)
```json
{
  "meta": { "lastTouchedVersion": "2026.3.3" },
  "models": {
    "providers": {
      "anthropic": {
        "baseUrl": "https://api.deepseek.com/v1",
        "apiKey": "[REDACTED]",
        "models": [{ "id": "deepseek-chat", "contextWindow": 128000 }]
      }
    }
  },
  "gateway": { "port": 18789, "token": "[REDACTED]" }
}
```

### 5.2 Nginx反向代理配置
**OpenClaw代理配置 (openclaw.conf)**
```nginx
server {
    listen 443 ssl;
    server_name ai.example.com;
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    location / {
        auth_request /auth_verify;
        error_page 401 = @redirect_login;
        proxy_pass http://openclaw-gateway:18789;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
    }
    location /auth_verify {
        internal;
        proxy_pass http://auth-server:5000/auth;
    }
}
```

**Mattermost代理配置 (mattermost.conf)**
```nginx
server {
    listen 443 ssl;
    server_name chat.example.com;
    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    location / {
        auth_request /auth_verify;
        error_page 401 = @redirect_login;
        proxy_pass http://mattermost:8065;
        proxy_http_version 1.1;
    }
    location /api/v4/websocket {
        proxy_pass http://mattermost:8065/api/v4/websocket;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 5.3 认证服务配置 (auth-server)
- **SMTP 主机**：smtp.qq.com
- **SMTP 端口**：587
- **SMTP 用户**：3022441811@qq.com
- **SMTP 密码**：[REDACTED]
- **发件邮箱**：3022441811@qq.com
- **Cookie 域名**：.metratio.com
- **JWT 密钥**：[REDACTED]

### 5.4 监控告警规则 (alert.rules.yml) 摘要
```yaml
groups:
- name: system
  rules:
  - alert: HighCPU (CPU >80% 5m)
  - alert: HighMemoryUsage (内存 >85% 5m)
  - alert: HighDiskUsage (磁盘 >85% 5m)
  - alert: ContainerDown (容器停止)
  - alert: HostDown (节点离线)
  - alert: ServiceDown (服务不可访问)
```

## 六、常用运维命令

### 6.1 服务启停管理
```bash
cd /srv/stack
docker compose up -d                     # 启动所有
docker compose up -d mm-postgres auth-server  # 按依赖顺序
docker compose down                       # 停止所有
```

### 6.2 日志查看
```bash
docker compose logs -f openclaw-gateway
docker compose logs -f mattermost
docker exec openclaw-gateway tail -f /home/node/.openclaw/logs/gateway.log
```

### 6.3 OpenClaw设备管理
```bash
openclaw devices list
openclaw devices approve <device-id> --roles operator
```

### 6.4 资源查看与清理
```bash
docker stats --no-stream
docker system prune -af
docker builder prune -af
df -h
```

### 6.5 健康检查
```bash
curl -f http://localhost:18789/health
curl -f http://localhost:8065/api/v4/system/ping
docker exec mm-postgres pg_isready -U postgres
curl -f http://localhost:9090/-/healthy
curl -f http://localhost:3100/ready
```

## 七、性能基准

### 7.1 典型资源使用（单次采样）
```bash
$ docker stats --no-stream
NAME                CPU %     MEM USAGE / LIMIT     MEM %
openclaw-gateway    0.00%     343.1MiB / 3.573GiB   9.38%
mm-postgres         4.12%     73.27MiB / 3.573GiB   2.00%
mattermost          0.05%     176.8MiB / 3.573GiB   4.83%
grafana             0.27%     287.5MiB / 3.573GiB   7.86%
prometheus          0.95%     228.2MiB / 3.573GiB   6.24%
loki                0.83%     194.3MiB / 3.573GiB   5.31%
...
```

### 7.2 系统负载
```bash
$ uptime
14:07:11 up 4 days, 20:42, load average: 0.00, 0.03, 0.00
```

## 八、数据备份与恢复

### 8.1 关键数据目录
| 容器 | 宿主机目录 | 内容 | 备份频率 |
|------|------------|------|----------|
| mm-postgres | /srv/data/mattermost/db | PostgreSQL数据 | 每日（cron） |
| mattermost | /srv/data/mattermost/{plugins,files,config} | 应用数据 | 每周 |
| openclaw-gateway | /srv/data/openclaw/config | OpenClaw配置和记忆 | 每日 |
| gateway-nginx | /srv/stack/nginx/ssl | SSL证书 | 每月/更新时 |

### 8.2 备份脚本示例
```bash
#!/bin/bash
BACKUP_DIR="/backup/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR
docker exec mm-postgres pg_dump -U postgres mattermost > $BACKUP_DIR/mattermost.sql
tar -czf $BACKUP_DIR/openclaw-config.tar.gz -C /srv/data/openclaw config
cp -r /srv/stack/nginx/ssl $BACKUP_DIR/
tar -czf $BACKUP_DIR/mattermost-files.tar.gz -C /srv/data/mattermost plugins files config
echo "备份完成: $BACKUP_DIR"
```

### 8.3 恢复流程
1. 停止相关服务：`docker compose stop mattermost openclaw-gateway`
2. 恢复数据库：`cat /backup/20240309/mattermost.sql | docker exec -i mm-postgres psql -U postgres mattermost`
3. 恢复配置文件：`tar -xzf /backup/20240309/openclaw-config.tar.gz -C /srv/data/openclaw/`
4. 重启服务：`docker compose up -d`

## 九、日志管理

### 9.1 Docker 日志驱动
所有容器使用 `json-file` 驱动，设大小限制。全局配置：
```bash
# /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

### 9.2 手动清理日志
```bash
truncate -s 0 /var/lib/docker/containers/*/*-json.log
```

## 十、定时任务

### 10.1 系统定时任务
- `/etc/cron.d/0hourly`：每小时运行
- `/etc/cron.d/raid-check`：每周日1点
- `/etc/cron.d/sgagenttask`、`/etc/cron.d/yunjing`：腾讯云组件

### 10.2 用户定时任务（crontab -l）
```
0 3 * * * certbot renew --quiet
10 3 * * * /usr/local/bin/ops backup
0 4 * * 0 /usr/local/bin/ops clean
```
- `certbot renew`：自动续期SSL证书（需确保Nginx停止？见12.3）
- `ops backup`：执行备份脚本（路径自定）
- `ops clean`：清理旧数据（如日志、Docker缓存）

## 十一、安全配置

### 11.1 防火墙 (firewalld)
```bash
public (active)
  ports: 22/tcp 443/tcp 80/tcp
```

### 11.2 iptables 关键规则
- Docker网络转发规则
- 腾讯云IP黑名单

### 11.3 SELinux/AppArmor
- SELinux：禁用
- AppArmor：未安装

## 十二、注意事项

### 12.1 已知问题
1. **容器日志无限制增长**：配置日志轮转。
2. **SSL证书续期**：手动复制证书到 `/srv/stack/nginx/ssl` 并重启Nginx（见12.3）。
3. **auth-server 依赖 SMTP**：邮箱密码定期更换。

### 12.2 安全提醒
- 定期轮换 OpenClaw API 密钥。
- 限制公网对监控端口的访问（3000、9090等）。
- 定期审查云厂商定时任务。

### 12.3 证书管理
```bash
# 检查证书过期时间
openssl x509 -in /srv/stack/nginx/ssl/fullchain.pem -noout -dates

# 续期流程
docker compose stop gateway-nginx
certbot renew --standalone
cp /etc/letsencrypt/live/example.com/*.pem /srv/stack/nginx/ssl/
docker compose start gateway-nginx
```

## 十三、应急快速参考

| 故障现象 | 快速排查命令 | 恢复操作 |
|----------|--------------|----------|
| Mattermost 无法访问 | `docker ps \| grep mattermost`<br>`docker logs --tail=50 mattermost` | `docker compose restart mattermost` |
| OpenClaw 无响应 | `curl localhost:18789/health`<br>`docker compose logs -f openclaw-gateway` | `docker compose restart openclaw-gateway` |
| 磁盘空间告急 | `df -h`<br>`du -sh /var/lib/docker/containers/*` | `truncate -s 0 /var/lib/docker/containers/*/*-json.log`<br>`docker system prune -af` |
| SSL证书过期 | `openssl x509 -in /srv/stack/nginx/ssl/fullchain.pem -noout -dates` | 按12.3续期 |
| 数据库连接失败 | `docker exec mm-postgres pg_isready -U postgres` | `docker compose restart mm-postgres` |

## 十四、附录

### 14.1 端口列表
| 端口 | 服务 | 访问控制 |
|------|------|----------|
| 22 | SSH | 公网 |
| 80 | HTTP→HTTPS | 公网 |
| 443 | HTTPS | 公网 |
| 3000 | Grafana | 内网 |
| 9090 | Prometheus | 内网 |
| 3100 | Loki | 内网 |
| 8081 | cAdvisor | 内网 |
| 9100 | node-exporter | 内网 |
| 9115 | blackbox-exporter | 内网 |
| 9093 | Alertmanager | 内网 |

### 14.2 环境变量列表（脱敏）
| 容器 | 变量名 | 用途 |
|------|--------|------|
| auth-server | SMTP_HOST=smtp.qq.com | 邮件服务 |
| | SMTP_USER=3022441811@qq.com | |
| | SMTP_PASS=[REDACTED] | |
| | COOKIE_DOMAIN=.metratio.com | 认证域 |
| | JWT_SECRET=[REDACTED] | 密钥 |
| openclaw-gateway | NODE_ENV=production | 运行环境 |
| | OPENCLAW_GATEWAY_TOKEN=[REDACTED] | 网关令牌 |

---
**文档信息**
- **生成时间**：2026-03-09
- **数据来源**：系统实时采集 + 手工校验
- **维护人**：ratio
- **版本**：v1.0


