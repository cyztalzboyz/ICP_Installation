# Day 1: Lab Exercise

## 1. Prepare OS 

* **Stop and Disable Firewall**
```
systemctl stop firewalld 
systemctl disable firewalld 
```

* **Disable SELINUX**
```
setenforce 0 
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config 
```

* **Set Ulimites**
```
grep -q -F '* soft nofile 102400' /etc/security/limits.conf || echo '* soft nofile 102400' >> /etc/security/limits.conf 
grep -q -F '* hard nofile 102400' /etc/security/limits.conf || echo '* hard nofile 102400' >> /etc/security/limits.conf 
```

* **Disable Swap**
```
swapoff -a 
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab 
echo vm.swappiness=0 >> /etc/sysctl.conf 
sysctl -p 
```

* **Disable IPv6**
```
sed -i '/^::1/ s/^\(.*\)$/#\1/g' /etc/hosts 
```

* **Update latest patches and install standard package**
```
yum update -y 
yum install -y ntpdate bash-completion open-vm-tools bind-utils net-tools lsof psmisc lvm2 
```

* **Reboot Server**
```
systemctl reboot 
```

## 2. Install Docker 

* **Create VG Docker**
```
echo -e "n\n\n\n\n\nt\n8e\nw" | fdisk /dev/xvdc 
vgcreate vg_docker /dev/xvdc1 
```

* **Create FileSystem Docker**
```
lvcreate -L 100G -n lv_dockerlib vg_docker 
mkfs -t xfs /dev/mapper/vg_docker-lv_dockerlib 
mkdir -p /var/lib/docker 
grep -q -F '/dev/mapper/vg_docker-lv_dockerlib /var/lib/docker xfs defaults 0 0' /etc/fstab || echo '/dev/mapper/vg_docker-lv_dockerlib /var/lib/docker xfs defaults 0 0' >> /etc/fstab 
mount -a 
```

* **Create FileSystem Thinpool**
```
lvcreate --wipesignatures y -L 80G -n lv_thinpool vg_docker 
lvcreate --wipesignatures y -L 1G -n lv_thinpoolmeta vg_docker 
lvconvert -y --zero n -c 512K --thinpool vg_docker/lv_thinpool --poolmetadata vg_docker/lv_thinpoolmeta 
cat <<EOF > /etc/lvm/profile/vg_docker-lv_thinpool.profile 
activation { 
  thin_pool_autoextend_threshold=80 
  thin_pool_autoextend_percent=20 
} 
EOF 
lvchange --metadataprofile vg_docker-lv_thinpool vg_docker/lv_thinpool 
lvs -o+seg_monitor
```

* **Install ICP Docker Package**
```
chmod a+x /tivrepos/icp3100/icp-docker-18.03.1_x86_64.bin 
/tivrepos/icp3100/icp-docker-18.03.1_x86_64.bin --install 
systemctl stop docker 
sed -i 's|--storage-driver=devicemapper$|--storage-driver=devicemapper --storage-opt dm.thinpooldev=/dev/mapper/vg_docker-lv_thinpool|' /usr/lib/systemd/system/docker.service 
systemctl daemon-reload 
systemctl start docker 
systemctl enable docker 
```

* **Preload ICP Docker Image**
```
tar -zxvf /tivrepos/icp3100/ibm-cloud-private-x86_64-3.1.0.tar.gz -O | docker load 
```

## 3. Install Gluster & Heketi 

* **Initial Variable**
```
IPADDR=$(ifconfig eth1 | sed -En 's/127.0.0.1//;s/.*inet (addr:)?(([0-9]*\.){3}[0-9]*).*/\2/p') 
HEKETI_CLI_KEY="Password" 
```

* **Create Gluster Repo**
```
cat <<EOF > /etc/yum.repos.d/gluster.repo 
[gluster] 
name=Gluster 
baseurl=http://mirror.centos.org/centos/7/storage/x86_64/gluster-4.1/ 
gpgcheck=0 
enabled=1 
EOF 
```

* **Install Glusterfs Server**
```
yum install -y glusterfs-server 
systemctl enable glusterd 
systemctl start glusterd 
systemctl status glusterd 
```

* **Setup Glusterfs Volume**
```
mkdir -p /heketidb/brick 
gluster volume create heketi-db ${IPADDR}:/heketidb/brick force 
gluster volume start heketi-db 
gluster volume info 
gluster volume status 
```

* **Install and Config Heketi**
```
yum install -y heketi 
cp -p /etc/heketi/heketi.json /etc/heketi/heketi.json_orig 
sed -i 's/8080/8081/g' /etc/heketi/heketi.json 
sed -i "s/My Secret/${HEKETI_CLI_KEY}/g" /etc/heketi/heketi.json 
sed -i 's/"mock",/"ssh",/g' /etc/heketi/heketi.json 
sed -i 's/"path\/to\/private_key"/"\/etc\/heketi\/heketi_key"/g' /etc/heketi/heketi.json 
sed -i 's/"sshuser"/"root"/g' /etc/heketi/heketi.json 
sed -i 's/"Optional: ssh port.  Default is 22"/"22"/g' /etc/heketi/heketi.json 
sed -i 's/"Optional: Specify fstab file on node.  Default is \/etc\/fstab"/"\/etc\/fstab"/g' /etc/heketi/heketi.json 
```

* **Generate and Copy SSH Key**
```
ssh-keygen -b 4096 -f ~/.ssh/id_rsa -N "" 
ssh-copy-id -i ~/.ssh/id_rsa.pub root@${IPADDR} 
cp ~/.ssh/id_rsa /etc/heketi/heketi_key 
chown heketi:heketi /etc/heketi/heketi_key 
```

* **Fix a Parameter in /usr/lib/systemd/system/heketi.service**
```
sed -i 's/-config/--config/g' /usr/lib/systemd/system/heketi.service 
systemctl daemon-reload 
systemctl enable heketi 
systemctl start heketi 
systemctl status heketi 
```

* **Intall Heketi CLI**
```
yum install -y heketi-client.x86_64 
```

* **Configure Root Profile**
```
echo "export HEKETI_CLI_SERVER=\"http://${IPADDR}:8081\"" >> /root/.bash_profile 
echo "export HEKETI_CLI_USER=\"admin\"" >> /root/.bash_profile 
echo "export HEKETI_CLI_KEY=\"${HEKETI_CLI_KEY}\"" >> /root/.bash_profile 
ssh root@${IPADDR} 
env | grep HEKETI
```

* **Create Heketi-Gluster**
```
heketi-cli topology info
mkdir -p /var/lib/heketi-gluster
chown heketi: /var/lib/heketi-gluster
echo "localhost:/heketidbstorage /var/lib/heketi-gluster glusterfs defaults,_netdev  0 0" >> /etc/fstab
cat <<EOF > /var/lib/heketi-gluster/topology.json
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "${IPADDR}"
              ],
              "storage": [
                "${IPADDR}"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/xvdf"
          ]
        }
      ]
    }
  ]
}
EOF
```

* **Backup Heketi DB**
```
cp -p /var/lib/heketi/heketi.db /var/lib/heketi/heketi.db_orig
```

* **Create Topology**
```
cd /var/lib/heketi-gluster/
heketi-cli topology load --json=topology.json
heketi-cli topology info
heketi-cli volume create --name=heketidbstorage --size=1 --durability="none"
heketi-cli topology info
gluster volume set heketi-db auth.allow ${IPADDR}
gluster volume set heketidbstorage auth.allow ${IPADDR}
gluster volume info
```

## 4. Install ICP 

* **Initial Variable**
```
HOSTNAME=$(hostname -s)
IPADDR=$(ifconfig eth1 | sed -En 's/127.0.0.1//;s/.*inet (addr:)?(([0-9]*\.){3}[0-9]*).*/\2/p')
DEFAULT_ADMIN_PASSWORD="Il0veyou!"
HEKETI_CLUSTER_ID=$(heketi-cli topology info | grep ^Cluster | awk '{print $3}')
```

* **Prepare ICP VG And Kubelet LV**
```
echo -e "n\n\n\n\n\nt\n8e\nw" | fdisk /dev/xvde
vgcreate vg_icp /dev/xvde1
lvcreate -L 60G -n lv_kubelet vg_icp
mkfs -t xfs /dev/mapper/vg_icp-lv_kubelet
mkdir -p /var/lib/kubelet
grep -q -F '/dev/mapper/vg_icp-lv_kubelet /var/lib/kubelet xfs defaults 0 0' /etc/fstab || echo '/dev/mapper/vg_icp-lv_kubelet /var/lib/kubelet xfs defaults 0 0' >> /etc/fstab
mount -a
```


* **Tune Kernel Parameter max_map_count to 262144**
```
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
sysctl -p
```



* **Prepare LV For Etcd And Mysql**
```
lvcreate -L 10G -n lv_etcd vg_icp
mkfs -t xfs /dev/mapper/vg_icp-lv_etcd
mkdir -p /var/lib/etcd
grep -q -F '/dev/mapper/vg_icp-lv_etcd /var/lib/etcd xfs defaults 0 0' /etc/fstab || echo '/dev/mapper/vg_icp-lv_etcd /var/lib/etcd xfs defaults 0 0' >> /etc/fstab
mount -a
lvcreate -L 10G -n lv_mysql vg_icp
mkfs -t xfs /dev/mapper/vg_icp-lv_mysql
mkdir -p /var/lib/mysql
grep -q -F '/dev/mapper/vg_icp-lv_mysql /var/lib/mysql xfs defaults 0 0' /etc/fstab || echo '/dev/mapper/vg_icp-lv_mysql /var/lib/mysql xfs defaults 0 0' >> /etc/fstab
mount -a
```

* **Prepare LV For ICP Application**
```
lvcreate -L 60G -n lv_icp vg_icp
mkfs -t xfs /dev/mapper/vg_icp-lv_icp
mkdir -p /var/lib/icp
grep -q -F '/dev/mapper/vg_icp-lv_icp /var/lib/icp xfs defaults 0 0' /etc/fstab || echo '/dev/mapper/vg_icp-lv_icp /var/lib/icp xfs defaults 0 0' >> /etc/fstab
mount -a
```

* **Copy ICP Config File Template**
```
mkdir -p /opt/ibm-cloud-private-x86_64-3.1.0
cd /opt/ibm-cloud-private-x86_64-3.1.0
docker run -v $(pwd):/data -e LICENSE=accept ibmcom/icp-inception-amd64:3.1.0-ee cp -r cluster /data
```

* **Share SSH Key to All ICP Nodes**
```
cat ~/.ssh/id_rsa > /opt/ibm-cloud-private-x86_64-3.1.0/cluster/ssh_key
```

* **Backup Original ICP Deployment Configuration File**
```
cp -p /opt/ibm-cloud-private-x86_64-3.1.0/cluster/config.yaml /opt/ibm-cloud-private-x86_64-3.1.0/cluster/config.yaml_orig
```

* **Add ICP Nodes to ICP Ansible Inventory File Cluster/Hosts**
```
cat <<EOF > /opt/ibm-cloud-private-x86_64-3.1.0/cluster/hosts
[master]
${IPADDR}

[worker]
${IPADDR}

[proxy]
${IPADDR}

EOF

cat /opt/ibm-cloud-private-x86_64-3.1.0/cluster/hosts
```

* **Modify ICP Deployment Configuration**
```
sed -i "s/# cluster_name: mycluster/cluster_name: ${HOSTNAME}/g" /opt/ibm-cloud-private-x86_64-3.1.0/cluster/config.yaml
sed -i "s/default_admin_password: admin/default_admin_password: ${DEFAULT_ADMIN_PASSWORD}/g" /opt/ibm-cloud-private-x86_64-3.1.0/cluster/config.yaml
sed -i 's/# auditlog_enabled: false/auditlog_enabled: true/g' /opt/ibm-cloud-private-x86_64-3.1.0/cluster/config.yaml
sed -i 's/# metrics_max_age: 1/metrics_max_age: 14/g' /opt/ibm-cloud-private-x86_64-3.1.0/cluster/config.yaml
sed -i 's/# logs_maxage: 1/logs_maxage: 14/g' /opt/ibm-cloud-private-x86_64-3.1.0/cluster/config.yaml

cat <<EOF >> /opt/ibm-cloud-private-x86_64-3.1.0/cluster/config.yaml
monitoring:
  prometheus:
    scrapeInterval: 1m
    evaluationInterval: 1m
    retention: 2d
    persistentVolume:
      enabled: true
      storageClass: "gluster-distributed"
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
  alertmanager:
    persistentVolume:
      enabled: true
      storageClass: "gluster-distributed"
    resources:
      limits:
        cpu: 200m
        memory: 256Mi
  grafana:
    user: "admin"
    password: "${DEFAULT_ADMIN_PASSWORD}"
    persistentVolume:
      enabled: true
      storageClass: "gluster-distributed"
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
EOF
```

* **Create Storage Class File**
```
cat <<EOF > /opt/ibm-cloud-private-x86_64-3.1.0/cluster/misc/storage_class/class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gluster-distributed
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://${IPADDR}:8081"
  clusterid: "${HEKETI_CLUSTER_ID}"
  restauthenabled: "false"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "none"
EOF
```

* **Copy ICP Installation Images to ICP Directory**
```
mkdir -p /opt/ibm-cloud-private-x86_64-3.1.0/cluster/images;
mv /tivrepos/icp3100/ibm-cloud-private-x86_64-3.1.0.tar.gz  /opt/ibm-cloud-private-x86_64-3.1.0/cluster/images/
```

* **Ensure Docker and Heketi Is Running**
```
systemctl status docker
systemctl status heketi
```

* **Run ICP Deployment**
```
cd /opt/ibm-cloud-private-x86_64-3.1.0/cluster/
docker run --net=host -t -e LICENSE=accept -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.1.0-ee install
```

* **Run Kubectl Docker For Using Kubectl Command**
```
docker run -e LICENSE=accept --net=host -v /usr/local/bin:/data ibmcom/icp-inception-amd64:3.1.0-ee cp /usr/local/bin/kubectl /data
```


