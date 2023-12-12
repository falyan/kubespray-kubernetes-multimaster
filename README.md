# kubespray-kubernetes-multimaster
Kubernetes with ansible using Kubespray

## Topologi
![Alt text](image.png)

- Minimum system requirements </br>
Installation with kubespray minimal 3 master node and 3 worker node, and for specification below:

| RAM    | CPU      | 
| :---   | :------- | 
| `4 GB` | `2 core` | 



- List Server 

| HOSTNAME    | IP            | KETERANGAN                                 |   OS             |
| :--------   | :-------      | :----------------------------------------- | :--------------  |
| `lb-master` | `10.10.90.51` | Load balance for kube api-server port 6443 | Ubuntu 22.04 LTS |
| `master-01` | `10.10.90.52` | Controle plane                             | Ubuntu 22.04 LTS |
| `master-02` | `10.10.90.53` | Controle plane                             | Ubuntu 22.04 LTS |
| `master-03` | `10.10.90.54` | Controle plane                             | Ubuntu 22.04 LTS |
| `worker-01` | `10.10.90.55` | Worker                                     | Ubuntu 22.04 LTS |
| `worker-02` | `10.10.90.56` | Worker                                     | Ubuntu 22.04 LTS |
| `worker-03` | `10.10.90.57` | Worker                                     | Ubuntu 22.04 LTS |

# Step 1 Setup load balancer for kube api server (master)

```bash
apt update
apt install haproxy -y

```
- Configurasi for HAproxy

```bash
vim /etc/haproxy/haproxy.cfg
```
change and add your configuration like bellow

```bash
listen stats
  bind    lb-cluster:9000
  mode    http
  stats   enable
  stats   hide-version
  stats   uri       /stats
  stats   refresh   30s
  stats   realm     Haproxy\ Statistics
  stats   auth      Admin:Password

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    bind lb-cluster:6443
    mode tcp
    option tcplog
    default_backend apiserver

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server master-01 master-01:6443 check
        server master-02 master-02:6443 check
        server master-03 master-03:6443 check

```
- makesure your configuration is valid check with command bellow

```bash
haproxy -f /etc/haproxy/haproxy.cfg -c
```
- if valid restart your haproxy service
```bash
systemctl restart haproxy 
systemctl status haproxy
```
- maksure your port 6443 is open, verification with command bellow
```bash
nc -vvvv lb-cluster 6443
```

# Kubespray installation and configuration (server deployer) 
in this tutorial server deployer is lb-master, Make sure the deployer server can SSH without a password to all nodes (MANDATORY)</br>

- Download kubespray on deployer server
```bash
git clone https://github.com/kubernetes-sigs/kubespray
```
- Install package management for python3 and library for manipulation network
```bash 
apt install python3-netaddr python3-pip -y
```
- install the required python packages

```bash
pip3 install -r requirements.txt
```
- Copy directory sample to new directory (in this tutorial the name of directory is cluster01)
```bash
cd kubespray
cp -rfp inventory/sample inventory/cluster01
```
- add config inventory playbooks
```bash
vi inventory/cluster01/inventory.ini
``` 
replace config bellow to inventory.ini
```bash
[all]
master-01 ansible_host=10.10.90.52 
master-02 ansible_host=10.10.90.53
master-03 ansible_host=10.10.90.54
worker-01 ansible_host=10.10.90.55
worker-02 ansible_host=10.10.90.56
worker-03 ansible_host=10.10.90.57

[kube-master]
master-01
master-02
master-03


[etcd]
master-01
master-02
master-03

[kube-node]
worker-01
worker-02
worker-03

[calico_rr]

[k8s-cluster:children]
kube-master
kube-node
calico_rr

```