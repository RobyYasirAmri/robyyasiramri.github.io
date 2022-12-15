---
title: "Easy Ways to Monitor Ubuntu VMs Using Node Exporter, Prometheus and Grafana"
date: 2022-02-18T08:06:25+06:00
description: Easy Ways to Monitor Ubuntu VMs Using Node Exporter, Prometheus and Grafana
menu:
  sidebar:
    name: Part I - Monitoring VM on Prometheus and Grafana
    identifier: monitoring-node-exporter-id
    parent: monitoring-category
    weight: 1
tags: ["Kubernetes", "Linux"]
categories: ["Monitoring", "Linux","Node-Exporter","Prometheus","Grafana"]
hero: /posts/monitoring/logo-prometheus.png
---
Halo Gaes,
Pada jurnal kali ini kita akan coba untuk menginstall Node Exporter pada Ubuntu dan mem-visualisasikan hasil dari metric Node Exporter tersebut kedalam Dashboard Grafana. Secara garis besar Node Exporter disini berfungsi sebagai agent untuk mengubah data-data pada server menjadi metric, kemudian metrics tersebut kita lempar ke Prometheus dan di visualisasikan metric itu kedalam Dashboard Grafana agar lebih mudah dalam membacanya.
Untuk lab kali ini hal-hal yang akan kita lakukan yaitu:
1. Install Node Exporter di setiap VM/Instances
2. Install Prometheus disalah satu VM/Instances
3. Install dan configure Grafana
4. Import Dashboard Grafana

Pada lab ini kita masih menggunakan environtment dari lab yang sebelumnya, jadi untuk topologi yang digunakan masih sama. Install dan konfigurasi Prometheus dan Grafana dilakukan di VM rb-k8s-lb1

{{< img src="/posts/3_kubernetes/1_cara-konfigurasi-ha/topologi-ha-kubernetes.png" height="700" width="600" align="center" title="Topologi" >}}

## Install Node Exporter di setiap VM/Instances
Okay, kita langsung saja ke step pertama yaitu install node exporter di semua VM/instances, kenapa? karena tujuan kita yaitu untuk memonitor resources semua VM tersebut, skuyy!

1. Langkah yang pertama harus kita lakukan yaitu install Node exporternya dengan command:
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz
```

2. Setelah kita download kemudian kita ekstrak file tersebut dengan command:
```
tar xvfz node_exporter-1.3.1.linux-amd64.tar.gz
```
3. Kemudian untuk memudahkan kita dalam proses intallasi karena kita tentu tidak akan melakukan install paket node exporter tersebut secara manual satu persatu, kita bisa lakukan menggunakan script. Untuk script pertama kita siapkan daftar list IP VM mana saja yang akan kita install node exporter. kita beri file dengan nama ip-list.txt 
```
sudo nano ip-list.txt
```
```
10.60.60.43
10.60.60.44
10.60.60.51
10.60.60.52
10.60.60.53
10.60.60.54
10.60.60.55
10.60.60.56
```
4. Lalu kita bikin script untuk install node_exporter tadi, kita namakan script ini add-node-exporter.sh
```
sudo nano add-node-exporter.sh
```
5. Kemudian kita bikin service node_exporter dengan nama node_exporter.service 
```
sudo nano node_exporter.service
```
```
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
6. Sampai sini kita harusnya sudah mempunyai 3 file, yaitu add-node-exporter.sh node_exporter.service dan add-node-exporter.sh. Kemudian kita bikin script untuk mentransfer file semua script tadi ke masing-masing VM. Kita beri nama file ini dengan nama transfer-node-exporter.sh 
```
sudo nano transfer-node-exporter.sh
```
```
#!/bin/bash

for i in $(cat ip-list.txt) 
do 
	echo "#### Execute On $i ####"
	scp node_exporter node_exporter.service add-node-exporter.sh $i:~/
done
```
7. Setelah file siap. kita tinggal jalankan script transfer-node-exporter.sh dengan cara: 
```
sudo chmod +x transfer-node-exporter.sh
sudo ./transfer-node-exporter.sh
```
8. Jika semua file telah di transfer ke masing-masing VM, kita siapkan satu script lagi untuk install node exporternya. kita namakan file ini dengan install-node-exporter.sh 
```
sudo nano install-node-exporter.sh
```
```
#!/bin/bash

for i in $(cat ip-list.txt) 
do 
	echo "#### Installing On $i ####"
	ssh $i bash add-node-exporter.sh systemctl status node_exporter.service
done
```
9. Untuk mulai install node exporter di semua VM, kita cukup menggunakan file yang terakhir saja karena file sebelumnya kita sudah copy ke masing-masing VM: 
```
sudo chmod +x install-node-exporter.sh
./install-node-exporter.sh
```
Apabila kita ingin menginstal node exporter di kemudian hari dengan jumlah VM yang lumayan banyak, kita tinggal rubah saja untuk list IPnya pada file IP-list.txt, kemudian kita transfer semua file menggunakan script transfer-node-exporter.sh lalu kita bisa jalankan script install-node-exporter.sh untuk langsung mulai install di semua VM. 

## Verifikasi
Untuk memverifikasi hasil install node-exporter dapat kita lakukan dengan menggunakan command: 
```
root@rb-k8s-lb1:~# sudo systemctl status node_exporter.service 
● node_exporter.service - Node Exporter
     Loaded: loaded (/etc/systemd/system/node_exporter.service; disabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-02-02 16:24:00 UTC; 1 months 6 days ago
   Main PID: 2006 (node_exporter)
      Tasks: 10 (limit: 4676)
     Memory: 22.4M
     CGroup: /system.slice/node_exporter.service
             └─2006 /usr/local/bin/node_exporter

Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.253Z caller=node_exporter.go:115 level=info collector=thermal_zone
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.253Z caller=node_exporter.go:115 level=info collector=time
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.253Z caller=node_exporter.go:115 level=info collector=timex
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.253Z caller=node_exporter.go:115 level=info collector=udp_queues
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.253Z caller=node_exporter.go:115 level=info collector=uname
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.253Z caller=node_exporter.go:115 level=info collector=vmstat
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.253Z caller=node_exporter.go:115 level=info collector=xfs
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.253Z caller=node_exporter.go:115 level=info collector=zfs
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.254Z caller=node_exporter.go:199 level=info msg="Listening on" address=:9100
Feb 02 16:24:00 rb-k8s-lb1 node_exporter[2006]: ts=2022-02-02T16:24:00.254Z caller=tls_config.go:195 level=info msg="TLS is disabled." http2=false
root@rb-k8s-lb1:~# 
```
atau bisa menggunakan perintah curl melalui terminal
```
root@rb-k8s-lb1:~# curl localhost:9100 -k 
```