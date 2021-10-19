
# **Overview**

This tutorial will guide you through working examples that will allow You to **experiment with different options to provide persistence storage to containers in different scenarios** as per this blogpost (LINK TBD).

We will details the steps for these use cases
1. Single Container running with persistent storage
2. Single Container with persistent storage running across multiple Container Hosts
3. Single Container with persistent storage running across multiple Container Hosts and multiple Availability Zones
4. Single container with single persistent datastore that spans across Availability Zones

Each setup will provide prerequisistes, instructions and clean up procedures.

**All code is provided** <u>**AS IS**</u>**: it is not meant for production workloads but for test environments only.**

You may incur in costs for testing these setups so we recommend to take this into account and tear down the environments aftre performing your tests.

# Single Container running with persistent storage

**Overview**

This tutorial will drive you through the creation of docker container that leverages backing storage for containers provided by Docker Volume.

![Alt text](/images/SingleContainerDockerVolume.png "SingleContainerDockerVolume")


This setup uses AWS Cloud9 as an environment to deploy your container and storage used by Docker volume is local to that Cloud9 instance.


If you want to test how to persist data for a single container running on a single Container host out you could use AWS Cloud9 ([https://aws.amazon.com/cloud9/](https://aws.amazon.com/cloud9/)). We recommend using an AWS Cloud9 desktop since this provides an environment to run any commands without needing to modify your own systems, and it provides a consistent experience that has been tested by the author. 
To deploy Cloud9 you can use this guide provide as part of the AWS EKS worshop 
[https://www.eksworkshop.com/020_prerequisites/workspace/](https://www.eksworkshop.com/020_prerequisites/workspace/)  
  
Here are the quick steps for **testing on AWS Cloud9 and Docker**.  
  
Install docker-compose ([https://docs.docker.com/compose/](https://docs.docker.com/compose/))  

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Now let’s deploy a MySQL container that uses a **docker volume to store persistent MySQL data**:  
```
cat << EoF > ${HOME}/mysqldocker.yaml  
version: '3'  
services:  
  db:  
    image: mysql:5.7  
    container_name: db  
    ports:  
      - "6033:3306"  
    environment:
      MYSQL_ROOT_PASSWORD: aVeryBadPassword   
    volumes:  
      - dbdata:/var/lib/mysql        
volumes:  
  dbdata:  
EoF
```
Launch the container:  
```
docker-compose -f ${HOME}/mysqldocker.yaml up -d
```
Now check that it is running and using a docker volume construct:  
```
docker ps  
  
docker volume ls
```
Take note of the container id

![Alt text](/images/0b-dockervolume.png "0b-dockervolume")


Now let’s see that the volume is using Docker volumes:
```
docker inspect *containerid*
```
![Alt text](/images/1b-inspect.png "1b-inspect")

We can see that the volume is being correctly configured by inspecting the Mounts configuration:

![Alt text](/images/2b-mounts.png "2b-mounts")

Let's see how this work: we will write a file inside the container and see that it gets written to the container host.

Connect to the container:
```
docker exec -it *containerid* bash
```

Create a file in the mount and exit:
```
touch /var/lib/mysql/testmount
exit
```

Now that we are back in the container host let's check taht the file is actually written locally:
```
ls /var/lib/docker/volumes/ec2-user_dbdata/_data
```

![Alt text](/images/DockerVolumeMount.png "DockerVolumeMount")


We have created a MySQL container that effectively uses a volume external from the container.

## Clean up Instructions

Remember to **clean up the deployment** by using:
```
docker stop containerid
docker-compose rm db
```

![Alt text](/images/DockerVolumeCleanup.png "DockerVolumeCleanup")


And deleting the AWS Cloud9 instances following these instructions (as per the eksworkshop clean up):
1. Go to your Cloud9 Environment
2. Select the environment named eksworkshop and pick delete

# Single Container with persistent storage running across multiple Container Hosts
This part of the tutorial will drive you through the creation of Kubernetes cluster (on Amazon EKS), configuration of backing storage for containers (Amazon EBS) and in the end you’ll be able to deploy a container with SQL server that can withstand the loss of a Container Host.

![Alt text](/images/SingleContainerMultipleContainerHosts.png "SingleContainerMultipleContainerHosts")


## Prerequisites
Create an EKS Cluster following the tutorial at https://www.eksworkshop.com/030_eksctl/prerequisites/




# Single Container with persistent storage running across multiple Container Hosts and multiple Availability Zones

This part of the tutorial will drive you through the creation of Kubernetes cluster (on Amazon EKS), configuration of backing storage for containers (Amazon EBS) and in the end you’ll be able to deploy a Cassandra cluster that can withstand the loss of a full Availability Zone. A procedure for testing the setup by simulating an AZ failure is provided as well.

![Alt text](/images/CassandraSetup.png "CassandraSetup")



**Prerequisites**  
  
You’d need to create an Amazon EKS cluster. To do so it’s very easy to leverage our EKS Workshop ([https://www.eksworkshop.com](https://www.eksworkshop.com)) as it provides a simple yet powerful environment that comes also with a handy AWS Cloud9 instance that you can use to issue the commands below to your Amazon EKS cluster.  
  
To create an EKS Cluster you can follow these steps: [https://www.eksworkshop.com/020_prerequisites/self_paced/account/](https://www.eksworkshop.com/020_prerequisites/self_paced/account/)  
  
If you want to perform the final failover test (see below) create the cluster using t3.medium (instead of t3.small) instances, i.e. when following the instructions at:  
[https://www.eksworkshop.com/030_eksctl/launcheks/](https://www.eksworkshop.com/030_eksctl/launcheks/)  
  
Change the line from the eksworkshop.yaml file from:  

```
instanceType: t3.small
```

To  

```
instanceType: t3.medium
```

You’ll incur in AWS costs to test this setup so make sure you are tearing down the environment after your tests by using this walk-through when you’re done testing:  
  
[https://www.eksworkshop.com/920_cleanup/](https://www.eksworkshop.com/920_cleanup/)  
  
_**Kubernetes Cluster Configuration**_  
  
After creating the cluster make sure that the nodes are in different Availabily Zones:  
```
kubectl get nodes -o wide
```

![Alt text](/images/0-kubectlwide.png "0-kubectlwide")


You will see that the Kubernetes nodes will be in different subnets (i.e. in different Availability Zones).  
  
We will be using EBS via CSI Drivers (check the introduction of this blog post if you want to know more about this setup and why it matters [https://aws.amazon.com/blogs/containers/using-ebs-snapshots-for-persistent-storage-with-your-eks-cluster/](https://aws.amazon.com/blogs/containers/using-ebs-snapshots-for-persistent-storage-with-your-eks-cluster/) )  
  
To set up EBS CSI Drivers we will use the instructions provided in our EKS Workshop:  
  
[https://www.eksworkshop.com/beginner/170_statefulset/ebs_csi_driver/](https://www.eksworkshop.com/beginner/170_statefulset/ebs_csi_driver/)  
  
**Define a Storage Class:**  
A StorageClass enables administrators to describe the "classes" of storage they offer. Different classes might map to quality-of-service levels/ storage tiers, or to different environments (test/production), or to ad hoc policies determined by the storage/clusters administrators.  
  
StorageClass is a construct that paired with dynamic provisioning allows an end user to auto create volumes on their own by just leveraging a Storage Class that the admin has made available from them. This way there’s no need for administrators to manually pre-provision volumes: they just provide a "pool" of storage where end users can carve the volumes that they need.  

**Create storage class** 


```
cat << EoF > mysql-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: mysql-gp2
provisioner: ebs.csi.aws.com # Amazon EBS CSI driver
parameters:
  type: gp2
  encrypted: 'true' # EBS volumes will always be encrypted by default
volumeBindingMode: Immediate # EBS volumes are AZ specific
reclaimPolicy: Delete
mountOptions:
- debug
EoF
```


```
kubectl create -f mysql-storageclass.yaml
```

Check that it is correctly created:  

```
kubectl get sc
```

Now we will **create a Cassandra Cluster** leveraging an example straight from Kubernetes documentation:  
[https://kubernetes.io/docs/tutorials/stateful-application/cassandra/](https://kubernetes.io/docs/tutorials/stateful-application/cassandra/)  
  
Create an headless service for Cassandra  
```
cat > cassandra-svc.yaml << EOF  
apiVersion: v1  
kind: Service  
metadata:  
  labels:  
    app: cassandra  
  name: cassandra  
spec:  
  clusterIP: None  
  ports:  
    - port: 9042  
  selector:  
    app: cassandra  
EOF
```
Create the service in Kubernetes:  
```
kubectl create -f cassandra-svc.yaml
```
Now we will create the Cassandra cluster itself (notice that the only change from the Kubernetes documentation example is the name of the storage class to be used, a great plus for workloads portability across different Kubernetes storage options):  
```
cat > cassandra-app.yaml << EOF  
apiVersion: apps/v1  
kind: StatefulSet  
metadata:  
  name: cassandra  
spec:  
  selector:  
    matchLabels:  
      app: cassandra  
  serviceName: cassandra  
  replicas: 3  
  template:  
    metadata:  
      labels:  
        app: cassandra  
    spec:  
      schedulerName: stork  
      containers:  
      - name: cassandra  
        image: cassandra:3  
        ports:  
          - containerPort: 7000  
            name: intra-node  
          - containerPort: 7001  
            name: tls-intra-node  
          - containerPort: 7199  
            name: jmx  
          - containerPort: 9042  
            name: cql  
        env:  
          - name: CASSANDRA_SEEDS  
            value: cassandra-0.cassandra.default.svc.cluster.local  
          - name: MAX_HEAP_SIZE   
            value: 512M  
          - name: HEAP_NEWSIZE  
            value: 512M  
          - name: CASSANDRA_CLUSTER_NAME  
            value: "Cassandra"  
          - name: CASSANDRA_DC  
            value: "DC1"  
          - name: CASSANDRA_RACK  
            value: "Rack1"  
          - name: CASSANDRA_AUTO_BOOTSTRAP  
            value: "false"              
          - name: CASSANDRA_ENDPOINT_SNITCH  
            value: GossipingPropertyFileSnitch  
        volumeMounts:  
        - name: cassandra-data  
          mountPath: /var/lib/cassandra  
  volumeClaimTemplates:  
  - metadata:  
      name: cassandra-data  
      annotations:  
        volume.beta.kubernetes.io/storage-class: mysql-gp2  
      labels:  
         app: cassandra  
    spec:  
      accessModes: [ "ReadWriteOnce" ]  
      resources:  
        requests:  
          storage: 1Gi  
EOF
```
  
Launch the StatefulSet using:  
```
kubectl apply -f cassandra-app.yaml 
```
It will take a while to create the deployment so you can monitor pod creation using:  
```
kubectl get pods --watch
```
And finally (usually after a few minutes) check that the StatefulSet has been deployed correctly:  
```
kubectl get statefulset
```
You can now check that the Cassandra cluster is operational by issuing this command (it will log you into a Cassandra node and check service availability):  
```
kubectl exec cassandra-0 -- nodetool status
```
You should see something like this; indicating that the three Cassandra nodes active (Up-U) are operational (Normal-N):

![Alt text](/images/1-cassandraup.png "1-cassandraup")

You can also check that the containers are running in different AZs, each one with its own dedicated storage  
```
kubectl get pods -o wide

kubectl get pvc
```
You should see three pods for Cassandra nodes and three volumes of 1 GB bound to the relevant pods:

![Alt text](/images/2-volumes.png "2-volumes")


**Failover Test**  
Now we will simulate a failure scenario where one node in one AZ becomes unavailable.  
To see that data gets persisted even in case of a failure let’s write some information into the Cassandra cluster.  
Connect to one node:  
```
kubectl exec -it cassandra-0 — cqlsh
```
And create a small table in Cassandra:  
```
CREATE KEYSPACE awsdemo WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 3 };  
CONSISTENCY QUORUM;  
use awsdemo;  
CREATE TABLE awsregions (regionCode text PRIMARY KEY, city text, country text);  
INSERT into awsregions(regionCode, city, country) values ('eu-south-1','Milan', 'Italy');  
INSERT into awsregions(regionCode, city, country) values ('af-south-1', 'Cape Town', 'South Africa');
```
Check that data has been written and exit from container:  
```
SELECT * FROM awsdemo.awsregions;  
exit
```
You should get an output similar to this:

![Alt text](/images/3-Cassandradata.png "3-Cassandradata")

Check that replication data is present for the Milan region (eu-south-1) data:  
```
kubectl exec -it cassandra-0 -- nodetool getendpoints awsdemo awsregions eu-south-1**_
```
You will see a list of the Cassandra nodes:

![Alt text](/images/3-Cassandraips.png "3-Cassandraips")

Now we will simulate a failure for one AZ (by taking down the related Kubernetes node) and see how the setup behaves.  
Let’s cordon ([https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)) the node where pod cassandra-0 is executing:  
```
NODE=`kubectl get pods cassandra-0 -o json | jq -r .spec.nodeName`

kubectl cordon ${NODE}

kubectl get nodes
```
In the output you’ll see that the container scheduling is now disabled for that node:

![Alt text](/images/5-cordon.png "5-cordon")

Now let’s delete the pod to see if it can restart:

kubectl delete pod cassandra-0

kubectl get pods

kubectl describe pods cassandra-0

The pod will not be able to restart and the last command will produce an output similar to:

![Alt text](/images/6-podscheduling.png "6-podscheduling")

As you can see from the latest output the pod cannot be scheduled as there are no Kubernetes nodes that can connect to the data (as for demo purposes we just had **only one node per AZ in our setup** and the EBS volume for the container exists only in one availability zone):

![Alt text](/images/7-onenodeperaz.png "7-onenodeperaz")

Let's bring the node back to service (simulating an AZ coming back up):  
```
kubectl uncordon $NODE
```
The pod will restart, we can check that with:  
```
kubectl get pods -o wide
```
The Cassandra cluster will be operational again in a few seconds:  
```
kubectl exec cassandra-0 -- nodetool status
```
And finally let’s check that the data is still present:  
```
kubectl exec cassandra-0 -- cqlsh -e 'SELECT * FROM awsdemo.awsregions;'
```
And it is distributed on all Cassandra nodes:  
```
kubectl exec -it cassandra-0 -- nodetool getendpoints awsdemo awsregions eu-south-1
```
The **Cassandra cluster is back online** and **data has been preserved**.  
 
We have **demonstrated how this type of setup can withstand the loss of a node in one AZ** and **how Amazon EBS storage plays a role in persisting the relevant data** for the application.

## Clean up Instructions




## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

