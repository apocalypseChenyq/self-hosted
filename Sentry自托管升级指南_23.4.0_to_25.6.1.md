# Sentry自托管升级指南：从23.4.0到25.6.1

## 概述

本指南将帮助您将Sentry自托管版本从23.4.0升级到25.6.1。由于版本跨度较大，需要通过多个Hard Stop版本进行逐步升级。

## ⚠️ 重要提醒

1. **数据备份**：升级前必须备份所有数据
2. **停机时间**：升级过程中服务将不可用
3. **Hard Stop**：必须按照指定的Hard Stop版本逐步升级
4. **测试环境**：建议先在测试环境进行升级验证

## 升级路径

根据官方文档，从23.4.0到25.6.1需要经过以下Hard Stop版本：

```
23.4.0 → 23.6.2 → 23.11.0 → 24.8.0 → 25.5.1 → 25.6.1
```

## 预备工作

### 1. 系统要求检查

确保系统满足最新版本要求：
- **内存**：至少4GB（推荐8GB+）
- **磁盘空间**：至少20GB可用空间
- **Docker**：版本20.10.0+
- **Docker Compose**：版本2.23.2+

### 2. 数据备份（必须）

#### 完整备份（推荐）
```bash
# 停止所有服务
docker-compose down

# 备份所有数据卷
docker run --rm -v $(pwd):/backup -v sentry-postgres:/data ubuntu tar czf /backup/postgres-backup-$(date +%Y%m%d).tar.gz -C /data .
docker run --rm -v $(pwd):/backup -v sentry-clickhouse:/data ubuntu tar czf /backup/clickhouse-backup-$(date +%Y%m%d).tar.gz -C /data .
docker run --rm -v $(pwd):/backup -v sentry-redis:/data ubuntu tar czf /backup/redis-backup-$(date +%Y%m%d).tar.gz -C /data .

# 备份配置文件
cp -r sentry/ sentry-backup-$(date +%Y%m%d)/
cp .env .env-backup-$(date +%Y%m%d)
```

#### 使用Sentry内置备份（可选）
```bash
# 创建数据备份
./scripts/backup.sh

# 备份文件将保存在 sentry/backup.json
```

### 3. 环境检查

```bash
# 检查当前版本
docker-compose exec web sentry --version

# 检查Docker版本
docker --version
docker-compose --version

# 检查磁盘空间
df -h
```

## 升级步骤

### 第一步：升级到23.6.2

#### 1. 获取代码
```bash
cd /path/to/sentry/self-hosted
git fetch --all
git checkout 23.6.2
```

#### 2. 检查配置文件
```bash
# 比较配置文件差异
diff -u sentry/sentry.conf.example.py sentry/sentry.conf.py
diff -u .env.example .env

# 手动更新必要的配置
```

#### 3. 执行升级
```bash
# 运行安装脚本
./install.sh

# 启动服务并等待就绪
docker-compose up --wait
```

#### 4. 验证升级
```bash
# 检查服务状态
docker-compose ps

# 检查版本
docker-compose exec web sentry --version

# 访问Web界面验证
curl -f http://localhost:9000/api/0/internal/health/
```

### 第二步：升级到23.11.0

#### 1. 切换版本
```bash
git checkout 23.11.0
```

#### 2. 重要配置更新

在23.11.0版本中有以下重要变更：
- 移除了`sentry run smtp`工作进程
- 引入了新的指标后端

更新`.env`文件：
```bash
# 检查新的环境变量
diff -u .env.example .env

# 合并必要的新配置到你的.env文件
```

#### 3. 执行升级
```bash
./install.sh
docker-compose up --wait
```

#### 4. 验证升级
```bash
docker-compose exec web sentry --version
```

### 第三步：升级到24.8.0

#### 1. 切换版本
```bash
git checkout 24.8.0
```

#### 2. 重要配置更新

24.8.0版本引入了多项重要更新：
- 移除了所有Zookeeper相关配置
- 引入了新的错误监控模式

#### 3. 清理旧资源
```bash
# 如果之前版本 < 21.12.0，执行清理
COMPOSE_PROJECT_NAME=sentry_onpremise docker-compose down --rmi local --remove-orphans
```

#### 4. 执行升级
```bash
./install.sh
docker-compose up --wait
```

### 第四步：升级到25.5.1（Hard Stop）

#### 1. 切换版本
```bash
git checkout 25.5.1
```

#### 2. 重要说明
25.5.1是一个重要的Hard Stop版本，包含了数据库迁移的重要变更。

#### 3. 执行升级
```bash
./install.sh
docker-compose up --wait
```

### 第五步：升级到25.6.1（最终版本）

#### 1. 切换版本
```bash
git checkout 25.6.1
```

#### 2. 新特性配置

25.6.1版本包含的新特性：
- ARM64支持改进
- SMTP容器更新
- Taskbroker服务（预览）
- 连续性能分析支持

#### 3. 更新配置文件

检查并更新关键配置：
```bash
# 检查sentry.conf.py的变更
diff -u sentry/sentry.conf.example.py sentry/sentry.conf.py

# 如果使用自定义.env.custom文件
cat .env.example >> .env.custom  # 添加新的环境变量
```

重要的新配置项：
```python
# sentry.conf.py中的新特性启用
SENTRY_FEATURES.update({
    'organizations:continuous-profiling': True,
    'organizations:session-replay-canvas': True,
    'organizations:trace-view': True,
})
```

#### 4. 执行最终升级
```bash
./install.sh
docker-compose up --wait
```

## 升级后验证

### 1. 系统健康检查
```bash
# 检查所有服务状态
docker-compose ps

# 健康检查
curl -f http://localhost:9000/api/0/internal/health/

# 检查日志
docker-compose logs --tail=100
```

### 2. 功能验证
- 登录Web界面
- 检查项目数据完整性
- 验证错误报告功能
- 测试新功能（如果适用）

### 3. 性能监控
```bash
# 检查资源使用
docker stats

# 检查磁盘使用
docker system df
```

## 版本间重大变更

### 23.6.2 → 23.11.0
- **移除SMTP工作进程**：不再支持`sentry run smtp`
- **指标后端启用**：新的指标和通用指标后端
- **会话数据迁移**：发布健康会话写入新的指标后端

### 23.11.0 → 24.8.0
- **Zookeeper移除**：完全移除Zookeeper依赖
- **Kafka升级**：迁移到无Zookeeper的Kafka
- **错误监控增强**：引入"errors-only"模式

### 24.8.0 → 25.5.1
- **数据库迁移压缩**：重要的数据库schema变更
- **配置清理**：移除过时的功能标志

### 25.5.1 → 25.6.1
- **ARM64支持**：Linux ARM64完全支持
- **SMTP容器更新**：更强大的SMTP服务
- **Taskbroker引入**：新的任务处理服务（预览）
- **连续性能分析**：启用持续性能监控

## 故障排查

### 常见问题及解决方案

#### 1. 数据库连接错误
```bash
# 检查PostgreSQL状态
docker-compose logs postgres

# 重启数据库服务
docker-compose restart postgres
```

#### 2. 迁移失败
```bash
# 查看迁移日志
docker-compose logs web | grep migration

# 手动运行迁移
docker-compose exec web sentry upgrade --noinput
```

#### 3. 磁盘空间不足
```bash
# 清理Docker镜像
docker system prune -a -f

# 清理日志
docker-compose logs --tail=0 -f > /dev/null
```

#### 4. 服务启动失败
```bash
# 查看详细错误日志
docker-compose logs --tail=50 [service_name]

# 重新构建镜像
docker-compose down
docker-compose up --build
```

## 回滚方案

如果升级失败，可以通过以下方式回滚：

### 1. 使用数据备份回滚
```bash
# 停止服务
docker-compose down

# 删除当前数据卷
docker volume rm sentry-postgres sentry-clickhouse sentry-redis

# 恢复备份
docker run --rm -v $(pwd):/backup -v sentry-postgres:/data ubuntu tar xzf /backup/postgres-backup-YYYYMMDD.tar.gz -C /data
docker run --rm -v $(pwd):/backup -v sentry-clickhouse:/data ubuntu tar xzf /backup/clickhouse-backup-YYYYMMDD.tar.gz -C /data
docker run --rm -v $(pwd):/backup -v sentry-redis:/data ubuntu tar xzf /backup/redis-backup-YYYYMMDD.tar.gz -C /data

# 切换到原版本
git checkout 23.4.0

# 恢复配置
cp -r sentry-backup-YYYYMMDD/* sentry/
cp .env-backup-YYYYMMDD .env

# 启动服务
docker-compose up --wait
```

### 2. 使用Sentry内置恢复
```bash
# 恢复数据
./scripts/restore.sh

# 重新配置
docker-compose exec web sentry repair --owner=admin@example.com
```

## 升级完成后的优化

### 1. 性能调优
```bash
# 检查内存使用
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# 调整worker数量（如果需要）
# 在docker-compose.yml中调整worker副本数
```

### 2. 监控配置
- 配置健康检查告警
- 设置磁盘使用监控
- 配置错误率阈值告警

### 3. 新功能启用
根据需要启用25.6.1的新功能：
- 连续性能分析
- Session Replay Canvas
- Trace View

## 总结

升级从23.4.0到25.6.1是一个重要的版本升级，包含了许多架构改进和新功能。务必：

1. **严格按照Hard Stop路径升级**
2. **在每个步骤前备份数据**
3. **验证每个版本的升级结果**
4. **准备回滚方案**
5. **在生产环境升级前进行测试**

升级完成后，您将获得：
- 更好的性能和稳定性
- 新的监控和分析功能
- 改进的用户界面
- 更强的扩展性支持

---

**创建时间**: 2025-01-21
**适用版本**: Sentry Self-Hosted 23.4.0 → 25.6.1
**文档版本**: 1.0
**预计升级时间**: 2-4小时（取决于数据量）
