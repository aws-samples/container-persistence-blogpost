# Single container with single persistent datastore that spans across Availability Zones

You can test the setup of a single container connected to a datastore that spans multiple AZs **by using an Amazon EKS cluster coupled with Amazon EFS**. This example architecture can be built by using the steps detailed in our EKS Workshop: https://www.eksworkshop.com/beginner/190_efs/ 

![Alt text](/images/4-EFSMultiple.png "4-EFSMultiple")


**All code is provided** <u>**AS IS**</u>**: it is not meant for production workloads but for test environments only.**

You may incur in costs for testing this setup so we recommend to take this into account and tear down the environments after performing your tests.

![Alt text](/images/4-SingleEFS.png "4-SingleEFS")

In this use case *one container writes and one reads from Amazon EFS* but in terms of architecture (i.e. containers using a multi AZ datastore) the setup is very similar to the one that we have detailed in the blog paragraph for this setup.

# Clean up instructions

Follow the clean up instructions provided at: https://www.eksworkshop.com/beginner/190_efs/cleaning/
