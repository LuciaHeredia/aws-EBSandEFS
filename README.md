# Introduction:
This is a guide for implementing the following:
1) Launch EC2 Instance, Create and Mount EBS Volume with UserData file.
2) Create and Mount EFS Volume with AWS EFS Dashboard and EFS mount helper.
 
<br/>

# Explanation:
## EBS (Elastic Block Store):
* An ***Amazon EBS Volume*** is a durable, block-level storage device that can be attached to ***Amazon EC2 instances***. <br/>
* Useful for storing data that requires frequent updates, such as system drives for instances or databases. <br/>
* They are specific to an ***availability zone*** and can be migrated between zones using ***snapshots***, but they cannot be attached to instances in different zones. <br/>
* It can be created during the process of creating an ***EC2 instance*** or as standalone volumes and attached to running ***EC2 instances***. <br/>

## EFS (Elastic File System):
* An ***AWS EFS volume*** is a scalable, simple, and managed file storage service, provided by ***Amazon Web Services (AWS)***. <br/>
* Useful for applications that require shared storage(from a ***file system***) across multiple ***Amazon Elastic Container Service (ECS) tasks*** or ***Amazon EC2 instances***. <br/>
* They are designed to be highly available and scalable, automatically adjusting ***storage capacity*** as you add or remove files. <br/>

<br/>

# Tasks + Steps:
## Mission 1: Launch EC2 Instance and Mount EBS Volume with UserData
### Step 1: Launch EC2 Instance
* Download **AWS CLI** and verify if it’s properly installed with the command:
  ```
  $ aws --version
  ```
* Create **AWS Access Key**:
  - Go to your AWS account overview.
  - Account menu in the upper-right (has your name on it), press "Security Credentials".
  - In section "Access keys", press "Create access key".
  - Save Access Key in a secure location.
* Configure and login the necessary IAM user with **AWS CLI**:
  ```
  $ aws configure
  ```
  Here we’ll need the access key and secret key for the IAM user, as well as the region id (can be found beside the region name in your AWS account):
  ```
  AWS Access Key ID [None]: accesskey
  AWS Secret Access Key [None]: secretkey
  Default region name [None]: us-west-2
  Default output format [None]:
  ```
  To update just the region name:
  ```
  $ aws configure
  AWS Access Key ID [****]:
  AWS Secret Access Key [****]:
  Default region name [us-west-1]: us-west-2
  Default output format [None]:
  ```
* Launch an **EC2 instance** using **AWS CLI** (specify the necessary details: **AMI ID, instance type, key pair, subnet, security group, and tags**):
  - Get **AMI ID**:
    - Go to "EC2 Dashboard".
    - Then in the left menu click on "Images", and then "AMI Catalog".
    - Here you'll find a list of base images from AWS along with the **AMI ID**.
  - Get **VPC ID** and **Subnet Id**:
    - Go to the "VPC Dashboard" and click on "VPCs", here you will get the "VPC ID".
    - Then in the left menu click on "Virtual private cloud", then "Subnets", and search with the "VPC ID" to list all the subnets associated with that **VPC** and copy the "Subnet ID".
  - Setting up **SSH key pair**, in order to authenticate our EC2 instance:
    ```
    $ aws ec2 create-key-pair \       
      --key-name  <your key name> \
      --query '<your query>' --output text > ~/.ssh/<key-file-name>
    ```
  - Setting up a **security group**, in order to control who has access to that instance:
    ```
    $ aws ec2 create-security-group \
      --group-name <group name> \
      --description "rules for this group" \
      --vpc-id <vpc-xxxx>
    $ aws ec2 authorize-security-group-ingress \
      --group-id <group-id> \
      --protocol tcp \
      --port 22 \
      --cidr 0.0.0.0/0
    ```
  - Finally, Launching the instance:
    ```
    $ aws ec2 run-instances \
      --image-id <ami-id> \
      --instance-type <instance-type> \
      --key-name <ec2-key-pair-name> \
      --subnet-id <subnet-id> \
      --security-group-ids <security-group-id> \
      --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=<instance-name>}]'
    ```
### Step 2: Create and Attach an EBS Volume
* Create an **EBS volume** in the same **availability zone** and attach it to the launched **EC2 instance**.
  - Create:
    ```
    $ aws ec2 create-volume --volume-type <type of storage volume> --availability-zone <availability-zone> --size <size-in-GiB>
    ```
  - Attach: ( *device-name* usually: /dev/sd* or /dev/xvd* )
    ```
    $ aws ec2 attach-volume --volume-id <id of created volume> --instance-id <instance-id where volume is to be attached> --device <device-name>
    ```
### Step 3: Mount EBS Volume Using UserData
* Connect to created EC2 instance with SSH Client:
  - Go to "EC2 Dashboard".
  - Press your "EC2 Instance", press "connect", and follow the instructions for "SSH client".
* Enter as root:
  ```
  $ sudo -i
  ```
* Create a **UserData script ( userdata.sh )** to be executed during instance launch:
  ```
  $ vim userdata.sh
  ```
* The script should format the **EBS volume**, create a mount point, and mount the volume. Also, mount EBS volume automatically on boot:
  - [userdata.sh](https://github.com/LuciaHeredia/aws-EBSandEFS/blob/main/userdata.sh) - make sure to change "EBS_DEVICE" and "MOUNT_POINT" endings.
* Ensure the **UserData script** is made executable:
  ```
  $ chmod +x userdata.sh
  ```
* Run the script:
  ```
  $ ./userdata.sh
  ```
* Check EBS volume(xvd*) has a set mount point:
  ```
  $ lsblk
  ```
* Exit and Check changes:
  - Exit root:
    ```
    $ exit
    ```
  - Exit EC2 instance:
    ```
    $ exit
    ```
  - Reboot EC2 instance:
    ```
    $ aws ec2 reboot-instances --instance-ids <instance-id>
    ```
  - Enter EC2 instance again with SSH Client as before.
  - Enter as root as before.
  - Check mount again:
    ```
    $ lsblk
    ```

## Mission 2: Create and Mount EFS Volume
### Step 1: Create an EFS Volume
* Creating Security Group for EFS:
  - Go to EC2 dashboard, in left menu press "Network & Security", then "Security Groups".
  - Set a "name" and "description".
  - Select "VPC". (EFS and its Security group must be in the same VPC).
  - In "inbound rules" section press "Add rule", and select the "Type" **NFS**, select "source" to **Anywhere**.
* Navigate to the **AWS Management Console** and access the **EFS service**.
* Create a new **EFS file system**, select "customize" and configure: name, throughput mode, performance mode, network access, including selecting VPC and defining security groups:
  - Set **EFS** "name".
  - Select "Throughput mode".
  - In "Additional settings" -> set "Performance mode".
  - In "Network access" select **VPC** and **security group** from before for all mount targets.
### Step 2: Mount EFS Volume on EC2 Instance
* Install the EFS mount helper on the EC2 instance using the package manager:
  - Enter EC2 instance again with SSH Client as before.
  - Install the EFS mount helper(NFS Client):
    ```
    $ sudo apt-get -y install nfs-common
    ```
* Create a mount point on the EC2 instance for the EFS volume:
  - Enter as root:
    ```
    $ sudo -i
    ```
  - Create mount point folder:
    ```
    $ mkdir <efs-mount-point>
    ```
* Mount the EFS volume to the specified mount point. Access the DNS name of the EFS file system from the console for mounting on EC2 instances:
  - In the **EFS dashboard**, go to the **EFS** created, press "attach".
  - Select "Mount via DNS" and use the NFS client, example:
    ```
    $ sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0000000000000.efs.<region>.amazonaws.com:/ <efs-mount-point>
    ```
  - Check mount:
    ```
    $ df -h
    ```

*Author*: [LuciaHeredia](https://github.com/LuciaHeredia)
