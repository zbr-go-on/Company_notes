# 安装

```bash
# 1. 进入临时目录（存放安装包）
cd /tmp

# 2. 下载InfluxDB（无需指定目录）
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.8.10.x86_64.rpm

# 3. 安装（自动分配路径）
yum install -y influxdb-1.8.10.x86_64.rpm


systemctl enable influxdb
systemctl start influxdb


# 4. 初始化操作（无需进入特定目录）
# 初始化数据库
influx
  CREATE DATABASE dolphin_monitor
  CREATE USER "dolphin" WITH PASSWORD 'dolphin@123'
  GRANT ALL ON dolphin_monitor TO dolphin
  exit

```

#### **关键目录详解（安装后自动生成）**

```bash
# 配置文件路径
/etc/influxdb/influxdb.conf

# 数据存储路径
/var/lib/influxdb/data

# 日志路径
/var/log/influxdb/influxd.log

# 服务脚本路径
/usr/lib/systemd/system/influxdb.service
```

总结来说，用户需要一个分步骤的指南，详细说明如何安全地更改InfluxDB的端口，更新相关服务配置，并验证整个系统的正常运行。同时，要避免影响现有的DolphinScheduler服务，确保所有操作不会导致服务中断。

---

## 解决方案：修改InfluxDB端口（绕过冲突）

**核心思路** ：保留DolphinScheduler的8086端口，将InfluxDB迁移到新端口（如`**9086**`）

### **一、修改InfluxDB端口**

```bash
vim /etc/influxdb/influxdb.conf

# 修改HTTP端口配置
[http]
  enabled = true
  bind-address = ":9086"  # 原8086改为9086

# 重启服务
systemctl restart influxdb

```

---

### **二、更新关联组件配置**

#### 1. **Telegraf（所有节点）**

```bash
# 编辑配置文件
vim /etc/telegraf/telegraf.conf

# 修改输出目标端口
[[outputs.influxdb]]
  urls = ["http://cdh201:9086"]  # 原8086改为9086
```

#### 2. **Grafana（CDH202）**

```bash
1. 登录Grafana：http://cdh202:3000
2. 进入 **Configuration -> Data Sources -> InfluxDB**
3. 修改URL为：http://cdh201:9086
4. 点击 **Save & Test**
```

### **四、验证数据流程**

```bash
# 在CDH201执行
influx -port 9086 -database 'dolphin_monitor' -execute 'SHOW MEASUREMENTS'

# 在Telegraf节点执行
telegraf --test --config /etc/telegraf/telegraf.conf
```

### **五、端口记录表（重要）**

|   |   |   |   |
|---|---|---|---|
|服务|源端口|新端口|文件路径|
|**InfluxDB HTTP**|**8086**|**9086**|`**/etc/influxdb/influxdb.conf**`|
|**Telegraf输出**|**8086**|**9086**|`**/etc/telegraf/telegraf.conf**`|
|**Grafana数据源**|**8086**|**9086**|**Grafana Web界面配置**|

### **六、风险规避**

**配置备份** ：

```bash
cp /etc/influxdb/influxdb.conf /etc/influxdb/influxdb.conf.bak
cp /etc/telegraf/telegraf.conf /etc/telegraf/telegraf.conf.bak
```

**回滚方案** ：

```bash
# 若出现问题，恢复原配置
cp /etc/influxdb/influxdb.conf.bak /etc/influxdb/influxdb.conf
systemctl restart influxdb
```

1. **使用正确的端口连接** ：如果InfluxDB确实运行在9086端口，用户应该使用`**influx -port 9086**`来连接，或者修改配置文件恢复默认端口。

## 命令行进入influx

influx -port 9086 -username 'dolphin' -password 'dolphin@123' -database 'dolphin_monitor'

#### **验证InfluxDB进程状态**

```bash
# 查看进程是否存活
ps aux | grep influxd

# 预期输出示例：
# influxdb  23824  0.1  1.2 1234567 98765 ?  Ssl  11:23   0:00 /usr/bin/influxd -config /etc/influxdb/influxdb.conf
```

  
  

---

### **一、InfluxDB Line Protocol 格式解析**

InfluxDB Line Protocol 是 Telegraf 的默认输入格式之一，其结构如下：

<measurement>[,<tag_key>=<tag_value>,...] <field_key>=<field_value>,... <timestamp>

- `**measurement**`：指标名称（如 `**dolphin_masters_info**`）。
- `**tags**`：键值对，用于标识数据（如 `**host=192.168.1.201,port=5678**`）。
- `**fields**`：实际的数值数据（如 `**cpu_usage=0.01**`）。
- `**timestamp**`：时间戳（可选，默认为当前时间，单位为纳秒）。

---