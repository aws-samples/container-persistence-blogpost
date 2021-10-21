# Single Container with persistent storage running across multiple Container Hosts
This part of the tutorial will drive you through the creation of Kubernetes cluster (on Amazon EKS), configuration of backing storage for containers (Amazon EBS) and in the end you’ll be able to deploy a Cassandra cluster that can withstand the loss of a Container Host.

We will **simulate a the failure of one container host of the EKS cluster in a single AZ** to demonstrate how an external storage (in this case EBS) can allow the application to withstand a Container host failure.
You can see the flow in these diagrams (**focus on Availability Zone 1**):

Initial Configuration where each conatiner in ech AZ is running and storing data on an extrenal EBS volume:

![Alt text](/images/Cassandra6Deployed.png "Cassandra6Deployed")

Failure of a Container host in AZ1

![Alt text](/images/Cassandra6Failure.png "Cassandra6Failure")

Container is restarted on the remaining host in AZ1 and reconnects to EBS exyerba storage:

![Alt text](/images/Cassandra6Restored.png "Cassandra6Restored")


**All code is provided** <u>**AS IS**</u>**: it is not meant for production workloads but for test environments only.**

You may incur in costs for testing this setup so we recommend to take this into account and tear down the environments after performing your tests.


## Prerequisites
Create an EKS Cluster following the tutorial at https://www.eksworkshop.com/030_eksctl/prerequisites/

We will use a Cloud9 instance to issue all the commands as described in the EKS Workshop. Remember to install all prerequisites as per the above link

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

The Cluster is distributed over 3 Availability Zones with **two** workers in each AZ.

To demonstrate that the application can survive the loss of a node in an AZ, we willd eploy a Cassandra cluster and simulate the loss of a worker nodei in one availability zone. The cassandra cluster will have one node in every AZ connected to an external EBS storage, as such **during the simulated failure the pod will be restarted on the remaining node in the AZ and the application will be able to reconnect to storage** (as this is external to the container and available on EBS).






# Clean up Instructions

Remember to delete the AWS Cloud9 instance following these instructions (as per the eksworkshop clean up):
1. Go to your Cloud9 Environment
2. Select the environment named eksworkshop and pick delete






























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



