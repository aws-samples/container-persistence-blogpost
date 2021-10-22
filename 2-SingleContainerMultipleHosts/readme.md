# Single Container with persistent storage running across multiple Container Hosts
This part of the tutorial will drive you through the creation of Kubernetes cluster (on Amazon EKS), configuration of backing storage for containers (Amazon EBS) and in the end you’ll be able to deploy a Cassandra cluster that can withstand the loss of a Container Host.

We will **simulate a the failure of one container host of the EKS cluster in a single AZ** to demonstrate how an external storage (in this case EBS) can allow the application to withstand a Container host failure.
You can see the flow in these diagrams (**focus on Availability Zone 1**):

Initial Configuration where each conatiner in ech AZ is running and storing data on an extrenal EBS volume:

![Alt text](/images/Cassandra6Deployed.png "Cassandra6Deployed")

Failure of a Container host in AZ1

![Alt text](/images/Cassandra6Failure.png "Cassandra6Failure")

Container is restarted on the remaining host in AZ1 and reconnects to EBS external storage:

![Alt text](/images/Cassandra6Restored.png "Cassandra6Restored")


**All code is provided** <u>**AS IS**</u>**: it is not meant for production workloads but for test environments only.**

You may incur in costs for testing this setup so we recommend to take this into account and tear down the environments after performing your tests.


## Prerequisites

To create an EKS Cluster you can follow these steps: [https://www.eksworkshop.com/020_prerequisites/self_paced/account/](https://www.eksworkshop.com/020_prerequisites/self_paced/account/). This will help You in setting up an AWS Cloud9 instance to use to issue the commands in this tutorial to your own EKS cluster. Use the link but when it comes to the creation of the ekscluster please refer to the instructions provided here.

We will use a Cloud9 instance to issue all the commands as described in the EKS Workshop. Remember to install all prerequisites as per the above link.

The ONLY difference is that at step 3 (https://www.eksworkshop.com/030_eksctl/launcheks/)  We will **create a cluster that is using 6 worker nodes**:

```
cat << EOF > eksworkshop6nodes.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl-6nodes
  region: ${AWS_REGION}
  version: "1.21"

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 6
  instanceType: t3.small
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

Then launch it:
```
eksctl create cluster -f eksworkshop6nodes.yaml
```

This will take a few minutes, after the command completed check if the cluster is available:

```
kubectl get nodes
```


Create **storage class for this cluster** by following the EKS tutorial for EBS CSI: https://www.eksworkshop.com/beginner/170_statefulset/ebs_csi_driver/ stop after completing the third step  https://www.eksworkshop.com/beginner/170_statefulset/storageclass/

To perform this part of the tutorial You can follow the steps below that have been customized with the cluster name we used above:

IAM policy (no changes):

```
export EBS_CSI_POLICY_NAME="Amazon_EBS_CSI_Driver"

mkdir ${HOME}/environment/ebs_statefulset
cd ${HOME}/environment/ebs_statefulset

# download the IAM policy document
curl -sSL -o ebs-csi-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json

# Create the IAM policy
aws iam create-policy \
  --region ${AWS_REGION} \
  --policy-name ${EBS_CSI_POLICY_NAME} \
  --policy-document file://${HOME}/environment/ebs_statefulset/ebs-csi-policy.json

# export the policy ARN as a variable
export EBS_CSI_POLICY_ARN=$(aws --region ${AWS_REGION} iam list-policies --query 'Policies[?PolicyName==`'$EBS_CSI_POLICY_NAME'`].Arn' --output text)
```

IAM OIDC provider for the cluster (changed cluster name with teh one used in this tutorial):

```
# Create an IAM OIDC provider for your cluster
eksctl utils associate-iam-oidc-provider \
  --region=$AWS_REGION \
  --cluster=eksworkshop-eksctl-6nodes \
  --approve

# Create a service account
eksctl create iamserviceaccount \
  --cluster eksworkshop-eksctl-6nodes \
  --name ebs-csi-controller-irsa \
  --namespace kube-system \
  --attach-policy-arn $EBS_CSI_POLICY_ARN \
  --override-existing-serviceaccounts \
  --approve
```

Configure a repository for the CSI driver:

```
# add the aws-ebs-csi-driver as a helm repo
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

# search for the driver
helm search  repo aws-ebs-csi-driver
```

And install it using Helm (remember to install helm in your Cloud 9 instance as per the instructions on the EKS Workshop linked above):

```
helm upgrade --install aws-ebs-csi-driver \
  --version=1.2.4 \
  --namespace kube-system \
  --set serviceAccount.controller.create=false \
  --set serviceAccount.snapshot.create=false \
  --set enableVolumeScheduling=true \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true \
  --set serviceAccount.snapshot.name=ebs-csi-controller-irsa \
  --set serviceAccount.controller.name=ebs-csi-controller-irsa \
  aws-ebs-csi-driver/aws-ebs-csi-driver

kubectl -n kube-system rollout status deployment ebs-csi-controller
```

Now let's **define a Storage Class**:

```
cat << EoF > ${HOME}/mysql-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: mysql-gp2
provisioner: ebs.csi.aws.com # Amazon EBS CSI driver
parameters:
  type: gp2
  encrypted: 'true' # EBS volumes will always be encrypted by default
volumeBindingMode: WaitForFirstConsumer # EBS volumes are AZ specific
reclaimPolicy: Delete
mountOptions:
- debug
EoF
```

And create it:

```
kubectl create -f ${HOME}/mysql-storageclass.yaml
```

Check that it is available:

```
kubectl describe storageclass mysql-gp2
```


Now you have a Kubernetes Cluster that can use EBS as an external storage provider.

The Cluster is distributed over 3 Availability Zones with **two** workers in each AZ.

To demonstrate that the application can survive the loss of a node in an AZ, we will deploy a Cassandra cluster and simulate the loss of a worker node in one availability zone. The Cassandra Cluster will have one node in every AZ connected to an external EBS storage, as such **during the simulated failure the pod will be restarted on the remaining node in the AZ and the application will be able to reconnect to storage** (as this is external to the container and available on EBS).

![Alt text](/images/Cassandra6Deployed.png "Cassandra6Deployed")

Deploy the Cassandra Cluster

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


## Failover Test
Now we will simulate a failure scenario where one container host node in one AZ becomes unavailable (as we have two nodes per AZ so we expect that the coonatiner to start on the operational container host in the same AZ and reconnect to storage).  
To see that data gets persisted even in case of a failure **let’s write some information into the Cassandra cluster before simulating the failure**.  

Connect to node zero of the Cassandra cluster:  
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
You will see the list of the Cassandra nodes:

![Alt text](/images/3-Cassandraips.png "3-Cassandraips")

Now we will simulate a failure for one AZ (by taking down the related Kubernetes container host node) and see how the setup behaves.  
Let’s cordon ([https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)) the node where pod cassandra-0 is executing:  
```
NODE=`kubectl get pods cassandra-0 -o json | jq -r .spec.nodeName`

kubectl cordon ${NODE}

kubectl get nodes
```
In the output you’ll see that the container scheduling is now disabled for that node:

![Alt text](/images/cordon6nodes.png "cordon6nodes")

Now let’s delete the pod (to simulate failure in the AZ) and see if it can restart:

```
kubectl delete pod cassandra-0

kubectl get pods

kubectl describe pods cassandra-0
```

The pod will restart and the last command will produce an output similar to:

![Alt text](/images/6-podscheduling.png "6-podscheduling")

As you can see from the latest output the pod cannot be scheduled as there are no Kubernetes nodes that can connect to the data (as for demo purposes we just had **only one node per AZ in our setup** and the EBS volume for the container exists only in one availability zone):

![Alt text](/images/7-onenodeperaz.png "7-onenodeperaz")

Let's see that the Cassandra cluster is still operational even if one container host is offline.

We can issue again commands to node 0 as it has been restarted on the container hiost available in AZ 1:

```
kubectl exec cassandra-0 -- nodetool status
```
Issuing the same command to node 1 will show us the Cassandra cluster status:

```
kubectl exec cassandra-0 -- nodetool status
```
We can see that all Cassandra nodes are active

And the data is still available

```
kubectl exec cassandra-0 -- cqlsh -e 'SELECT * FROM awsdemo.awsregions;'
```
And it is still set to be protected over the three nodes:
```
kubectl exec -it cassandra-1 -- nodetool getendpoints awsdemo awsregions eu-south-1
```

![Alt text](/images/cassandradatastillpresent.png "cassandradatastillpresent")


The **Cassandra pod in AZ 1 is back online** even after one Container host in zone 1 has failed and **data has been preserved**.  
 
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


And proceed to delete each of them with

```
kubectl delete pv *pvid*
```

Check from EBS Console that the volume have been removed.

Delete the EKS cluster 

```

Check that the command completes successfullY: cluster is removed from EKS console and that EC2 instances are removed.


Remember to delete also Your AWS Cloud9 instance following these instructions (as per the eksworkshop clean up):
1. Go to your Cloud9 Environment
2. Select the environment named eksworkshop and pick delete

