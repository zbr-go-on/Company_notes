# 安装Telegraf

```bash
wget https://dl.influxdata.com/telegraf/releases/telegraf-1.28.3-1.x86_64.rpm
yum install -y telegraf-1.28.3-1.x86_64.rpm
```

# 编辑配置文件（/etc/telegraf/telegraf.conf）

```shell

# 将采集的数据输出到influxdb数据库中 
[[outputs.influxdb]]
  ## 配置API请求
  urls = ["http://127.0.0.1:9086"]
  database = "dolphin_monitor"
  username = "dolphin"
  password = "dolphin@123"


# 系统基础运维指标采集
[[inputs.cpu]]
[[inputs.mem]]
[[inputs.disk]]
[[inputs.nstat]]

# 配置dolphinscheduler的采集 --按需配置
[[inputs.http]]
  urls = ["http://192.168.1.201:12345/dolphinscheduler/monitor/masters"]
  method = "GET"
  headers = {
    "token" = "b8cce0d21b68842c942d2afc64011d88",  # 添加认证Token
    "Content-Type" = "application/json"
  }
  timeout = "10s"
  
  ## 数据解析配置
  data_format = "json"
  name_override = "dolphin_masters_info"  # 自定义指标名称
  json_query = "data"  # 提取data数组中的内容
## 可选：添加标签（例如集群名称）
  [inputs.http.tags]
    cluster = "master"
## 可选：降低采集频率（默认10秒）
  interval = "1m"
# Telegraf配置
[[inputs.exec]]
  command = "/bin/bash /zhongbingrui/telegraf_shell/parse_dolphin_masters.sh"
  interval = "1m"
  data_format = "influx"
  name_override = "dolphin_masters_info"

```

利用dolphin提供的API，处理JSON数据为telegraf可支持的格式数据

```bash
#!/bin/bash
curl -s "http://192.168.1.201:12345/dolphinscheduler/monitor/masters" \
  -H "token: b8cce0d21b68842c942d2afc64011d88" | \
jq -r '.data[] | 
  "dolphin_masters_info,host=\(.host),port=\(.port) " +
  "startup_time=\(.resInfo | fromjson.startupTime)," +
  "cpu_usage=\(.resInfo | fromjson.cpuUsage)," +
  "memory_usage=\(.resInfo | fromjson.memoryUsage)," +
  "disk_available=\(.resInfo | fromjson.diskAvailable)"'
```

**考虑到后续dolphinscheduler会被阿里云的DataWorks取代，所以减少了对dolphinscheduler-API的研究**

```bash
# 1. 语法检查（必须无报错）
telegraf --test --config /etc/telegraf/telegraf.conf

# 2. 查看配置结构（确保插件独立）
grep -n '^\[\[inputs\.' /etc/telegraf/telegraf.conf
# 应输出类似：
# 4000:[[inputs.http]]
# 4020:[[inputs.cpu]]

# 3. 检查JSON字段映射（自动展开后的字段）
curl -s "http://192.168.1.201:12345/dolphinscheduler/monitor/masters" \
  -H "token: b8cce0d21b68842c942d2afc64011d88" | jq '.data[0].resInfo_startupTime'
```

# 启动服务

```bash
systemctl enable telegraf
systemctl start telegraf
```

# 验证数据

```bash
influx -database 'dolphin_monitor' -execute 'SELECT * FROM dolphin_task_stats LIMIT 10'
```