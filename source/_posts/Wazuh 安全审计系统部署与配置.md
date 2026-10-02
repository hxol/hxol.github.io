---
title: Wazuh 安全审计系统部署与配置
date: 2026-09-30 15:30:00
tags: [笔记, Wazuh, 安全, 自托管]
---


# Wazuh 安全审计系统部署与配置

## 1. 系统要求

### 1.1 单节点堆栈部署 (Single-node)
- **系统**: Linux 或 Windows
- **架构**: AMD64
- **CPU**: 至少 4 个内核
- **内存**: Docker 主机至少 8 GB RAM
- **磁盘空间**: 至少 50 GB（用于 Docker 映像和数据卷）

### 1.2 多节点堆栈部署 (Multi-node)
- **系统**: Linux 或 Windows
- **架构**: AMD64
- **CPU**: 至少 4 个内核
- **内存**: Docker 主机至少 16 GB RAM
- **磁盘空间**: 至少 100 GB（用于 Docker 映像和数据卷）

### 1.3 Wazuh 代理部署 (Agent)
- **系统**: Linux 或 Windows
- **架构**: AMD64
- **CPU**: 至少 2 个内核
- **内存**: Docker 主机至少 1 GB RAM
- **磁盘空间**: 至少 10 GB（用于 Docker 映像和日志）

---

## 2. 部署准备与克隆仓库

**💡 配置小技巧**：
若想修改 Wazuh 的核心配置文件 `/var/ossec/etc/ossec.conf`，可以直接在宿主机修改 `wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf`，它会自动映射到容器内。也可以在 Wazuh Dashboard 页面右上角点击 **Server management -> Settings -> Edit configuration** 进行在线编辑。

**克隆官方 Docker 仓库**：
```bash
sudo mkdir -p /opt/wazuh 
VERSION=v4.14.7 
git clone --depth 1 https://github.com/wazuh/wazuh-docker.git -b $VERSION /tmp/wazuh-docker 
sudo cp -R /tmp/wazuh-docker/single-node /opt/wazuh/$VERSION 
cd /opt/wazuh/$VERSION
```

---

## 3. 可选附加功能

### 3.1 开启 `wazuh-archives-*` 索引及 ISM (索引生命周期管理)
archives 索引会记录**所有**日志。为了防止硬盘空间被耗尽，开启此功能时**必须**同时配置 ISM。

#### 第一步：生成纯文本及 JSON 格式归档日志
编辑 Manager 配置文件：
```bash
sudo nano /opt/wazuh/v4.14.7/config/wazuh_cluster/wazuh_manager.conf
```
找到 `<ossec_config>` -> `<global>` 下的以下两项，将 `no` 改为 `yes`：
```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```

#### 第二步：让 Filebeat 收集并发送归档日志
修改 Filebeat 挂载方式以便于编辑：
```bash
sudo nano /opt/wazuh/v4.14.7/docker-compose.yml
```
将 `- filebeat_etc:/etc/filebeat` 改为本地映射 `- ./filebeat:/etc/filebeat`，并将 `volumes:` 下的 `filebeat_etc` 及其下级缩进注释掉。

编辑本地 `filebeat.yml`：
```bash
sudo nano /opt/wazuh/v4.14.7/filebeat/filebeat.yml
```
在 `filebeat.modules:` 下找到 `archives:`，将 `enabled: false` 改为 `enabled: true`。

重启容器生效：
```bash
cd /opt/wazuh/v4.14.7/ && sudo docker compose restart
```

*(注：如果不使用目录映射，也可以通过 `docker cp` 将文件复制出来修改后再覆盖回去重启)*

#### 第三步：修改 Indexer 模板为开启 ISM 做准备
下载官方模板：
```bash
cd /tmp && rm -f wazuh-template.json
wget https://raw.githubusercontent.com/wazuh/wazuh/v4.14.7/extensions/elasticsearch/7.x/wazuh-template.json
```
修改模板：
```bash
nano /tmp/wazuh-template.json
```
> ⚠️ **注意**：JSON 文件不支持 `//` 注释，请在实际修改时**不要**带入下方的中文注释说明！

在 `"settings"` 块中做如下两处修改：
1. 将分片数改为单节点推荐的 `1`。
2. 在 `settings` 结尾处新增 ISM 策略绑定。
```json
"settings": {
  "index.refresh_interval": "5s",
  "index.number_of_shards": "1",           // <-- 修改为 1,对于单节点 Indexer,通常建议设为 "1"
  "index.number_of_replicas": "0",
  ...
  "index.query.default_field": [
    ...
    "syscheck.changed_attributes",
    "title"
  ],                                     // <--- default_field 数组结束,后面加上逗号
  "index.plugins.index_state_management.policy_id": "wazuh-alerts-90-days-retention"   // <--- 新增的 ISM 策略设置
},                                   // <--- settings 对象结束
"mappings": { ...
```
随后，前往 Dashboard 界面的 Dev Tools (开发工具) 执行 `PUT _template/wazuh` 并将整个 JSON 粘贴进去导入。

#### 第四步：在面板中创建 ISM 策略
1. 打开 Wazuh Dashboard，点击左上角导航菜单 -> **Index Management** (索引管理) -> **State Management Policies**。
2. 点击 **Create policy**，选择 **Visual editor**。
3. **Policy ID**: `wazuh-alerts-90-days-retention`
4. 添加初始状态 **hot** (Actions 留空，暂不设转换)。
5. 添加最终状态 **delete** (Action 添加 Delete 动作)。
6. 回到 **hot** 状态编辑，添加 Transition：目标选择 **delete**，条件选 **Minimum Index Age**，值为 `90 Days`。
7. 确保 `hot` 为 Initial state，点击 Create 保存。

#### 第五步：创建 Dashboard 索引模式
1. 导航到 **Dashboards Management** -> **Index patterns**。
2. 点击 **+ Create index pattern**。
3. 输入 `wazuh-archives-*`，若系统提示匹配成功，点击 Next。
4. Time field 选择 `@timestamp`，完成创建。
5. 去 **Discover** 页面选择该索引测试搜索（如 `location:"<代理IP>"`）。

---

### 3.2 添加 ntfy 作为通知服务 (Webhook)

编辑配置文件：
```bash
sudo nano /opt/wazuh/v4.14.7/config/wazuh_cluster/wazuh_manager.conf
```
在 `<ossec_config>` 块内添加。根据您的 ntfy 配置选择无认证或带认证版本：

**无认证方式：**
```xml
<integration>
  <name>webhook</name>
  <hook_url>https://ntfy.example.com/wazuh</hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

**带基础认证方式（Token 或 账号密码）：**
```xml
<integration>
  <name>webhook</name>
  <hook_url>https://<USERNAME>:<PASSWORD>@ntfy.example.com/<TOPIC></hook_url>
  <level>7</level>
  <alert_format>json</alert_format>
</integration>
```

### 3.3 接收外部 Syslog (如 OpenWrt)
如果需要让 Wazuh Manager 接收路由器等设备的 Syslog 日志，在同一文件中添加 `<remote>` 模块：
```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips><OPENWRT_IP_ADDRESS></allowed-ips> 
</remote>
```
*(同时建议在 `<global>` 下的 `<white_list>` 中将该 IP 加白)。*

---

## 4. 启动与初始化服务

### 4.1 调整端口与系统参数
为了把 443 端口留给反代服务器（如 Nginx/Traefik），修改 `docker-compose.yml` 中的 Dashboard 端口映射，将 `443:5601` 变更为 `1076:5601`。

修改 Linux 系统环境限制（需 root 权限）：
```bash
sudo sysctl -w vm.max_map_count=262144
```
*(提示：为使重启后不失效，建议将 `vm.max_map_count=262144` 写入 `/etc/sysctl.conf` 并执行 `sysctl -p`)*

### 4.2 生成证书并启动
```bash
cd /opt/wazuh/v4.14.7/ 
sudo docker compose -f generate-indexer-certs.yml run --rm generator
sudo docker compose up -d
```

---

## 5. 更改默认账户密码 (极其重要)

Wazuh 提供两类核心用户，建议全部更改密码（密码需 8~64 位，含大小写字母、数字及特殊符号）。

### 5.1 修改 Wazuh Indexer 用户 (`admin` / `kibanaserver`)
1. 退出当前的 Dashboard 登录会话。
2. 停止当前容器栈：`docker compose down`。
3. 生成新密码的 Hash 值：
   ```bash
   docker run --rm -ti wazuh/wazuh-indexer:4.14.7 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh
   ```
   复制终端输出的 Hash 字符串。
4. 在 `docker-compose.yml` 中修改明文密码（若含 `$` 需转义为 `$$`）。
5. 编辑 `config/wazuh_indexer/internal_users.yml`，将对应用户的 `hash:` 替换为刚刚生成的 Hash 值。
6. 启动容器：`docker compose up -d`。
7. 进入 Indexer 容器应用新的安全配置：
   ```bash
   docker exec -it single-node-wazuh.indexer-1 bash
   # 在容器内执行以下环境配置和刷新脚本
   export INSTALLATION_DIR=/usr/share/wazuh-indexer
   export CONFIG_DIR=$INSTALLATION_DIR/config
   CACERT=$CONFIG_DIR/certs/root-ca.pem
   KEY=$CONFIG_DIR/certs/admin-key.pem
   CERT=$CONFIG_DIR/certs/admin.pem
   export JAVA_HOME=/usr/share/wazuh-indexer/jdk
   
   bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd $CONFIG_DIR/opensearch-security/ -nhnv -cacert $CACERT -cert $CERT -key $KEY -p 9200 -icl
   ```

### 5.2 修改 Wazuh API 用户 (`wazuh-wui`)
1. 修改 `config/wazuh_dashboard/wazuh.yml` 中的 `password:` 为新密码。
2. 在 `docker-compose.yml` 中替换所有 API 旧密码（`wazuh.manager` 和 `wazuh.dashboard` 的 `API_PASSWORD`）。
3. 重启容器栈：`docker compose down && docker compose up -d`。

---

## 6. 日常运维操作

### 6.1 删除示例告警数据
导航到左侧菜单 -> **Indexer management** -> **Index Management** -> **Indexes**。搜索包含 `sample` 的索引（如 `wazuh-alerts-4.x-sample-auditing`），勾选后点击 **Actions** -> **Delete**，确认删除即可。

### 6.2 在 Wazuh 中彻底删除 Agent
> **⚠️ 最佳实践**：删除 Manager 端的 Agent 记录前，建议先在客户端机器上卸载 Wazuh Agent 软件，否则客户端会自动尝试重新注册。

**方法一：通过命令行工具 (适用于 Docker 部署)**
1. 获取 Manager 容器名称：`docker ps`。
2. 进入容器内部：
   ```bash
   docker exec -it single-node-wazuh.manager-1 bash
   ```
3. 运行管理工具：
   ```bash
   /var/ossec/bin/manage_agents
   ```
4. 输入 `R` (Remove)，再输入需删除的 Agent ID，输入 `y` 确认。

**方法二：通过 API 接口调用**
在 Dashboard 的 API Console 或通过 curl 调用：
```http
DELETE /agents?agents_list={agent_id}&purge=true
```

---

## 7. 客户端安装与监控接入

### 7.1 Linux/Windows 主机部署 Wazuh Agent

**方式一：Docker 容器安装**
```bash
sudo mkdir -p /opt/wazuh-agent 
VERSION=v4.14.7 
git clone --depth 1 https://github.com/wazuh/wazuh-docker.git -b $VERSION /tmp/wazuh-docker 
sudo cp -R /tmp/wazuh-docker/wazuh-agent /opt/wazuh-agent/$VERSION 
cd /opt/wazuh-agent/$VERSION
```
修改 `docker-compose.yml`，将 `WAZUH_MANAGER_SERVER` 的值替换为您的 Manager IP 地址 `<WAZUH_MANAGER_IP>`，然后执行 `docker compose up -d` 启动。

**方式二：二进制/APT 直接安装**
1. 导入 GPG 密钥并添加官方仓库（也可替换为您的内部镜像源，如 Nexus 私服）：
   ```bash
   curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
   chmod 644 /usr/share/keyrings/wazuh.gpg
   echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee -a /etc/apt/sources.list.d/wazuh.list
   ```
2. 执行安装（替换其中的 IP 地址）：
   ```bash
   sudo apt update && sudo WAZUH_MANAGER="<WAZUH_MANAGER_IP>" apt install wazuh-agent
   sudo systemctl enable --now wazuh-agent
   ```
*(注：后期若想更改 Manager IP，修改客户端的 `/var/ossec/etc/ossec.conf`，重启服务即可)*

### 7.2 OpenWrt 路由器的安全审计接入
由于 OpenWrt 无法直接安装 Agent，需采用 `syslog-ng` 将日志推送到 Wazuh Manager。

1. 停用默认日志系统：
   ```bash
   /etc/init.d/log disable && /etc/init.d/log stop
   ```
2. 安装 `syslog-ng`。
3. 编辑配置：`cp /etc/syslog-ng.conf /etc/syslog-ng.conf.bak && nano /etc/syslog-ng.conf`，在末尾添加 Wazuh 推送目的地：
   ```conf
   # 目标定义
   destination d_wazuh {
           network(
               "<WAZUH_MANAGER_IP>"  # 替换为您的 Wazuh Manager IP
               transport("udp")
               port(514)
           );
   };
   # 路由规则
   log {
           source(src);
           source(kernel);
           destination(d_wazuh);
   };
   ```
4. 校验并重启：
   ```bash
   syslog-ng -s -f /etc/syslog-ng.conf
   /etc/init.d/syslog-ng restart
   ```

---

## 8. 防火墙规则参考 (UFW 示例)

为确保代理能正常与 Manager 通信，需在各节点打通对应端口。核心端口为 **TCP 1514**。

**Agent 端放行出站连接（假设 Manager IP 为 `10.0.0.100`）：**
```bash
sudo ufw allow out to 10.0.0.100 port 1514 proto tcp comment 'Allow Wazuh Agent to Manager (Registration/Events)'
```
若需要 Agent 响应 Manager 触发的远程升级，还需开放 **TCP 1515**：
```bash
sudo ufw allow out to 10.0.0.100 port 1515 proto tcp comment 'Allow Wazuh Agent to Manager (Remote Upgrade)'
```
