# Prometheus监控



## 一、 总览

主要组件：

- Prometheus server： 用于收集和存储时间序列数据
- exporter： 客户端生成监控指标
- Alertmanager： 处理警报
- Grafana： 数据可视化和输出
- Pushgateway：主动推送数据给Prometheus server



架构图：

![普罗米修斯建筑](https://prometheus.io/assets/architecture.png)



## 二 、环境搭建

**基础环境**

| 名字            | 版本                          |
| --------------- | ----------------------------- |
| OS              | CentOS Linux release 7.8.2003 |
| docker          | 19.03.11                      |
| docker-composer | 1.26.0                        |
| IP              | 192.168.0.80                  |



### 2.1 编辑配置文件

#### 2.1.1 编辑prometheus配置文件

/etc/prometheus/prometheus.yml

```yaml
# 全局配置
global:
  scrape_interval:     15s # 多久收集一次数据,默认是1分钟.
  evaluation_interval: 15s # 每隔15秒评估一次规则. 默认是每分钟.
  # scrape_timeout is set to the global default (10s).

# 告警配置
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['192.168.0.80:9093']

# 加载一次规则，并根据全局“评估间隔”定期评估它们。
rule_files:
  - "/etc/prometheus/rules.yml"
  
# 控制Prometheus监视哪些资源
# 默认配置中，有一个名为prometheus的作业，它会收集Prometheus服务器公开的时间序列数据。
scrape_configs:
  # 作业名称将作为标签“job=<job_name>`添加到此配置中获取的任何数据。
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'machine'
    static_configs:
    - targets: ['192.168.0.81:9100']
      labels:
          group: 'db'
    - targets: ['localhost:9100']
      labels:
          group: 'db'
```

#### 2.1.2 编辑规则文件

/etc/prometheus/rules.yml

```yaml
groups:
- name: example
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      serverity: page
    annotations:
      summary: "Instance {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."

```

#### 2.1.3 编辑告警配置文件

```yaml

global:
  resolve_timeout: 5m
  smtp_smarthost: 'xxx@xxx:587'
  smtp_from: 'zhaoysz@xxx'
  smtp_auth_username: 'xxx@xxx'
  smtp_auth_password: 'xxxx'
  smtp_require_tls: true

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'test-mails'
receivers:
- name: 'test-mails'
  email_configs:
  - to: 'scottcho@qq.com'
```



### 2.2 docker-composer

#### 2.2.1 编辑docker-composer

/docker-compose/prometheus/docker-compose.yml

```yml
version: "3.2"

volumes:
  prometheus_data: {}
  grafana_data: {}
  alertmanager_data: {}


services:

  prometheus:
    image: prom/prometheus
    volumes:
    - /etc/prometheus/:/etc/prometheus/
    - prometheus_data:/prometheus
    command:
    - '--config.file=/etc/prometheus/prometheus.yml'
    - '--storage.tsdb.path=/prometheus'
    - '--web.console.libraries=/usr/share/prometheus/console_libraries'
    - '--web.console.templates=/usr/share/prometheus/consoles'
    - '--web.external-url=http://192.168.0.12:9090/'
    - '--web.enable-lifecycle'
    - '--storage.tsdb.retention=15d'
    ports:
    - 9090:9090
    links:
    - alertmanager:alertmanager
    restart: always

  alertmanager:
    image: prom/alertmanager
    ports:
    - 9093:9093
    volumes:
    - /etc/alertmanager/:/etc/alertmanager/
    - alertmanager_data:/alertmanager
    command:
    - '--config.file=/etc/alertmanager/alertmanager.yml'
    - '--storage.path=/alertmanager'
    restart: always


  grafana:
    image: grafana/grafana
    ports:
    - 3000:3000
    volumes:
    - /etc/grafana/:/etc/grafana/provisioning/
    - grafana_data:/var/lib/grafana
    environment:
    - GF_INSTALL_PLUGINS=camptocamp-prometheus-alertmanager-datasource
    links:
    - prometheus:prometheus
    - alertmanager:alertmanager
    restart: always
```



#### 2.2.2  启动composer

```bash
docker-compose up
```



#### 2.2.3 访问端点

http://localhost:9090    				Prometheus server主页

http://localhost:9090/metrics      Prometheus server自身指标

http://192.168.0.80:3000            Grafana



### 2.3 安装Node_Export

node_export用于采集客户端硬件信息

yum 安装方法:   https://copr.fedorainfracloud.org/coprs/ibotty/prometheus-exporters/

```bash
curl -Lo /etc/yum.repos.d/_copr_ibotty-prometheus-exporters.repo https://copr.fedorainfracloud.org/coprs/ibotty/prometheus-exporters/repo/epel-7/ibotty-prometheus-exporters-epel-7.repo
yum -y install node_exporter
systemctl start node_exporter
systemctl enable node_exporter.service
```



二进制文件

官网下载地址(https://prometheus.io/download/)

```bash
tar -zxvf node_exporter-1.0.0-rc.1.linux-amd64.tar.gz
 ./node_exporter --web.listen-address=:9100 
```



访问地址： http://localhost:9100/metrics   



### 2.4 配置Grafana

访问http://192.168.0.80:3000/，初始登录账号/密码： admin/admin



#### 2.4.1创建Prometheus数据源

1. 单击侧栏中的“齿轮”以打开“配置”菜单。
2. 单击“数据源”。
3. 点击“添加数据源”。
4. 选择“ Prometheus”作为类型。
5. 设置适当的Prometheus服务器网址（例如，`http://192.168.0.80:9090/`）
6. 根据需要调整其他数据源设置（例如，选择正确的访问方法）。
7. 单击“保存并测试”以保存新的数据源。

![image-20200701145501769](C:\Users\Scott\AppData\Roaming\Typora\typora-user-images\image-20200701145501769.png)



#### 2.4.2 导入Node-Export仪表板

官方模板查询地址: https://grafana.com/grafana/dashboards

找到模板编号8919

![image-20200701150431232](C:\Users\Scott\AppData\Roaming\Typora\typora-user-images\image-20200701150431232.png)

将模板导入到Grafana

左侧菜单找到+图标，点击导入，输入编号，点击Load，然后选择数据源，出现下图

![image-20200701150759350](C:\Users\Scott\AppData\Roaming\Typora\typora-user-images\image-20200701150759350.png)

最后点击Import，出现图表

