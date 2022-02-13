---
title: "Cara Konfigurasi High Availabilty Kubernetes Cluster di Ubuntu 20.4"
date: 2022-02-13T08:06:25+06:00
description: Introduction to Sample Post
menu:
  sidebar:
    name: Kubernetes
    identifier: Kubernetes
    weight: 10
tags: ["Kubernetes", "Linux"]
categories: ["Kubernetes", "Linux"]
---
Pada kesempatan kali ini saya akan membagikan bagaimana cara konfigurasi High Availability Cluster Kubernetes di Ubuntu menggunakan HAProxy dan Keepalived. Kita menggunakan Keepalived dan HAproxy untuk load balancing dan high availabilty-nya. Hal yang perlu di persiapkan antara lain:
1. Kebutuhan hosts
2. konfigurasi HAproxy dan Keepalived
3. Konfigurasi clustering Kubernetes menggunakan Kubeadm.

## Architecture Clusters.

Disini kita menggunakan 3 master, 3 worker, 2 loadbalancer, dan 1 IP untuk digunakan sebagai Virtual IP Address. VIP artinya jika terjadi error atau 1 node mati IP dapat di teruskan ke node yang memungkinkan terjadinya flaiover, sehingga tercapai High Availability sesuai dengan tujuan lab kita saat ini.

{{< img src="/posts/kubernetes/topologi-ha-kubernetes.png" height="700" width="600" align="center" title="Topologi" >}}

{{< vs 3 >}}


Perlu digaris bawahi, bahwa HAproxy dan Keepalived tidak di install di node master. HAproxy dan Keepalived hanya di install di node loadbalancer1 dan loadbalancer2

Host yang digunakan:

   | Hostname     | IP Address  | Roles                |
   | ------------ | ----------- | -------------------- |
   | rb-k8s-lb1   | 10.60.60.43 | HAproxy & Keepalived |
   | rb-k8s-lb2   | 10.60.60.44 | HAproxy & Keepalived |
   | rb-k8s-master1   | 10.60.60.51 | master, etcd  |
   | rb-k8s-master2   | 10.60.60.52 | master, etcd |
   | rb-k8s-master3   | 10.60.60.53 | master, etcd |
   | rb-k8s-worker1   | 10.60.60.54 | worker |
   | rb-k8s-worker2   | 10.60.60.55 | worker |
   | rb-k8s-worker3   | 10.60.60.56 | worker |
   |         -        | 10.60.60.45 | VirtualIP |

## Konfigurasi Load Balancing:
Keepalived dan HAproxy di install pada node rb-k8s-lb1 dan rb-k8s-lb2. Dengan sekenario jika salah satu node loadbalancer ada yang down/mati maka Virtual IP akan otomatis menggunakan jalur di node loadbalancer yang sedang running sehingga cluster kubernetes tidak mengalami kegagalan/error.

### Install Keepalived dan HAproxy pada kedua node

1. Jalankan pada node rb-k8s-lb1 dan rb-k8s-lb2:
```
sudo apt install keepalived haproxy psmisc -y
```
2. Lakukan konfigurasi pada haproxy di kedua node loadbalancer:
```
sudo vi /etc/haproxy/haproxy.cfg
```
```
global
    log /dev/log  local0 warning
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
   
   stats socket /var/lib/haproxy/stats
   
defaults
  log global
  option  httplog
  option  dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000
   
frontend kube-apiserver
  bind *:6443
  mode tcp
  option tcplog
  default_backend kube-apiserver
   
backend kube-apiserver
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server rb-k8s-master1 10.60.60.51:6443 check # Replace the IP address with your own.
    server rb-k8s-master2 10.60.60.52:6443 check # Replace the IP address with your own.
    server rb-k8s-master3 10.60.60.53:6443 check # Replace the IP address with your own.
```
3. Restart service load HAproxy:
```
sudo systemctl restart haproxy
```
4. Kemudian restart dan enable service HAproxy:
```
sudo systemctl restart haproxy
```
```
sudo systemctl enable haproxy
```
5. Pastikan service HAproxy active:
```
root@rb-k8s-lb1:~# systemctl status haproxy.service 
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/lib/systemd/system/haproxy.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2022-02-12 17:29:05 UTC; 10min ago
       Docs: man:haproxy(1)
             file:/usr/share/doc/haproxy/configuration.txt.gz
    Process: 463986 ExecStartPre=/usr/sbin/haproxy -f $CONFIG -c -q $EXTRAOPTS (code=exited, status=0/SUCCESS)
   Main PID: 464001 (haproxy)
      Tasks: 5 (limit: 4676)
     Memory: 3.6M
     CGroup: /system.slice/haproxy.service
             ├─464001 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock
             └─464002 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -S /run/haproxy-master.sock

Feb 12 17:29:05 rb-k8s-lb1 systemd[1]: Starting HAProxy Load Balancer...
Feb 12 17:29:05 rb-k8s-lb1 haproxy[464001]: [WARNING] 042/172905 (464001) : parsing [/etc/haproxy/haproxy.cfg:28] : backend 'kube-apiserver' >
Feb 12 17:29:05 rb-k8s-lb1 haproxy[464001]: [NOTICE] 042/172905 (464001) : New worker #1 (464002) forked
Feb 12 17:29:05 rb-k8s-lb1 systemd[1]: Started HAProxy Load Balancer.
Feb 12 17:29:08 rb-k8s-lb1 haproxy[464002]: [WARNING] 042/172908 (464002) : Server kube-apiserver/rb-k8s-master2 is DOWN, reason: Layer4 conn>
Feb 12 17:29:12 rb-k8s-lb1 haproxy[464002]: [WARNING] 042/172912 (464002) : Server kube-apiserver/rb-k8s-master3 is DOWN, reason: Layer4 conn>
```
### Konfigurasi Keepalived
1. Lakukan konfigurasi keepalived pada kedua node rb-k8s-lb1 dan rb-k8s-lb2:
```
sudo /etc/keepalived/keepalived.conf
```
Konfigurasi keepalived.conf pada rb-k8s-lb1:
```
global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}
   
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
   
vrrp_instance haproxy-vip {
  state BACKUP
  priority 100
  interface ens3                       # Network card
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  unicast_src_ip 10.60.60.43      # The IP address of this machine rb-k8s-lb1
  unicast_peer {
    10.60.60.44                         # The IP address of peer machines rb-k8s-lb2
  }
   
  virtual_ipaddress {
    10.60.60.45/24                  # The VIP address
  }
   
  track_script {
    chk_haproxy
  }
}
```
Konfigurasi keepalived.conf pada rb-k8s-lb2:
```
global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}
   
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
   
vrrp_instance haproxy-vip {
  state BACKUP
  priority 100
  interface ens3                       # Network card
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  unicast_src_ip 10.60.60.44      # The IP address of this machine rb-k8s-lb2
  unicast_peer {
    10.60.60.43                         # The IP address of peer machines rb-k8s-lb1
  }
   
  virtual_ipaddress {
    10.60.60.45/24                  # The VIP address
  }
   
  track_script {
    chk_haproxy
  }
}
```
3. Kemudian restart dan enable service keepalived
```
sudo systemctl restart keepalived
```
```
sudo systemctl enable keepalived
```
4. Pastikan service keepalived running
Status keepalived pada rb-k8s-lb1:
```
root@rb-k8s-lb1:~# sudo systemctl status keepalived.service 
● keepalived.service - Keepalive Daemon (LVS and VRRP)
     Loaded: loaded (/lib/systemd/system/keepalived.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-02-02 16:07:10 UTC; 1 weeks 3 days ago
   Main PID: 647 (keepalived)
      Tasks: 2 (limit: 4676)
     Memory: 6.5M
     CGroup: /system.slice/keepalived.service
             ├─647 /usr/sbin/keepalived --dont-fork
             └─708 /usr/sbin/keepalived --dont-fork

Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: Registering Kernel netlink command channel
Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: Opening file '/etc/keepalived/keepalived.conf'.
Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: WARNING - default user 'keepalived_script' for script execution does not exist - please crea>
Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: WARNING - script `killall` resolved by path search to `/usr/bin/killall`. Please specify ful>
Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: SECURITY VIOLATION - scripts are being executed but script_security not enabled.
Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: Registering gratuitous ARP shared channel
Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: (haproxy-vip) Entering BACKUP STATE (init)
Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: VRRP_Script(chk_haproxy) succeeded
Feb 02 16:07:10 rb-k8s-lb1 Keepalived_vrrp[708]: (haproxy-vip) Changing effective priority from 100 to 102
Feb 02 16:07:14 rb-k8s-lb1 Keepalived_vrrp[708]: (haproxy-vip) Entering MASTER STATE
```
Status keepalived pada rb-k8s-lb2:
```
root@rb-k8s-lb2:~# sudo systemctl status keepalived.service 
● keepalived.service - Keepalive Daemon (LVS and VRRP)
     Loaded: loaded (/lib/systemd/system/keepalived.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2022-02-02 16:07:12 UTC; 1 weeks 3 days ago
   Main PID: 647 (keepalived)
      Tasks: 2 (limit: 4676)
     Memory: 6.6M
     CGroup: /system.slice/keepalived.service
             ├─647 /usr/sbin/keepalived --dont-fork
             └─699 /usr/sbin/keepalived --dont-fork

Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: Registering Kernel netlink reflector
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: Registering Kernel netlink command channel
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: Opening file '/etc/keepalived/keepalived.conf'.
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: WARNING - default user 'keepalived_script' for script execution does not exist - please crea>
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: WARNING - script `killall` resolved by path search to `/usr/bin/killall`. Please specify ful>
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: SECURITY VIOLATION - scripts are being executed but script_security not enabled.
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: Registering gratuitous ARP shared channel
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: (haproxy-vip) Entering BACKUP STATE (init)
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: VRRP_Script(chk_haproxy) succeeded
Feb 02 16:07:13 rb-k8s-lb2 Keepalived_vrrp[699]: (haproxy-vip) Changing effective priority from 100 to 102
```
## Verivikasi High Availability
Sebelum kita membuat cluster kubernetes kita harus memastikan konfigurasi high availabilty kita berfungsi dengan semestinya

1. Cek IP Address pada node rb-k8s-lb1:
```
root@rb-k8s-lb1:~# ip a
1: lo:  mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3:  mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:72:9a:4b brd ff:ff:ff:ff:ff:ff
    inet 10.60.60.43/24 brd 10.60.60.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet 10.60.60.45/24 scope global secondary ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe72:9a4b/64 scope link 
       valid_lft forever preferred_lft forever
```
Bisa kita lihat bahwa VIP 10.60.60.45 berhasil ditambahkan pada node rb-k8s-lb1. kemudian kita simulasikan jika node rb-k8s-lb1 down, maka seharusnya VIP 10.60.60.45 akan otomatis berpindah ke node rb-k8s-lb2

2. Kita matikan service HAproxy pada node rb-k8s-lb1:
```
sudo systemctl stop haproxy.service
```
3. Lalu kita cek lagi IP Address pada node rb-k8s-lb1:
```
root@rb-k8s-lb1:~# ip a
1: lo:  mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3:  mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:72:9a:4b brd ff:ff:ff:ff:ff:ff
    inet 10.60.60.43/24 brd 10.60.60.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe72:9a4b/64 scope link 
       valid_lft forever preferred_lft forever
```
Pada node rb-k8s-lb1 VIP 10.60.60.45 sudah tidak ada, maka kita harus pastikan jika VIP berada pada node rb-k8s-lb2

4. Cek IP Address pada node rb-k8s-lb2:
```

root@rb-k8s-lb2:~# ip a
1: lo:  mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens3:  mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:aa:3d:02 brd ff:ff:ff:ff:ff:ff
    inet 10.60.60.44/24 brd 10.60.60.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet 10.60.60.45/24 scope global secondary ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:feaa:3d02/64 scope link 
       valid_lft forever preferred_lft forever
```
Sesuai harapan kita bahwa VIP sudah berpindah ke node rb-k8s-lb2 jika disimulasikan node rb-k8s-lb1 mati dengan menonaktifkan service haproxy pada rb-k8s-lb1.

5. Jangan lupa kita start lagi service haproxy pada node rb-k82-lb1:
```
sudo systemctl start haproxy.service
```

Pada postingan berikutnya kita akan melanjutkan dengan mengkonfigurasi cluster kubernetes menggunakan alamat VIP yang sudah kita buat tadi