### Deploy Cloud9 IDE:
This lab documentation is made for N.Virginia region (us-east-1). Please make note of this, and change accordingly for your deployment.
Login to the AWS EC2 Console, go to Cloud9 Services. <br/>
Create a **new environment** e.g. "Cloud9 Lab - Containerized Nodejs application" <br/>
>#Select Environment type : "Create a new instance for environment (EC2)<br/>
>#Instance type : t2.micro (1 GiB RAM + 1 vCPU)  <br/>
>#Platform : Amazon Linux <br/>
>#Cost-saving setting: after 30 minutes <br/>
>#Network settings : Select an existing vpc and subnet, or create a new one . Select a Public Subnet, connected to Internet Gateway for Preview-URL Testing <br/>
>#Review, click Create <br/>

* **IAM AdministratorAccess Role for Cloud9 Instance :**
>#Go to IAM Service, create a role <br/>
>#Type of trusted entity : **AWS Service** <br/>
>#Choose the service that will use this role : **EC2**, 'Click Next:permissions' <br/>
>#Select from existing policies: **AdministratorAccess**, 'Next:Tags'  <br/>
>#Add-tags (optional) <br/>
>#Role Name: **Admin-Role_for_Cloud9_Instance** <br/>
>#Open EC2 Service console, select the Cloud9 Instance <br/>
>#Actions => Instance Settings => Attach/Replace IAM Role => Select **Admin-Role_for_Cloud9_Instance** => Apply<br/>

>#**In Cloud9 console => Preferences(Gear Icon on Upper Right Corner) => AWS Settings => Disable "AWS Managed Temporary Credentials. Slide the Grey button to cover Green area. Green area should not be visible** :relaxed:  <br/>


* **Capture Cluster Unique Name. This Name will be used to create a ssh key pair as well:**
```
read -p "Enter a unique HPC cluster Name : " HPC_CLUSTER_NAME ; 
echo -e "\n * * \e[106m ...HPC Cluster Name to be used is... : "$HPC_CLUSTER_NAME"\e[0m \n"

```
Saving the Cluster Name to bash-profile. In case the Cloud9 Instance Reboots, these variables will be saved and exported automatically
```
echo "export HPC_CLUSTER_NAME=${HPC_CLUSTER_NAME}" >> ~/.bash_profile
```

Configure AWS CLI
```
# run aws configure command. Do not enter Access key/Secret and output format. Just Enter the region name. 
aws configure


# Test aws CLI
aws ec2 describe-instances
```

Create a Bucket to store config and temp files. 

```
BUCKET_POSTFIX=$(uuidgen --random | cut -d'-' -f1)
aws s3 mb s3://bucket-${BUCKET_POSTFIX}

cat << EOF
***** Take Note of Your Bucket Name *****
Bucket Name = bucket-${BUCKET_POSTFIX}
*****************************************
EOF


echo "export BUCKET_POSTFIX=${BUCKET_POSTFIX}" >> ~/.bash_profile 
export BUCKET_POSTFIX=${BUCKET_POSTFIX}
echo "export Temp_S3_BUCKET=bucket-${BUCKET_POSTFIX}" >> ~/.bash_profile ; tail ~/.bash_profile

```

Download a file from the internet to your Cloud9 instance. For example, download synthetic subsurface model https://wiki.seg.org/wiki/SEG_C3_45_shot. 

```
wget http://s3.amazonaws.com/open.source.geoscience/open_data/seg_eage_salt/SEG_C3NA_Velocity.sgy

#Copy the file to S3 bucket
aws s3 cp ./SEG_C3NA_Velocity.sgy s3://bucket-${BUCKET_POSTFIX}/SEG_C3NA_Velocity.sgy

#Verify the File-copy operaton
aws s3 ls s3://bucket-${BUCKET_POSTFIX}/

#Remove file from local achine
rm SEG_C3NA_Velocity.sgy
```

Create a Key for SSH to Master Node
```
aws ec2 create-key-pair --key-name lab-ssh-key-$HPC_CLUSTER_NAME --query KeyMaterial --output text > ~/.ssh/lab-ssh-key-$HPC_CLUSTER_NAME
chmod 600 ~/.ssh/lab-ssh-key-$HPC_CLUSTER_NAME

export master_node_ssh_key=lab-ssh-key-${HPC_CLUSTER_NAME}
echo "export master_node_ssh_key=lab-ssh-key-${HPC_CLUSTER_NAME}" >> ~/.bash_profile ; tail ~/.bash_profile

# To Delete the Jey
##aws ec2 delete-key-pair --key-name lab-ssh-key-$HPC_CLUSTER_NAME

#Check Key creation
aws ec2 describe-key-pairs


#Copy the File from S3 Bucket
aws s3 cp s3://bucket-${BUCKET_POSTFIX}/SEG_C3NA_Velocity.sgy .

```

###Setting Up and Configuring Parallel Cluster

```
# optionally, you might need to install pip first
# sudo easy_install pip

pip-3.6 install aws-parallelcluster -U --user

```

# Create Config File for Parallel Cluster
```
IFACE=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/)
SUBNET_ID=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/${IFACE}/subnet-id)
VPC_ID=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/${IFACE}/vpc-id)

mkdir -p ~/.parallelcluster
cat > ~/.parallelcluster/config << EOF
[aws]
aws_region_name = us-east-1

[cluster default]
key_name = master-node-ssh-key
vpc_settings = public

[vpc public]
vpc_id = ${VPC_ID}
master_subnet_id = ${SUBNET_ID}

[global]
cluster_template = default
update_check = false
sanity_check = true

[aliases]
ssh = ssh {CFN_USER}@{MASTER_IP} {ARGS}
EOF


# Copying to environment folder for reference
cp ~/.parallelcluster/config ~/environment/parallelcluster-config-$HPC_CLUSTER_NAME

```

Check the Version of Parallel Cluster
```
pcluster version

which pcluster

export PATH=`which pcluster`:$PATH  ; echo $PATH
echo "export PATH=${PATH}" >> ~/.bash_profile ; tail ~/.bash_profile

```
# Creating Cluster Config

```
IFACE=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/)
SUBNET_ID=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/${IFACE}/subnet-id)
VPC_ID=$(curl --silent http://169.254.169.254/latest/meta-data/network/interfaces/macs/${IFACE}/vpc-id)
AZ=$(curl http://169.254.169.254/latest/meta-data/placement/availability-zone)
REGION=${AZ::-1}

cd ~/environment
cat > my-cluster-config-${HPC_CLUSTER_NAME}.conf << EOF
[aws]
aws_region_name = ${REGION}

[global]
cluster_template = default
update_check = false
sanity_check = true

[cluster default]
key_name = ${master_node_ssh_key}
vpc_settings = public
ebs_settings = myebs
compute_instance_type = c5.xlarge
master_instance_type = c5.xlarge
cluster_type = ondemand
placement_group = DYNAMIC
placement = compute
initial_queue_size = 0
max_queue_size = 8
disable_hyperthreading = true
s3_read_write_resource = *
scheduler = slurm
#For c5n.18x+ : enable_efa = compute

[vpc public]
vpc_id = ${VPC_ID}
master_subnet_id = ${SUBNET_ID}

[ebs myebs]
shared_dir = /shared
volume_type = gp2
volume_size = 50

[aliases]
ssh = ssh {CFN_USER}@{MASTER_IP} {ARGS}
EOF
```

# Build the ParallelCluster
```
pcluster create hpclab-${HPC_CLUSTER_NAME} -c my-cluster-config-${HPC_CLUSTER_NAME}.conf

```

# Login to Cluster and run basic Hello World job
```
#list existing clusters
pcluster list --color


pcluster ssh hpclab-${HPC_CLUSTER_NAME} -i ~/.ssh/lab-ssh-key-$HPC_CLUSTER_NAME

##List existing partitions and nodes per partition
sinfo

##List jobs in the queues or running
squeue

##List available modules
module av

##Load a particular module. In this case, this command loads IntelMPI in your environment and checks the path of mpirun
module load intelmpi
which mpirun


##List mounted volumes
showmount -e localhost
```


##Create the Hello World Application
```
cat > mpi_hello_world.c << EOF
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>
#include <unistd.h>

int main(int argc, char **argv){
  int step, node, hostlen;
  char hostname[256];
  hostlen = 255;

  MPI_Init(&argc,&argv);
  MPI_Comm_rank(MPI_COMM_WORLD, &node);
  MPI_Get_processor_name(hostname, &hostlen);

  for (step = 1; step < 5; step++) {
    printf("Hello World from Step %d on Node %d, (%s)\n", step, node, hostname);
    sleep(2);
  }

 MPI_Finalize();
}
EOF

module load intelmpi
mpicc mpi_hello_world.c -o mpi_hello_world


```

#test your application locally on the head node
```
mpirun -n 4 ./mpi_hello_world
```

#Create a Submission Script
```
cat > submission_script.sbatch << EOF
#!/bin/bash
#SBATCH --job-name=hello-world-job
#SBATCH --ntasks=4
#SBATCH --output=%x_%j.out

mpirun ./mpi_hello_world
EOF
```

#Submit your First Job
```
sbatch submission_script.sbatch

squeue


sinfo


```


