# Single Container running with persistent storage

**All code is provided** <u>**AS IS**</u>**: it is not meant for production workloads but for test environments only.**

You may incur in costs for testing this setup so we recommend to take this into account and tear down the environments after performing your tests.

**Overview**

This tutorial will drive you through the **creation of Docker container that leverages backing storage for containers provided by Docker Volume**.

![Alt text](/images/SingleContainerDockerVolume.png "SingleContainerDockerVolume")


This setup uses **AWS Cloud9 as an environment to deploy your container and storage used by Docker volume is local to that Cloud9 instance**.


If you want to test how to persist data for a single container running on a single Container host out you could use AWS Cloud9 ([https://aws.amazon.com/cloud9/](https://aws.amazon.com/cloud9/)). We recommend using an AWS Cloud9 desktop since this provides an environment to run any commands without needing to modify your own systems, and it provides a consistent experience that has been tested by the author. 

To deploy Cloud9 you can use this guide provide as part of the AWS EKS worshop 
[https://www.eksworkshop.com/020_prerequisites/workspace/]([https://www.eksworkshop.com/020_prerequisites/workspace/](https://archive.eksworkshop.com/020_prerequisites/workspace/))  
  
Here are the quick steps for **testing on AWS Cloud9 and Docker**.  
  
Install docker-compose ([https://docs.docker.com/compose/](https://docs.docker.com/compose/))  on your Cloud9 Instance:

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Now let’s deploy a MySQL container that uses a **Docker volume to store persistent MySQL data**:  
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


Now let’s see that the **volume is using Docker volumes**:
```
docker inspect *containerid*
```
![Alt text](/images/1b-inspect.png "1b-inspect")

We can see that the volume is being correctly configured by inspecting the Mounts configuration:

![Alt text](/images/2b-mounts.png "2b-mounts")

Let's see how this work: **we will write a file inside the container and see that it gets written to the container host**.

Connect to the container:
```
docker exec -it *containerid* bash
```

Create a file in the mount and exit:
```
touch /var/lib/mysql/testmount
exit
```

Now that **we are back in the container host let's check that the file is actually written locally**:
```
ls /var/lib/docker/volumes/ec2-user_dbdata/_data
```

![Alt text](/images/DockerVolumeMount.png "DockerVolumeMount")


We have effectively **created a MySQL container that effectively uses a volume external from the container**.

## Clean up Instructions

Please remember to **clean up the deployment** by using:
```
docker stop containerid
docker-compose rm db
```

![Alt text](/images/DockerVolumeCleanup.png "DockerVolumeCleanup")


And deleting the AWS Cloud9 instances following these instructions (as per the eksworkshop clean up):
1. Go to your Cloud9 Environment
2. Select the environment named eksworkshop and pick delete
