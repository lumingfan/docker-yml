# 1. 启动 (自动拉取镜像)
docker-compose up -d

# 2. 查看日志
docker-compose logs -f mysql

# 3. 进入容器
docker-compose exec mysql bash

# 4. 连接数据库
docker-compose exec mysql mysql -u root -p

# 5. 停止
docker-compose down

# 6. 删除数据卷 (慎用！会清空所有数据)
docker-compose down -v
