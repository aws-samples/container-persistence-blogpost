# Single Container with persistent storage running across multiple Container Hosts and multiple Availability Zones

There are two high level scenarios in this use case:
1. Active/Standby Databases on two Containers with independent persistent datastores
2. Active/Active databases on 2+ containers with independent persistent datastores

**All code is provided** <u>**AS IS**</u>**: it is not meant for production workloads but for test environments only.**

You may incur in costs for testing this setup so we recommend to take this into account and tear down the environments after performing your tests.


# Active/Standby Databases on two Containers with independent persistent datastores

If you want to try out this example a step by step setup is provided in our EKS Workshop, it covers exactly a *MySQL database setup over two availability zones*:
_https://www.eksworkshop.com/beginner/170_statefulset/_ (https://www.eksworkshop.com/beginner/170_statefulset/)

![Alt text](/images/activestandbyMySQL.png "activestandbyMySQL")


# Active/Active databases on 2+ containers with independent persistent datastores

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

Or just use this file:

```
cat << EOF > eksworkshop-medium.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  region: ${AWS_REGION}
  version: "1.21"

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.medium
  ssh:
    enableSsm: true

# To enable all of the control plane logs, uncomment below:
# cloudWatch:
#  clusterLogging:
#    enableTypes: ["*"]

secretsEncryption:
  keyARN: ${MASTER_ARN}
EOF
```

And issue the create command with:

```
eksctl create cluster -f eksworkshop-medium.yaml
```

You’ll incur in AWS costs to test this setup so make sure you are tearing down the environment after your tests by using this walk-through when you’re done testing:  
  
[https://www.eksworkshop.com/920_cleanup/](https://www.eksworkshop.com/920_cleanup/)  

Or just check the clean up instructions at the end of thi section.
  
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

You can use the instructions provided at the link as we haven't changed the EKS cluster name so all commands are already pointing to this cluster.
  
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
  labels:
    app: cassandra
spec:
  serviceName: cassandra
  replicas: 3
  selector:
    matchLabels:
      app: cassandra
  template:
    metadata:
      labels:
        app: cassandra
    spec:
      terminationGracePeriodSeconds: 1800
      containers:
      - name: cassandra
        image: gcr.io/google-samples/cassandra:v11
        imagePullPolicy: Always
        ports:
        - containerPort: 7000
          name: intra-node
        - containerPort: 7001
          name: tls-intra-node
        - containerPort: 7199
          name: jmx
        - containerPort: 9042
          name: cql
        resources:
          limits:
            cpu: "500m"
            memory: 1Gi
          requests:
            cpu: "500m"
            memory: 1Gi
        securityContext:
          capabilities:
            add:
              - IPC_LOCK
        lifecycle:
          preStop:
            exec:
              command: 
              - /bin/sh
              - -c
              - nodetool drain
        env:
          - name: MAX_HEAP_SIZE
            value: 512M
          - name: HEAP_NEWSIZE
            value: 100M
          - name: CASSANDRA_SEEDS
            value: "cassandra-0.cassandra.default.svc.cluster.local"
          - name: CASSANDRA_CLUSTER_NAME
            value: "K8Demo"
          - name: CASSANDRA_DC
            value: "DC1-K8Demo"
          - name: CASSANDRA_RACK
            value: "Rack1-K8Demo"
          - name: POD_IP
            valueFrom:
              fieldRef:
                fieldPath: status.podIP
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - /ready-probe.sh
          initialDelaySeconds: 15
          timeoutSeconds: 5
        volumeMounts:
        - name: cassandra-data
          mountPath: /cassandra_data
  volumeClaimTemplates:
  - metadata:
      name: cassandra-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: mysql-gp2
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
kubectl exec -it cassandra-0 -- cqlsh
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

```
kubectl delete pod cassandra-0

kubectl get pods

kubectl describe pods cassandra-0
```

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

Delete the Cassandra stateful set:

```
kubectl delete -f cassandra-app.yaml 
```

Delete the persistent volumes. Find them by using:

```
kubectl get pv
```


and proceed to delete each of them with

```
kubectl delete pv *pvid*
```

Check from EBS Console that the volume have been removed.

Delete the EKS cluster 

```

Check that the command completes successfullY: cluster is removed from EKS console and that EC2 instances are removed.

