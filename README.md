# Highly Available PostgreSQL Cluster Using Patroni and HAProxy

## **Architecture**
- **etcd node**: 10.0.0.10 (Stores cluster state)
- **Node1 (Primary)**: 10.0.0.5
- **Node2 (Replica)**: 10.0.0.7
- **Node3 (Replica)**: 10.0.0.8
- **HAProxy node**: 192.168.68.140

## **Step 1: Setup PostgreSQL Nodes (node1, node2)**

Run these commands on **node1** and **node2**:
```bash
# Step 1: Update and install dependencies
sudo apt update
sudo apt install -y \
    net-tools \
    postgresql \
    postgresql-server-dev-16 \
    python3 \
    python3-pip \
    python3-full \
    python3-venv \
    build-essential \
    libpq-dev

# Step 2: Optional - Stop PostgreSQL if using Patroni
sudo systemctl stop postgresql

# Step 3: Set hostname (change node1 as needed)
sudo hostnamectl set-hostname node1

# Step 4: Create Python virtual environment
python3 -m venv ~/patroni-venv
source ~/patroni-venv/bin/activate

# Step 5: Upgrade pip and setuptools
pip install --upgrade pip setuptools

# Step 6: Install Python packages (with fallback binary psycopg2)
pip install psycopg2 || pip install psycopg2-binary
pip install patroni python-etcd

# Step 7: Verify packages (optional)
pip list
sudo apt update
sudo apt install postgresql -y
sudo mkdir -p /data/patroni
sudo chown -R postgres:postgres /data/patroni
sudo chown -R ubuntu:ubuntu /data/patroni
```
```
sudo systemctl enable patroni

```

## **Step 2: Setup etcd Node**

Run these commands on **etcd node (10.0.0.10)**:
```bash
sudo apt update
sudo apt install -y net-tools curl tar
sudo hostnamectl set-hostname etcdnode
ETCD_VER=v3.5.13
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
cd etcd-${ETCD_VER}-linux-amd64
sudo mv etcd etcdctl /usr/local/bin/
cd ~
sudo useradd -r -s /sbin/nologin etcd
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
```
Edit the etcd configuration:
```bash
sudo nano /etc/default/etcd
```
Add the following lines:
```bash
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="http://10.0.0.10:2380"
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://10.0.0.10:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.10:2380"
ETCD_INITIAL_CLUSTER="default=http://10.0.0.10:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.10:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```
Restart and verify etcd:
```bash
sudo chown -R etcd:etcd /var/lib/etcd
sudo systemctl daemon-reload
sudo systemctl restart etcd
sudo systemctl status etcd   
curl http://10.0.0.10:2379/members
sudo ufw allow 2379

```

## **Step 3: Setup HAProxy Node**

Run these commands on **HAProxy node (192.168.68.140)**:
```bash
sudo apt update
sudo hostnamectl set-hostname haproxynode
sudo apt install -y net-tools haproxy
```

## **Step 4: Configure Patroni on node1**

Create Patroni config file:
```bash
sudo nano /etc/patroni.yml
```
Add the following:
```yaml
scope: postgres
namespace: /db/
name: node1
restapi:
    listen: 10.0.0.5:8008
    connect_address: 10.0.0.5:8008
etcd:
    host: 10.0.0.10:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - host replication replicator 10.0.0.0/28 md5
  - host replication replicator 10.0.0.5/0 md5
  - host replication replicator 10.0.0.7/0 md5
  - host replication replicator 10.0.0.8/0 md5
  - host all all 0.0.0.0/0 md5
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb
postgresql:
  listen: 10.0.0.5:5432
  connect_address: 10.0.0.5:5432
  bin_dir: /usr/lib/postgresql/16/bin
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: admin@123
    superuser:
      username: postgres
      password: admin@123
  parameters:
      unix_socket_directories: '.'
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false

```

Create data directory:
```bash
sudo mkdir -p /data/patroni
sudo chown postgres:postgres /data/patroni
sudo chmod 700 /data/patroni
```

Create systemd service:
```bash
sudo nano /etc/systemd/system/patroni.service
```
Add the following:
```ini
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
ExecStart=/home/ubuntu/patroni-venv/bin/patroni /etc/patroni.yml
Restart=on-failure
KillMode=process
TimeoutSec=30
Environment=PATH=/home/ubuntu/patroni-venv/bin:/usr/bin:/bin

[Install]
WantedBy=multi-user.target

```
```
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
```

## **Step 5: Configure Patroni on node2**

Repeat the same steps as **node1**, but update `patroni.yml`:
```yaml
scope: postgres
namespace: /db/
name: node2
restapi:
    listen: 10.0.0.7:8008
    connect_address: 10.0.0.7:8008
etcd:
    host: 10.0.0.10:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - host replication replicator 10.0.0.0/28 md5
  - host replication replicator 10.0.0.5/0 md5
  - host replication replicator 10.0.0.7/0 md5
  - host replication replicator 10.0.0.8/0 md5
  - host all all 0.0.0.0/0 md5
  users:
    admin:
      password: admin
      options:
        - createrole
        - createdb
postgresql:
  listen: 10.0.0.7:5432
  connect_address: 10.0.0.7:5432
  bin_dir: /usr/lib/postgresql/16/bin
  data_dir: /data/patroni
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: admin@123
    superuser:
      username: postgres
      password: admin@123
  parameters:
      unix_socket_directories: '.'
tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

## **Step 6: Start Patroni on All Nodes**
Run on **node1** and **node2**:
```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl restart patroni
sudo systemctl status patroni
```
Verify Patroni cluster:
```bash
patronictl -c /etc/patroni.yml list
```

## **Step 7: Configure HAProxy**

Edit HAProxy configuration:
```bash
sudo nano /etc/haproxy/haproxy.cfg
```
Add the following:
```ini
global
    maxconn 100
    log     10.0.0.0 local2

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.0.0.5:5432 maxconn 100 check port 8008
    server node2 10.0.0.7:5432 maxconn 100 check port 8008
    server node3 10.0.0.8:5432 maxconn 100 check port 8008

```
Restart HAProxy:
```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

## **Step 8: Verify Cluster Functionality**



Check PostgreSQL connection via Node1 .
```bash
psql -h 10.0.0.5 -p 5432 -U postgres -W
```
Check PostgreSQL connection via Node2 .
```bash
psql -h 10.0.0.7 -p 5432 -U postgres -W
```
Check PostgreSQL connection via Node3 .
```bash
psql -h 10.0.0.8 -p 5432 -U postgres -W
```
Check PostgreSQL connection via HAProxy:
```bash
psql -h 192.168.68.140 -p 5000 -U postgres -W
```

Check HAProxy status:
```bash
sudo systemctl status haproxy
```

Check Patroni logs:
```bash
sudo journalctl -u patroni --no-pager
```
## **Step 9: To verify the failover**
Stop the master node to check the replica becomes master or not 
```bash
    sudo systemctl stop patroni
```
now check the patroni status 
```bash 
    patronictl -c /etc/patroni.yml list
```


## As  you can see the graphical representation of High Availabilty and Failover Cluster 

![alt text](img/CMS_2020.05_HA_SQL_Patroni_Arch.png)
Your **Highly Available PostgreSQL Cluster** with Patroni and HAProxy is now ready!

