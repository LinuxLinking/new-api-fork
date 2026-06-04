# New API 云端部署完全指南

## 概述
本指南将帮助您在服务器上部署 New API，实现：
- ✅ 24小时在线服务
- ✅ Web 端管理界面
- ✅ 云端统一管理
- ✅ 随时随地访问

---

## 目录
- [服务器准备](#服务器准备)
- [快速部署（Docker Compose）](#快速部署docker-compose)
- [配置说明](#配置说明)
- [Web 管理面板使用](#web-管理面板使用)
- [安全建议](#安全建议)
- [常见问题](#常见问题)

---

## 服务器准备

### 1. 服务器要求
| 组件 | 最低配置 | 推荐配置 |
|-----|---------|---------|
| CPU | 1 核 | 2 核或以上 |
| 内存 | 1 GB | 2 GB 或以上 |
| 硬盘 | 10 GB | 20 GB 或以上 |
| 系统 | Linux (Ubuntu/CentOS/Debian) | Ubuntu 20.04/22.04 |

### 2. 安装 Docker 和 Docker Compose

#### Ubuntu/Debian 系统：
```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 启动 Docker
sudo systemctl start docker
sudo systemctl enable docker

# 安装 Docker Compose (Docker v2 已包含)
docker --version
docker compose version
```

#### CentOS/RHEL 系统：
```bash
# 安装 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 启动 Docker
sudo systemctl start docker
sudo systemctl enable docker
```

### 3. 配置防火墙（可选但推荐）
```bash
# Ubuntu (UFW)
sudo ufw allow 3000/tcp
sudo ufw allow ssh
sudo ufw enable

# CentOS (firewalld)
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

---

## 快速部署（Docker Compose）

### 方法一：使用完整 docker-compose.yml（推荐）

这个方案包含 PostgreSQL 数据库和 Redis 缓存，适合生产环境：

```bash
# 1. 创建目录
mkdir -p /opt/new-api && cd /opt/new-api

# 2. 下载或创建 docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.4'

services:
  new-api:
    image: calciumion/new-api:latest
    container_name: new-api
    restart: always
    command: --log-dir /app/logs
    ports:
      - "3000:3000"
    volumes:
      - ./data:/data
      - ./logs:/app/logs
    environment:
      # ⚠️ 重要：生产环境请修改这些密码！
      - SQL_DSN=postgresql://newapi_user:your_secure_password@postgres:5432/new-api
      - REDIS_CONN_STRING=redis://:your_redis_password@redis:6379
      - TZ=Asia/Shanghai
      - ERROR_LOG_ENABLED=true
      - BATCH_UPDATE_ENABLED=true
      - NODE_NAME=new-api-server-1
      # ⚠️ 多机部署必须设置 SESSION_SECRET
      # - SESSION_SECRET=change_this_to_a_random_string_very_long
    depends_on:
      - redis
      - postgres
    networks:
      - new-api-network

  redis:
    image: redis:latest
    container_name: new-api-redis
    restart: always
    # ⚠️ 请修改 Redis 密码
    command: ["redis-server", "--requirepass", "your_redis_password"]
    networks:
      - new-api-network

  postgres:
    image: postgres:15
    container_name: new-api-postgres
    restart: always
    environment:
      # ⚠️ 请修改数据库密码
      POSTGRES_USER: newapi_user
      POSTGRES_PASSWORD: your_secure_password
      POSTGRES_DB: new-api
    volumes:
      - pg_data:/var/lib/postgresql/data
    networks:
      - new-api-network

volumes:
  pg_data:

networks:
  new-api-network:
    driver: bridge
EOF

# 3. 启动服务
docker compose up -d

# 4. 查看状态
docker compose ps

# 5. 查看日志
docker compose logs -f new-api
```

### 方法二：简化版（仅 SQLite）

如果您想更简单，可以使用 SQLite 数据库：

```bash
# 1. 创建目录
mkdir -p /opt/new-api && cd /opt/new-api

# 2. 创建简化版 docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.4'

services:
  new-api:
    image: calciumion/new-api:latest
    container_name: new-api
    restart: always
    ports:
      - "3000:3000"
    volumes:
      - ./data:/data
    environment:
      - TZ=Asia/Shanghai
EOF

# 3. 启动
docker compose up -d
```

---

## 配置说明

### 环境变量详解

参考文件 [`.env.example`](file:///workspace/.env.example) 和 [`docker-compose.yml`](file:///workspace/docker-compose.yml)：

| 变量 | 说明 | 默认值 |
|-----|------|--------|
| `TZ` | 时区 | `Asia/Shanghai` |
| `SQL_DSN` | 数据库连接字符串（MySQL/PostgreSQL） | SQLite |
| `REDIS_CONN_STRING` | Redis 连接字符串 | - |
| `SESSION_SECRET` | 会话密钥（多机部署必须） | - |
| `ERROR_LOG_ENABLED` | 启用错误日志 | `false` |
| `STREAMING_TIMEOUT` | 流式响应超时（秒） | `120` |

### 配置反向代理（可选但推荐）

使用 Nginx 配置域名和 HTTPS：

```nginx
server {
    listen 80;
    server_name your-domain.com;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

然后使用 Certbot 申请免费 SSL 证书：
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

---

## Web 管理面板使用

### 1. 首次登录

部署完成后，在浏览器访问：
```
http://your-server-ip:3000
```

默认账号（如果未设置）：
- 用户名：`root`
- 密码：`123456`

⚠️ **重要：首次登录后立即修改密码！**

### 2. 核心功能

#### 📊 仪表板
- 查看系统状态
- 统计数据和图表
- 实时监控

#### 🔗 通道管理
在 `通道` 菜单中：
1. 添加新通道（支持 50+ 种 AI 服务）
2. 配置支持的模型
3. 设置优先级和权重
4. 分组管理

#### 👥 用户管理
- 创建用户
- 分配额度
- 管理权限

#### 🎫 Token 管理
- 生成访问令牌
- 设置模型限制
- 配置分组

#### 💳 充值与订阅
- 配置支付方式（如需）
- 管理用户额度

---

## 安全建议

### 1. 基础安全
```bash
# ⚠️ 修改所有默认密码
# - 数据库密码
# - Redis 密码
# - New API 管理员密码
# - SESSION_SECRET（生成随机字符串）
```

生成随机 SESSION_SECRET：
```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

### 2. 网络安全
- 使用防火墙限制访问
- 配置 HTTPS
- 不要将数据库端口暴露到公网
- 考虑使用 VPN 或内网访问

### 3. 数据备份
```bash
# 备份数据目录
tar -czf new-api-backup-$(date +%Y%m%d).tar.gz /opt/new-api/data

# 自动备份脚本示例
cat > /usr/local/bin/backup-new-api.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/backups"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR
cd /opt/new-api
docker compose exec -T postgres pg_dump -U newapi_user new-api > $BACKUP_DIR/db-$DATE.sql
tar -czf $BACKUP_DIR/data-$DATE.tar.gz ./data
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
EOF

chmod +x /usr/local/bin/backup-new-api.sh

# 添加到 crontab 每天凌晨 2 点备份
(crontab -l ; echo "0 2 * * * /usr/local/bin/backup-new-api.sh") | crontab -
```

### 4. 更新维护
```bash
# 更新到最新版本
cd /opt/new-api
docker compose pull
docker compose up -d

# 查看日志
docker compose logs -f --tail=100 new-api
```

---

## 常见问题

### Q: 如何修改端口？
A: 修改 `docker-compose.yml` 中的 `ports` 配置：
```yaml
ports:
  - "8080:3000"  # 左边是宿主机端口，可自定义
```

### Q: 数据存在哪里？
A: 默认存在 `./data` 目录，包括：
- SQLite 数据库（如果使用）
- 配置文件
- 上传文件

### Q: 如何从旧的 One API 迁移？
A: New API 兼容 One API 的数据库，直接使用相同的数据库连接即可。

### Q: 忘记管理员密码怎么办？
A: 由于使用了加密，最简单的方法是重新初始化数据库（会丢失数据），或如果有备份从备份恢复。

---

## 总结

现在您已经拥有：
- ✅ 24小时运行的 New API 服务
- ✅ Web 管理界面（http://your-ip:3000）
- ✅ 云端统一管理
- ✅ 随时可以通过 API 访问

下一步：
1. 登录管理面板修改默认密码
2. 添加您的 AI 通道（OpenAI、Claude 等）
3. 创建 Token 开始使用
4. 根据需要配置域名和 HTTPS

如有问题，查看 [官方文档](https://docs.newapi.pro/en/docs)。
