---
layout: post
date: 2024-05-27
title: "커스텀 메트릭으로 전부 시각화하기! (telegraf, azure monitor exporter, prometheus)"
tags: [Monitoring, telegraf, Azure Monitor Exporter, Custom Metric, prometheus, ]
categories: [Azure, ]
---


모니터링을 위한 Cloud Native Service에서 지원하는 메트릭이 점차 늘어나고 있지만, 고객이 요구하는 메트릭 중 지원하지 않는 경우가 꼭, 항상, 하나쯤은 있더라.. 이럴 땐 속 시원하게 커스텀 메트릭을 만들어버리는게 멘탈에 좋다 🫠


따라서 이번 포스팅에선 표현하고자 하는 모든 데이터를 시각화하는 걸 목표로 한다.


진행 순서는 telegraf 기반으로 커스텀 메트릭을 만들어 Azure Monitor에 저장하고, 저장된 데이터를 Exporter를 사용해  Prometheus에 내보내 시각화하도록 한다.


# 구성 요소


## [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/)

- 에이전트 기반으로 서버에서 모든 종류의 데이터를 수집하고 저장할 수 있는 Open Source SW(이하 OSS)
- 데이터를 수집하고 저장하는 다양한 플러그인 지원

## [Azure Monitor Exporter](https://github.com/RobustPerception/azure_metrics_exporter)

- Azure Monitor에 적재된 메트릭을 Prometheus 서버로 내보내기 위한 Exporter
- 공식 지원은 아니지만 Prometheus Docs에서 가이드되는 Github Repo

## Prometheus

- CNCF에 속한 모니터링용 OSS
- 시계열 데이터를 수집 및 저장하고 시각화, 쿼리, 경고 등 모니터링을 위한 다양한 기능을 제공

# 설치


## 0. 진행 환경

- VM 2대
	- 모니터링 대상 서버 (이하 타겟 서버)
		- OS : CentOS 7.9
		- 관리 ID 활성
	- Prometheus 설치 대상 서버

## 1. telegraf 구성


### 1.1 패키지 설치

- [링크](https://www.influxdata.com/time-series-platform/telegraf/)를 참고하여 타겟 서버의 OS에 맞춰 설치 진행

{% raw %}
```bash
# Redhat/CentOS 설치
cat <<EOF | sudo tee /etc/yum.repos.d/influxdata.repo
[influxdata]
name = InfluxData Repository - Stable
baseurl = https://repos.influxdata.com/stable/\$basearch/main
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdata-archive_compat.key
EOF

yum install -y telegraf influxdb
```
{% endraw %}


### 1.2 플러그인 설정

- 데이터 수집을 위한 input 플러그인과 데이터를 저장하기 위한 output 플러그인 설정
	- input 플러그인 : cpu, mem, exec
	- output 플러그인 : azure_monitor, _influxdb (optional)_
- [전체 플러그인 참고](https://docs.influxdata.com/telegraf/v1/plugins/)

{% raw %}
```bash
cat <<'EOF' >> /etc/telegraf/telegraf.conf
# custom settings
## output plugins
[[outputs.azure_monitor]]

[[outputs.influxdb]]
  database = "telegraf"

## input plugins
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false

[[inputs.mem]]

[[inputs.exec]]
  commands = ["sh -c 'echo \"{\\\"DiskUsageTmp\\\": $(du -s /tmp | cut -f1)}\"'"]
  data_format = "json"
  
[[inputs.exec]]
  commands = ["sh -c 'echo \"{\\\"DiskUsageOpt\\\": $(du -s /opt | cut -f1)}\"'"]
  data_format = "json"
EOF

systemctl start telegraf
systemctl status telegraf
systemctl enable telegraf
```
{% endraw %}


### 1.3 수집 데이터 확인

- Azure Monitor에서 커스텀 메트릭 확인
	- Azure Monitor 또는 타겟 서버 블레이드 > 메트릭 > 네임스페이스 ‘telegraf/exec’ 선택 > 커스텀 메트릭 선택

		![0](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/0.png)

	- 대시보드 확인 (그래프 변화를 위해 더미파일 생성)

		![1](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/1.png)

- influx에서 로컬 저장 데이터 확인

{% raw %}
```bash
# influxd 데몬 실행
systemctl start influxd
systemctl status influxd

# influx 접속 및 쿼리
influx
> use telegraf
_Using database telegraf_
> select * from exec order by time desc limit 5
name: exec
time                DiskUsageOpt DiskUsageTmp host
----                ------------ ------------ ----
1716225830000000000 1048576      1048580      
1716225820000000000 1048576      1048580      
1716225810000000000 1048576      1048580      
1716225800000000000 1048576      1048580      
1716225790000000000 1048576      1048580      
> quit

# 쿼리 데이터 검증
du -s /opt /tmp
1048576	/opt
1048580	/tmp
```
{% endraw %}


![2](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/2.png)


## 2. Prometheus 서버 구성


### 2.1 사용자 및 디렉토리 추가


{% raw %}
```bash
useradd -m -s /bin/false prometheus
id prometheus
uid=1001(prometheus) gid=1001(prometheus) groups=1001(prometheus)

mkdir /etc/prometheus /var/lib/prometheus
chown prometheus /var/lib/prometheus
```
{% endraw %}


### 2.2 패키지 설치

- 다운로드 링크 : [https://prometheus.io/download/](https://prometheus.io/download/)
- 본 테스트에선 LTS 버전 사용 (작성일 기준 v2.45.5)

{% raw %}
```bash
cd /opt
wget https://github.com/prometheus/prometheus/releases/download/v2.45.5/prometheus-2.45.5.linux-amd64.tar.gz
tar -zxvf ./prometheus-2.45.5.linux-amd64.tar.gz
cd prometheus-2.45.5.linux-amd64/

cp -rp prometheus /usr/local/bin
cp -rp promtool /usr/local/bin
```
{% endraw %}


### 2.3 config 파일 생성


{% raw %}
```bash
cat <<EOF > /etc/prometheus/prometheus.yml
# Global config
global:
  scrape_interval:     60s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 60s # Evaluate rules every 15 seconds. The default is every 1 minute.
  scrape_timeout: 15s  # scrape_timeout is set to the global default (10s).

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
  - job_name: azure
    static_configs:
      - targets: ['localhost:9276']
EOF
```
{% endraw %}

- global
	- scrape_interval → 데이터 수집 주기
	- evaluation_interval → 규칙 평가 주기
	- scrape_timeout → 데이터 스크랩 시 최대 허용 시간
- scrape_configs : 데이터 수집 대상 정의
	- “prometheus” job → prometheus 서버
	- “azure” job → Azure Monitor에서 메트릭을 읽어오는 Exporter

### 2.4 데몬 구성 및 실행


{% raw %}
```bash
cat <<EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Time Series Collection and Processing Server
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start prometheus && systemctl status prometheus
systemctl enable prometheus

netstat -antup | grep 9090
```
{% endraw %}


### 2.5 대시보드 접속 확인

- 브라우저 실행 후 \<promethes_ip\>:9090 접속

	![3](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/3.png)


## 3. Azure Monitor Exporter 구성


### 3.1 빌드를 위한 Go 설치

- 다운로드 링크 : https://go.dev/doc/install

{% raw %}
```bash
wget https://go.dev/dl/go1.22.3.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.3.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

go version
# go version go1.22.3 linux/amd64
```
{% endraw %}


### 3.2 Azure Monitor Exporter 소스 코드 Clone


{% raw %}
```bash
git clone https://github.com/RobustPerception/azure_metrics_exporter
cd azure_metrics_exporter/
go build

ll | grep azure_metrics_exporter
-rwxr-xr-x. 1 root root 13016773 May 14 05:58 azure_metrics_exporter
```
{% endraw %}


### 3.3 구성 파일(azure.yml) 업데이트

- 기본 구성파일 = \<빌드 디렉토리\>/azure.yml
- Azure Service Principal 생성 후 credentials의 각 value에 기입

{% raw %}
```bash
cat <<EOF > azure.yml
active_directory_authority_url: "https://login.microsoftonline.com/"
resource_manager_url: "https://management.azure.com/"
credentials:
  subscription_id: <subscription_id>
  client_id: <SP_id>
  client_secret: <SP_secret>
  tenant_id: <tenant_id>

targets:
resource_groups:
  - resource_group: "<resource_group_id>"  # optional
    resource_types:
    - 'Microsoft.Compute/virtualMachines'
    resource_name_include_re:
    - "<vm_name>.*"  # optional
    metric_namespace: "Telegraf/exec"
    metrics:
    - name: "DiskUsageOpt"
    - name: "DiskUsageTmp"
EOF
```
{% endraw %}


### 3.4 에이전트 실행


{% raw %}
```bash
./azure_metrics_exporter
2024/05/23 03:57:37 azure_metrics_exporter listening on port :9276
```
{% endraw %}


## 4. 로그 저장 흐름 확인


### 4.1  타겟 서버/telegraf - 로그 수집 및 전달

- telegraf의 로컬 DB인 influx 내 수집 데이터 저장 및 Azure Monitor로 전달

	![4](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/4.png)


### 4.2 Azure Monitor - 커스텀 메트릭 저장


![5](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/5.png)


![6](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/6.png)


### 4.3 모니터링 서버/Azure Monitor Exporter - 커스텀 메트릭 가져오기

- Exporter는 별도로 데이터를 저장하지 않고 Prometheus 서버의 GET 요청 당시의 데이터만 반환
- 커맨드 수행 : `curl <exporter_server_ip>:9276/metrics`

	![7](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/7.png)


### 4.4 모니터링 서버/Prometheus

- 구성파일에서 선언해둔 job의 엔드포인트에 주기적으로 GET 요청
- 메트릭 데이터를 수집하고 자체 데이터베이스(TSDB)에 저장
	- GET 요청 시 응답값

		![8](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/8.png)

	- TSDB 저장 데이터
		- 기본 디렉토리 경로 = /var/lib/prometheus
		- 아래 각각의 디렉토리 내 압축된 형태로 저장되며 PromQL 기반으로 쿼리가 가능

			![9](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/9.png)

	- 웹 UI 쿼리 화면

		![10](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/10.png)


		![11](/assets/img/2024-05-27-커스텀-메트릭으로-전부-시각화하기!-(telegraf,-azure-monitor-exporter,-prometheus).md/11.png)

