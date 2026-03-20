### 三台服务器差异化配置

| 服务器 | 内网IP | NODE_NAME | PUBLISH_HOST | 是否部署Kibana |
|--------|--------|-----------|--------------|----------------|
| 物理机 | 192.168.1.106 | node1 | 192.168.1.106 | ✅ 是 |
| 虚拟机1 | 192.168.1.107 | node2 | 192.168.1.107 | ❌ 否 |
| 虚拟机2 | 192.168.1.108 | node3 | 192.168.1.108 | ❌ 否 |

#### 🚀 启动命令示例（每台服务器执行）

```bash
# 物理机 (192.168.1.106)
export NODE_NAME=node1
export PUBLISH_HOST=192.168.1.106
docker-compose up -d

# 虚拟机1 (192.168.1.107)
export NODE_NAME=node2
export PUBLISH_HOST=192.168.1.107
docker-compose up -d

# 虚拟机2 (192.168.1.108)
export NODE_NAME=node3
export PUBLISH_HOST=192.168.1.108
docker-compose up -d
```

> 💡 建议将环境变量写入 `.env` 文件，避免每次手动 export

---

## ⚠️ 关键注意事项

### 🔧 系统前置要求（三台服务器都要配置）

```bash
# 1. 修改 sysctl.conf（允许内存锁定）
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
sysctl -p

# 2. 确保 ulimit 无限制（docker-compose 中已配置，但宿主机也建议检查）
ulimit -l unlimited

# 3. 开放防火墙端口
firewall-cmd --add-port=9200/tcp --permanent  # HTTP API
firewall-cmd --add-port=9300/tcp --permanent  # 集群通信（关键！）
firewall-cmd --reload
```

### 🔍 集群发现机制说明

| 配置项 | 作用 | 注意事项 |
|--------|------|----------|
| `discovery.seed_hosts` | 告知节点去哪些地址寻找其他节点 | 填所有节点的 **publish_host** |
| `cluster.initial_master_nodes` | 首次集群引导时选举主节点 | **仅首次启动生效**，集群形成后自动忽略 |
| `network.publish_host` | 节点对外通信的地址 | 必须是其他节点能访问到的内网IP |

### 🔄 关于 `cluster.initial_master_nodes`

- **仅在三台节点都是全新空数据目录时需要**
- 如果某台节点已有数据，可能导致冲突
- 集群成功组建后，该配置可移除（但留着也无害）

---

## 🧪 验证集群状态

```bash
# 在任意节点执行
curl http://192.168.1.106:9200/_cluster/health?pretty

# 期望返回
{
  "cluster_name": "es-cluster",
  "status": "green",    # 或 yellow（单副本时正常）
  "number_of_nodes": 3, # 确认3个节点都加入
  "number_of_data_nodes": 3
}

# 查看节点列表
curl http://192.168.1.106:9200/_cat/nodes?v
```

---

## 🔄 如果已经运行了单节点怎么办？

1. **停止并清理旧容器**（⚠️ 注意备份数据！）
   ```bash
   docker-compose down
   # 如需重置数据：docker volume rm xxx_esdata
   ```

2. **使用新配置重新启动**（按上述步骤）

3. **如果保留数据**：确保 `cluster.name` 与原来不同，或先导出索引再迁移

---

## 📦 关于镜像构建

你的 Dockerfile 已经正确安装了 IK 分词器：

```dockerfile
RUN bin/elasticsearch-plugin install -b https://release.infinilabs.com/analysis-ik/stable/elasticsearch-analysis-ik-8.12.0.zip
```

✅ **无需重新构建**，三台服务器共用同一个 `elasticsearch-ik:8.12.0` 镜像即可。

> 如果后续要升级插件版本或添加其他插件，才需要重新 `docker build` 并推送到各节点。

---

## 🔐 生产环境建议（非必须，但推荐）

```yaml
# 1. 启用安全认证（移除 xpack.security.enabled=false）
# 2. 配置 TLS/SSL 加密节点间通信
# 3. 设置合理的副本数（默认1，三节点可保持）
# 4. 使用 docker swarm 或 k8s 管理更便捷
# 5. 配置监控和日志收集（Metricbeat + Filebeat）
```

如有具体报错或网络不通的问题，可以贴出日志进一步排查！🚀
