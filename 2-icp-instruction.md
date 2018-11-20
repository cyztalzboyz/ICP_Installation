# Day 2: Lab Exercise
## 1. Docker Pull/Tag/Push Image 

* **Initial Variable**
```
HOSTNAME=$(hostname -s)
```

* **Docker Login**
```
docker login ${HOSTNAME}.icp:8500
```

* **Docker Pull Images**
```
docker pull bankwing/hello-world:0.1
docker pull bankwing/hello-world:0.2
```

* **Docker Tag**
```
docker tag bankwing/hello-world:0.1 ${HOSTNAME}.icp:8500/default/hello-world:0.1
docker tag bankwing/hello-world:0.2 ${HOSTNAME}.icp:8500/default/hello-world:0.2
```

* **Docker Push**
```
docker push ${HOSTNAME}.icp:8500/default/hello-world:0.1
docker push ${HOSTNAME}.icp:8500/default/hello-world:0.2
```

* **Docker Remove Images**
```
docker rmi bankwing/hello-world:0.1
docker rmi bankwing/hello-world:0.2
docker rmi ${HOSTNAME}.icp:8500/default/hello-world:0.1
docker rmi ${HOSTNAME}.icp:8500/default/hello-world:0.2
```

## 2. Adding a worker node 

* **Change to the cluster directory within your IBM Cloud Private installation directory.**
```
cd /opt/ibm-cloud-private-x86_64-3.1.0/cluster
```
* **Ensure that the installer is available in your /opt/ibm-cloud-private-x86_64-3.1.0/cluster/images directory.**
* **To add a worker node for Linux® 64-bit, run the following command:**
```
docker run -e LICENSE=accept --net=host -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.1.0-ee worker -l ip_address_workernode1,ip_address_workernode2
```

## 3. Remove a cluster node from your IBM® Cloud Private cluster.

* **Change to the cluster directory within your IBM Cloud Private installation directory.**
```
cd /opt/ibm-cloud-private-x86_64-3.1.0/cluster/
```
* **To remove a node for Linux® 64-bit, run the following command:**
```
docker run -e LICENSE=accept --net=host -v "$(pwd)":/installer/cluster ibmcom/icp-inception-amd64:3.1.0-ee uninstall -l ip_address_clusternode1,ip_address_clusternode2
```
