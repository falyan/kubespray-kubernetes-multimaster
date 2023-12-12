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
| `lb-master` | `10.10.90.51` | Load balance for kube api-server port 6443 | Ubuntu 20.04 LTS |
| `master-01` | `10.10.90.52` | Controle plane                             | Ubuntu 20.04 LTS |
| `master-02` | `10.10.90.53` | Controle plane                             | Ubuntu 20.04 LTS |
| `master-03` | `10.10.90.54` | Controle plane                             | Ubuntu 20.04 LTS |
| `worker-01` | `10.10.90.55` | Worker                                     | Ubuntu 20.04 LTS |
| `worker-02` | `10.10.90.56` | Worker                                     | Ubuntu 20.04 LTS |
| `worker-03` | `10.10.90.57` | Worker                                     | Ubuntu 20.04 LTS |

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
  bind    10.10.90.51:9000
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
    bind 10.10.90.51:6443
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
        server master-01 10.10.90.52:6443 check
        server master-02 10.10.90.53:6443 check
        server master-03 10.10.90.54:6443 check

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
in this tutorial server deployer is lb-master (10.10.90.51), Make sure the deployer server can SSH without a password to all nodes (MANDATORY)</br>

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

- Verify connection all node from server deployer, to check the connectivity for the hosts in your inventory
```bash
ansible -m ping -i inventory/cluster01/inventory.ini --become --become-user=root all
```
if "SUCCESS" indicates that the host is reachable and "ping": "pong" indicates that the ping module was executed successfully on the remote host

- set ip forwarding for allow the forwarding of IP packets between different network interfaces on the system **(DO ALL NODE)**
```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
```
- apply ip forwarding
```bash
sysctl --system
```
- change configuration group var and find config bellow
```bash
vim inventory/cluster01/group_vars/all/all.yml
```
```bash
bin_dir: /usr/local/bin
loadbalancer_apiserver:
  address: "10.10.90.51"
  port: "6443"

```
- change configuration cluster and find config bellow

```bash
vim inventory/cluster01/group_vars/k8s_cluster/k8s-cluster.yml
```

```bash
container_manager: containerd
cluster_name: cluster.local
kube_network_plugin: calico
k8s_image_pull_policy: IfNotPresent
supplementary_addresses_in_ssl_keys: [10.10.90.52, 10.10.90.53, 10.10.90.52, 10.10.90.51]
```
# running palybook  
before running playbook best practice is run tmux first for keep your session server
```bash
tmux
```
- run your playbooks to install your cluster
```bash
ansible-playbook -i inventory/cluster01/inventory.ini --become --become-user=root cluster.yml
```

this part will take a several minutes depent your connection

## 🔗 About me
[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/falyan-zuril-587585247/)
