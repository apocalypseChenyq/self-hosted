# Sentry自托管服务管理指南

## 概述

本文档提供Sentry自托管版本的日常服务管理操作指南，包括启动、关闭、重启等常用操作。

## 前置条件

- ✅ Sentry已成功安装
- ✅ 位于Sentry安装目录（包含docker-compose.yml文件）
- ✅ Docker和Docker Compose正常运行

## 🚀 服务启动

### 推荐启动方式

#### 1. 后台启动（推荐日常使用）
```bash
docker-compose up -d
```
**特点**：
- ✅ 后台运行，不占用终端
- ✅ 终端关闭后服务继续运行
- ✅ 适合生产环境使用

#### 2. 启动并等待就绪
```bash
docker-compose up --wait
```
**特点**：
- ✅ 等待所有服务完全启动
- ✅ 确保服务健康检查通过
- ✅ 适合初次启动或验证

#### 3. 前台启动（调试用）
```bash
docker-compose up
```
**特点**：
- 🔍 实时查看所有服务日志
- 🔍 适合调试和故障排查
- ⚠️ 终端关闭会停止服务

### 启动验证

启动后验证服务状态：
```bash
# 查看所有服务状态
docker-compose ps

# 检查Web服务健康状态
curl -f http://localhost:9000/api/0/internal/health/ || echo "服务未就绪"

# 访问Web界面
open http://localhost:9000  # macOS
# 或在浏览器中访问 http://localhost:9000
```

## 🔴 服务关闭

### 推荐关闭方式

#### 1. 优雅停止（推荐）
```bash
docker-compose down
```
**特点**：
- ✅ 优雅停止所有容器
- ✅ 删除容器但保留数据卷
- ✅ 保留所有用户数据和配置
- ✅ 下次启动快速恢复

#### 2. 仅停止服务（保留容器）
```bash
docker-compose stop
```
**特点**：
- ✅ 停止服务但不删除容器
- ✅ 可快速重启：`docker-compose start`
- ✅ 适合临时维护

#### 3. 完全清理（危险操作）
```bash
docker-compose down -v
```
**⚠️ 警告**：
- 🚨 **会删除所有数据卷**
- 🚨 **会丢失所有Sentry数据**
- 🚨 **仅在完全重置时使用**

### 关闭验证

确认服务已停止：
```bash
# 检查是否还有运行的容器
docker-compose ps

# 检查端口是否释放
lsof -i :9000 || echo "端口9000已释放"
```

## 🔄 服务重启

### 重启所有服务
```bash
docker-compose restart
```

### 重启特定服务
```bash
# 重启Web服务
docker-compose restart web

# 重启数据库
docker-compose restart postgres

# 重启多个服务
docker-compose restart web worker
```

### 强制重建并重启
```bash
# 重建镜像并启动
docker-compose up -d --build

# 强制重新创建容器
docker-compose up -d --force-recreate
```

## 📊 服务监控

### 查看服务状态
```bash
# 查看所有服务状态
docker-compose ps

# 查看健康状态详情
docker-compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

### 查看日志
```bash
# 查看所有服务日志
docker-compose logs

# 查看特定服务日志
docker-compose logs web
docker-compose logs postgres

# 实时查看日志
docker-compose logs -f web

# 查看最近的日志
docker-compose logs --tail=100 web
```

### 资源使用监控
```bash
# 查看容器资源使用
docker stats

# 查看磁盘使用
docker system df

# 查看数据卷使用
docker volume ls
```

## 🔧 日常维护

### 更新服务
```bash
# 拉取最新镜像
docker-compose pull

# 应用更新（重启服务）
docker-compose up -d --pull always
```

### 清理资源
```bash
# 清理未使用的镜像和容器
docker system prune -f

# 清理未使用的数据卷（谨慎使用）
docker volume prune
```

### 备份数据
```bash
# 备份PostgreSQL数据
docker-compose exec postgres pg_dump -U postgres sentry > sentry_backup_$(date +%Y%m%d).sql

# 备份数据卷
docker run --rm -v sentry-self-hosted_sentry-postgres:/data -v $(pwd):/backup ubuntu tar czf /backup/postgres_backup_$(date +%Y%m%d).tar.gz -C /data .
```

## 🚨 故障排查

### 常见问题及解决方案

#### 1. 服务启动失败
```bash
# 检查端口占用
lsof -i :9000
netstat -tulpn | grep :9000

# 检查磁盘空间
df -h

# 检查Docker状态
docker info
```

#### 2. 服务无响应
```bash
# 检查服务健康状态
docker-compose ps

# 查看错误日志
docker-compose logs web --tail=50

# 重启问题服务
docker-compose restart web
```

#### 3. 数据库连接问题
```bash
# 检查数据库状态
docker-compose logs postgres

# 测试数据库连接
docker-compose exec postgres psql -U postgres -c "SELECT version();"
```

### 紧急恢复步骤
```bash
# 1. 停止所有服务
docker-compose down

# 2. 清理问题容器
docker-compose down --remove-orphans

# 3. 重新启动
docker-compose up -d --force-recreate

# 4. 检查状态
docker-compose ps
docker-compose logs
```

## 📋 快速参考

### 常用命令速查表

| 操作 | 命令 | 说明 |
|------|------|------|
| 启动服务 | `docker-compose up -d` | 后台启动所有服务 |
| 停止服务 | `docker-compose down` | 优雅停止并删除容器 |
| 重启服务 | `docker-compose restart` | 重启所有服务 |
| 查看状态 | `docker-compose ps` | 查看服务运行状态 |
| 查看日志 | `docker-compose logs web` | 查看Web服务日志 |
| 进入容器 | `docker-compose exec web bash` | 进入Web容器 |
| 更新服务 | `docker-compose pull && docker-compose up -d` | 更新并重启 |

### 服务访问地址

| 服务 | 地址 | 说明 |
|------|------|------|
| Sentry Web界面 | http://localhost:9000 | 主要管理界面 |
| 健康检查 | http://localhost:9000/api/0/internal/health/ | 服务健康状态 |

## 🔧 自动化脚本

### 创建启动脚本
```bash
# 创建启动脚本
cat > start-sentry.sh << 'EOF'
#!/bin/bash
echo "启动Sentry服务..."
cd /path/to/sentry/self-hosted
docker-compose up -d --wait
echo "Sentry服务已启动: http://localhost:9000"
EOF

chmod +x start-sentry.sh
```

### 创建停止脚本
```bash
# 创建停止脚本
cat > stop-sentry.sh << 'EOF'
#!/bin/bash
echo "停止Sentry服务..."
cd /path/to/sentry/self-hosted
docker-compose down
echo "Sentry服务已停止"
EOF

chmod +x stop-sentry.sh
```

## 📝 注意事项

### ⚠️ 重要提醒
1. **数据安全**：使用`docker-compose down -v`会删除所有数据
2. **端口冲突**：确保9000端口未被其他应用占用
3. **资源需求**：Sentry需要足够的内存和磁盘空间
4. **备份策略**：定期备份数据库和配置文件

### 🔧 性能优化
- 监控容器资源使用情况
- 定期清理不必要的日志和临时文件
- 根据使用量调整容器资源限制

---

**文档版本**: 1.0
**更新时间**: 2025-06-21
**适用版本**: Sentry Self-Hosted latest
