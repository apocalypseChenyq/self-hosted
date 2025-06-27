# Sentry自托管安装Docker镜像拉取问题解决方案

## 问题描述

在安装Sentry自托管版本时，遇到以下错误：

```
[symbolicator-cleanup internal] load metadata for docker.io/getsentry/symbolicator:nightly
#3 ERROR: failed to authorize: DeadlineExceeded: failed to fetch oauth token: Post "https://auth.docker.io/token": dial tcp 199.16.156.38:443: i/o timeout
------
 > [symbolicator-cleanup internal] load metadata for docker.io/getsentry/symbolicator:nightly:
------
failed to solve: DeadlineExceeded: DeadlineExceeded: DeadlineExceeded: getsentry/symbolicator:nightly: failed to resolve source metadata for docker.io/getsentry/symbolicator:nightly: failed to authorize: DeadlineExceeded: failed to fetch oauth token: Post "https://auth.docker.io/token": dial tcp 199.16.156.38:443: i/o timeout
```

## 问题分析

### 根本原因
1. **网络连接问题**：无法正常访问Docker Hub的认证服务器(`auth.docker.io`)
2. **镜像版本配置问题**：`.env`文件配置使用`nightly`标签，但需要从网络拉取
3. **本地镜像可用性**：本地已有相关镜像的`nightly`版本，但配置要求`latest`版本

### 影响范围
- Sentry自托管安装无法完成
- Docker镜像构建失败
- 主要影响symbolicator、vroom等服务

## 解决方案

### 方案一：配置Docker国内镜像源（推荐）

1. **编辑Docker daemon配置文件**：
   ```bash
   vim ~/.docker/daemon.json
   ```

2. **添加国内镜像源**：
   ```json
   {
     "builder": {
       "gc": {
         "defaultKeepStorage": "20GB",
         "enabled": true
       }
     },
     "experimental": false,
     "registry-mirrors": [
       "https://docker.m.daocloud.io",
       "https://dockerhub.azk8s.cn",
       "https://reg-mirror.qiniu.com"
     ]
   }
   ```

3. **重启Docker服务**：
   ```bash
   # macOS
   # 点击状态栏Docker图标 -> Restart Docker Desktop

   # Linux
   sudo systemctl restart docker
   ```

### 方案二：修改镜像版本配置

1. **修改.env文件**：
   ```bash
   # 将所有nightly标签改为latest
   sed -i '' 's/:nightly/:latest/g' .env
   ```

2. **验证修改结果**：
   ```bash
   cat .env | grep IMAGE
   ```

   应该显示：
   ```
   SENTRY_IMAGE=getsentry/sentry:latest
   SNUBA_IMAGE=getsentry/snuba:latest
   RELAY_IMAGE=getsentry/relay:latest
   SYMBOLICATOR_IMAGE=getsentry/symbolicator:latest
   TASKBROKER_IMAGE=getsentry/taskbroker:latest
   VROOM_IMAGE=getsentry/vroom:latest
   ```

### 方案三：使用本地镜像标记

1. **检查本地可用镜像**：
   ```bash
   docker images | grep nightly
   ```

2. **将nightly镜像标记为latest**：
   ```bash
   docker tag getsentry/sentry:nightly getsentry/sentry:latest
   docker tag getsentry/relay:nightly getsentry/relay:latest
   docker tag getsentry/snuba:nightly getsentry/snuba:latest
   docker tag getsentry/taskbroker:nightly getsentry/taskbroker:latest
   docker tag getsentry/vroom:nightly getsentry/vroom:latest
   docker tag getsentry/symbolicator:nightly getsentry/symbolicator:latest
   ```

3. **验证镜像标记**：
   ```bash
   docker images | grep latest | grep getsentry
   ```

## 执行步骤

### 完整解决流程

1. **配置Docker镜像源**（可选但推荐）
2. **修改环境变量配置**：
   ```bash
   cd /path/to/sentry/self-hosted
   sed -i '' 's/:nightly/:latest/g' .env
   ```

3. **标记本地镜像**：
   ```bash
   # 批量标记所有nightly镜像为latest
   for image in sentry relay snuba taskbroker vroom symbolicator; do
     docker tag getsentry/$image:nightly getsentry/$image:latest
   done
   ```

4. **清理Docker缓存**（可选）：
   ```bash
   docker system prune -f
   ```

5. **重新运行安装**：
   ```bash
   ./install.sh
   ```

6. **启动服务**：
   ```bash
   docker-compose up --wait
   ```

## 验证解决方案

### 检查安装是否成功
安装成功的标志：
- ✅ 所有Docker镜像构建完成
- ✅ 数据库迁移执行完毕
- ✅ 显示"You're all done!"消息
- ✅ 提示运行`docker-compose up --wait`

### 检查服务状态
```bash
# 查看所有服务状态
docker-compose ps

# 查看日志（如有问题）
docker-compose logs
```

## 后续操作

### 创建管理员用户
```bash
docker-compose exec web sentry createuser
```

### 访问Sentry界面
- 默认访问地址：http://localhost:9000
- 根据`.env`文件中的`SENTRY_BIND`配置确定端口

## 预防措施

### 长期解决方案
1. **保持Docker镜像源配置**：确保`~/.docker/daemon.json`中的镜像源配置持续有效
2. **定期更新镜像**：
   ```bash
   docker-compose pull
   docker-compose up -d
   ```

3. **备份关键配置**：
   - 备份`.env`文件
   - 备份`docker-compose.yml`（如有修改）
   - 备份数据库和数据卷

### 网络环境优化
- 考虑使用VPN或代理改善Docker Hub访问
- 配置企业内部Docker Registry（如适用）

## 常见问题

### Q: 修改镜像源后仍然无法拉取镜像？
A:
1. 确认Docker服务已重启
2. 尝试手动拉取镜像测试：`docker pull hello-world`
3. 检查网络连接和防火墙设置

### Q: 本地没有nightly版本镜像怎么办？
A:
1. 使用VPN连接后重新运行安装
2. 寻找其他可用的镜像源
3. 考虑使用离线安装包

### Q: 服务启动后无法访问Web界面？
A:
1. 检查端口配置：`docker-compose ps`
2. 确认防火墙设置
3. 查看服务日志：`docker-compose logs web`

## 相关资源

- [Sentry自托管官方文档](https://develop.sentry.dev/self-hosted/)
- [Docker镜像源配置指南](https://docs.docker.com/registry/recipes/mirror/)
- [Sentry GitHub Issues](https://github.com/getsentry/self-hosted/issues)

---

**创建时间**: 2025-06-21
**适用版本**: Sentry Self-Hosted latest
**测试环境**: macOS 24.5.0, Docker Desktop
