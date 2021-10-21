# Single Container with persistent storage running across multiple Container Hosts
This part of the tutorial will drive you through the creation of Kubernetes cluster (on Amazon EKS), configuration of backing storage for containers (Amazon EBS) and in the end you’ll be able to deploy a container with SQL server that can withstand the loss of a Container Host.

![Alt text](/images/SingleContainerMultipleContainerHosts.png "SingleContainerMultipleContainerHosts")

**All code is provided** <u>**AS IS**</u>**: it is not meant for production workloads but for test environments only.**

You may incur in costs for testing this setup so we recommend to take this into account and tear down the environments after performing your tests.


## Prerequisites
Create an EKS Cluster following the tutorial at https://www.eksworkshop.com/030_eksctl/prerequisites/

The ONLY difference is that at step 3 (https://www.eksworkshop.com/030_eksctl/launcheks/)  We will create a cluster that resides only in two Availability Zone (this is **NOT** a best practice and it is just for demo purposes):

```
cat << EOF > eksworkshop4nodes2azs.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl-4nodes2azs
  region: ${AWS_REGION}
  version: "1.21"

availabilityZones: ["${AZS[0]}", "${AZS[1]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 4
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
eksctl create cluster -f eksworkshop4nodes2azs.yaml
```

This will take a few minutes, after the command completed check if the cluster is available:
```
kubectl get nodes
```


Create storage class for this cluster by following the EKS tutorial for EBS CSI: https://www.eksworkshop.com/beginner/170_statefulset/ebs_csi_driver/ stop after completing the third step  https://www.eksworkshop.com/beginner/170_statefulset/storageclass/

Now you have a Kubernetes Cluster that can use EBS as an external storage provider.

We will now deploy a PostgreSQL container that uses this external storage.

As storage is provided by EBS the storage will be available in a single AZ.

Create a Persistent Volume Claim for our PostgreSQL container (this uses the storage class that we prepared above).

```
cat << EoF > pvcpostgresql.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvcpostgresql
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: mysql-gp2
  resources:
    requests:
      storage: 4Gi
```

Check that the persistent volume claim has been created:
```
kubectl get pvc
```

Create a password for our postgreSQL
```
echo awsawsaws123 > password.txt
tr --delete '\n' <password.txt >.strippedpassword.txt && mv .strippedpassword.txt password.txt
kubectl create secret generic postgresql-pass --from-file=password.txt
```

```
kubectl get secrets
```

Create the PostgreSQL deployment that will consume this storage:

```
cat << EoF > apppostgresql.yam
kind: Service
metadata:
  name: postgres
spec:
  ports:
    - port: 5432
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  labels:
    name: postgres
spec:
  strategy:
    rollingUpdate:
       maxSurge: 1
       maxUnavailable: 1
    type: RollingUpdate
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:9.6
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgresql-pass
                  key: password.txt
          ports:
            - containerPort: 5432
              name: postgres
          volumeMounts:
            - mountPath: /var/lib/postgresql
              name: pg-storage
      volumes:
        - name: pg-storage
          persistentVolumeClaim:
            claimName: pvcpostgresql
EoF
```

Check that it is running:
```
kubectl get pods
```
Connect to the container:
```
kubectl exec -it Containerid bash
```



