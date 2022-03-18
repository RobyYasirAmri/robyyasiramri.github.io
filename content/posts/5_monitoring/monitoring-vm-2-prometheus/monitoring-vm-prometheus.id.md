---
title: "Cara Mudah Monitoring VM Ubuntu Menggunakan Node Exporter, Prometheus dan Grafana - Bagian II"
date: 2022-02-18T08:06:25+06:00
description: Cara Mudah Monitoring VM Ubuntu Menggunakan Node Exporter, Prometheus dan Grafana
menu:
  sidebar:
    name: Bagian II - Monitoring VM on Prometheus and Grafana
    identifier: monitoring-prometheus-2-en
    parent: monitoring-category
    weight: 2
tags: ["Kubernetes", "Linux"]
categories: ["Monitoring", "Linux","Node-Exporter","Prometheus","Grafana"]
hero: /posts/monitoring/logo-prometheus.png
---

Hai Gaes! 
Pada jurnal sebelumnya kita sudah bahas mengenai Cara Mudah Monitoring VM Ubuntu Menggunakan Node Exporter, Prometheus dan Grafana bagian satu yaitu install node exporter nih, sekarang kita lanjut buat install Prometheus dan Grafananya skuy! 
Environtment dan resources masih sama seperti yang sebelumnya yah, kali ini kita lanjutin labnya.

{{< img src="/posts/kubernetes/topologi-ha-kubernetes.png" height="700" width="600" align="center" title="Topologi" >}}

Oke, karena kita masih menggunakan topologi yang sama kita bisa langsung lanjut aja ke cara Install Prometheus disalah satu VM/Instances. Ingat kita install Prometheus di VM rb-k8s-lb1 ya

## Install dan konfigurasi

1. Langkah pertama kita download file Prometheusnya dulu dengan command:
```
wget https://github.com/prometheus/prometheus/releases/download/v2.33.0/prometheus-2.33.0.linux-amd64.tar.gz
```
2. Kemudian kita ekstrak hasil download tadi:
```
tar xvfz prometheus-2.33.0.linux-amd64.tar.gz
```
3. Kita pindahkan hasil ekstrak ke directory /opt
```
mv prometheus-2.33.0.linux-amd64 /opt
```
4. Kemudian ubah isi file prometheus.yaml menjadi seperti ini:
```
sudo cp /opt/prometheus-2.33.0.linux-amd64/prometheus.yml /opt/prometheus-2.33.0.linux-amd64/prometheus.yml.bak
sudo nano /opt/prometheus-2.33.0.linux-amd64/prometheus.yml
```
```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=job_name` to any timeseries scraped from this config.
  - job_name: 'loadbalancer'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: 
      - '10.60.60.43:9100'
      - '10.60.60.44:9100'

  - job_name: 'master'
    static_configs:
    - targets:
      - '10.60.60.51:9100'
      - '10.60.60.52:9100'
      - '10.60.60.53:9100'

  - job_name: 'worker'
    static_configs:
    - targets:
      - '10.60.60.54:9100'
      - '10.60.60.55:9100'
      - '10.60.60.56:9100'
```
5. Buat file dengan nama prometheus_server.service agar ketika server di reboot/restart service tetap berjalan
```
nano /etc/systemd/system/prometheus_server.service
```
```
[Unit]
Description=Prometheus Server

[Service]
User=root
ExecStart=/opt/prometheus-2.33.0.linux-amd64/prometheus --config.file=/opt/prometheus-2.33.0.linux-amd64/prometheus.yml --web.external-url=http://10.60.60.43:9090/

[Install]
WantedBy=default.target
```
6. Kemudian Enable, start, Service tersebut dengan command:
```
systemctl enable prometheus_server.service
systemctl start prometheus_server.service\
```
## Verifikasi

7. Pastikan service Prometheus berjalan
```
systemctl status prometheus_server.service
```
```
root@rb-k8s-lb1:/opt/prometheus-2.33.0.linux-amd64# systemctl status prometheus_server.service
● prometheus_server.service - Prometheus Server
     Loaded: loaded (/etc/systemd/system/prometheus_server.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2022-03-11 18:59:53 UTC; 2s ago
   Main PID: 1408527 (prometheus)
      Tasks: 10 (limit: 4676)
     Memory: 70.7M
     CGroup: /system.slice/prometheus_server.service
             └─1408527 /opt/prometheus-2.33.0.linux-amd64/prometheus --config.file=/opt/prometheus-2.33.0.linux-amd64/prometheus.yml --web.external-url=http://10.60.60.43:9090/

Mar 11 18:59:54 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:54.952Z caller=head.go:604 level=info component=tsdb msg="WAL segment loaded" segment=371 maxSegment=374
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.148Z caller=head.go:604 level=info component=tsdb msg="WAL segment loaded" segment=372 maxSegment=374
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.340Z caller=head.go:604 level=info component=tsdb msg="WAL segment loaded" segment=373 maxSegment=374
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.341Z caller=head.go:604 level=info component=tsdb msg="WAL segment loaded" segment=374 maxSegment=374
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.341Z caller=head.go:610 level=info component=tsdb msg="WAL replay completed" checkpoint_replay_duration=1.203064377s wal_replay_duration=534.227604ms total_replay_duration=1.776991108s
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.346Z caller=main.go:944 level=info fs_type=EXT4_SUPER_MAGIC
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.346Z caller=main.go:947 level=info msg="TSDB started"
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.346Z caller=main.go:1128 level=info msg="Loading configuration file" filename=/opt/prometheus-2.33.0.linux-amd64/prometheus.yml
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.357Z caller=main.go:1165 level=info msg="Completed loading of configuration file" filename=/opt/prometheus-2.33.0.linux-amd64/prometheus.yml totalDuration=10.632798ms db_storage=1.542µs remote_stora>
Mar 11 18:59:55 rb-k8s-lb1 prometheus[1408527]: ts=2022-03-11T18:59:55.357Z caller=main.go:896 level=info msg="Server is ready to receive web requests."
root@rb-k8s-lb1:/opt/prometheus-2.33.0.linux-amd64#
```
Untuk testing bisa gunakan browser atau dengan command curl melalui port 9090
```
root@rb-k8s-lb1:/opt/prometheus-2.33.0.linux-amd64# curl 10.60.60.43:9090
```
{{< img src="/posts/monitoring/prometheus-1.png" height="700" width="600" align="center" title="Topologi" >}}

{{< img src="/posts/monitoring/prometheus-2.png" height="700" width="600" align="center" title="Topologi" >}}

Ok gaes, untuk kali ini sampai sini dulu, nanti kita lanjut untuk install Grafananya yah, terimakasih