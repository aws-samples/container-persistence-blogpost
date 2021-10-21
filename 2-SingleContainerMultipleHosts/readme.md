# Single Container with persistent storage running across multiple Container Hosts
This part of the tutorial will drive you through the creation of Kubernetes cluster (on Amazon EKS), configuration of backing storage for containers (Amazon EBS) and in the end you’ll be able to deploy a container with SQL server that can withstand the loss of a Container Host.

![Alt text](/images/SingleContainerMultipleContainerHosts.png "SingleContainerMultipleContainerHosts")

**All code is provided** <u>**AS IS**</u>**: it is not meant for production workloads but for test environments only.**

You may incur in costs for testing this setup so we recommend to take this into account and tear down the environments after performing your tests.


## Prerequisites
Create an EKS Cluster following the tutorial at https://www.eksworkshop.com/030_eksctl/prerequisites/

The ONLY difference is that at step 3 (https://www.eksworkshop.com/030_eksctl/launcheks/)  We will create a cluster that is using 3 worker nodes:

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


Create storage class for this cluster by following the EKS tutorial for EBS CSI: https://www.eksworkshop.com/beginner/170_statefulset/ebs_csi_driver/ stop after completing the third step  https://www.eksworkshop.com/beginner/170_statefulset/storageclass/

Here are the steps customized with the cluster name we used above:

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

IAM OIDC provider for the cluster (changed cluster name):


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

Configure a repository for the CSI driver

```
# add the aws-ebs-csi-driver as a helm repo
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver

# search for the driver
helm search  repo aws-ebs-csi-driver
```

And install it:

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

Now let's define a Storage Class:

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

And create it

```
kubectl create -f ${HOME}/mysql-storageclass.yaml
```

Check that it is available:

```
kubectl describe storageclass mysql-gp2
```




Now you have a Kubernetes Cluster that can use EBS as an external storage provider.

The Cluster is distributed over 3 Availability Zones with two workers in each AZ.

To demonstarte that the application can survuve the loss of a node in an AZ (i.e




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
EoF
```
And create it

```
kubectl create -f pvcpostgresql.yaml
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
cat > app-postgresql.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      schedulerName: stork
      containers:
      - name: postgresql
        image: postgres:9.5
        imagePullPolicy: "Always"
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          value: pgbench
        - name: PGUSER
          value: pgbench
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-pass
              key: password.txt
        - name: PGBENCH_PASSWORD
          value: superpostgresql
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgresqldb
      volumes:
      - name: postgresqldb
        persistentVolumeClaim:
          claimName: pvcpostgreqsl
EOF
```

Deploy the pod:

```
kubectl create -f app-postgresql.yaml
```





























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



