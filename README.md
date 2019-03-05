# sce
## secure computing environment builder walk-through

### Surgical Site Infection (SSI) Project

* [This repo](https://github.com/robfatland/sce)
* [UW IT Wiki page on ip address assignment](https://wiki.cac.washington.edu/pages/viewpage.action?pageId=53181947)
  * 10.5.0.0/17 and 10.5.128.0/17 for Azure and AWS respectively
* R and R-Studio
* Scale of TB
* [Older reference material](https://cloudmaven.github.io/documentation/aws_hipaa.html)


### Starting procedure


This sequence of steps gets two Virtual Machines (VMs) running on AWS. The first is a publicly visible guardian
machine called a bastion server. The second resides inside a sub-space called a Virtual Private Cloud (VPC) on
a private sub-network. It is called the worker. 


> Some details of what follows are not fully resolved and are marked with <flag>

> Every resource described here is tagged with a name that includes `sce` for *Secure Computing Environment*

* Start assumption: You have opened / are opening an AWS account through DLT
* Inform DLT and AWS that PHI/HIPAA-compliant work is in progress on this account 
* Assemble the following components in a Virtual Private Cloud (VPC)
  * VPC **sce** with characteristics
    * ID **vpc-74e0c60d** in US-West-2 (Oregon)
      * ...with Availability Zone (AZ) = C on the Network Interface; see below
    * IPv4 CIDR 10.0.0.0/16 has 2**(32-16) addresses available (65k)
    * Network ACL acl-ad60c9d5
    * Route table rtb-884a6df0 named **sce Main** 
    * Tenancy: Default; DNS resolution Enabled; DNS hostnames Enabled; Owner: Czar account; No flow logs
  * Subnets
    * sce Public **subnet-826a0dd8** IPv4 10.0.0.0/24 (256) Route table rtb-ae4b6cd6 **sce Second**
    * sce Private **subnet-e4680fb3** IPv4 10.0.1.0/24 (256) Route table rtb-884a6df0 **sce Main**
  * Route tables
    * **sce Main** rtb-884a6df0 
      * Destinations 
        * 10.0.0.0/16 Target local
        * pl-68a54001 (com.amazonaws.us-west-2.s3, 54.231.160.0/19, 52.218.128.0/17, 52.92.32.0/22) 
          * Target vpce-e923e880
        * 0.0.0.0/0 Target NAT Gateway **nat-02b636be6988c3f83** 
    * **sce Second** rtb-0ae4b6cd6
      * Destinations
        * 10.0.0.0/16 Target local
        * pl-68a54001 (com.amazonaws.us-west-2.s3, 54.231.160.0/19, 52.218.128.0/17, 52.92.32.0/22)	
          * Target vpce-e923e880
        * 0.0.0.0/0 Target Internet Gateway **igw-eb22528d**
  * End points
    * S3
      * **vpce-e923e880**  Service name **com.amazonaws.us-west-2.s3** Type **Gateway** 
      * <flag> what is this S3 bucket named, is it encrypted, how is it accessed?
  * Internet Gateway 
    * **sce** **igw-eb22528d** 
  * NAT Gateway
    * **sce** **nat-02b636be698** 
      * Elastic IP address 52.39.211.96 Private IP 10.0.0.104  
      * Network Interface eni-476df85e
      * Subnet **subnet-826a0dd8** (sce Public)
      * CloudWatch enabled; see Monitoring tab
  * Elastic IPs
    * **sce** 34.218.177.20 **eipalloc-2a088f16** 
      * Private IP address: None given Scope: vpc (seems to be un-used)
    * **--no name--** 52.39.211.96 **eipalloc-5386a237** 
      * Private IP address: 10.0.0.104 Scope: vpc eipassoc-91738e4b 
      * Network Interface ID: **eni-476df85e**
  * Network Interface
    * **sce** **eni-476df85e** Subnet ID **subnet-826a0dd8** Zone us-west-2c
    * Description: **Interface for NAT Gateway Interface for NAT Gateway nat-02b636be6988c3f83**
    * IPv4 Public IP: 52.39.211.96
    * Primary private IPv4 IP 10.0.0.104
  * <flag> Flow Log?
  * <flag> CloudWatch: Gets us what?
  * <flag> CloudTrail: Looks like what? Granularity?
  * <flag> Roles? EMR_EC2_DefaultRole is in use but what does it do? Is it useful?
  * <flag> Security Groups? 
    * added **sce** for **sce worker**
      * needs improvement from 'open to the world'
  * <flag> AWS Config with 52 pre-designed rules that we can add; not to mention customized
  

> Note: The private subnet with CIDR block 10.0.1.0/24 is home to the EC2 Worker; firewalled behind a NAT Gateway
that blocks traffic in such as *ssh*. The public subnet is for external access via the EC2 Bastion server, with
CIDR block 10.0.0.0/24. The public subnet connects to the internet via an Internet Gateway. It also hosts the 
NAT Gateway; so the NAT Gateway is not 'sequestered' on the private subnet. Both the Bastion and the NAT Gateway are 
on the public subnet but also have private subnet IP addresses. This is true of *any* resource on the public subnet: 
It always has a private ip address in the VPC as well. Public names resolve to private addresses within the VPC as needed. 


> Now that the framework is in place the next step is to start up the two EC2 instances, `sce bastion` and `sce worker`.


### EC2 Worker and Bastion

* EC2 for Worker **sce worker** **i-09c35aea9ee6c15be**
  * Launched from AMI **moby-ami-test** = **ami-0cf27374** (JupyterHub pre-installed)
  * Instance Type **m5.large**
  * Configure Instance Details
    * On the above VPC, **sce Private** subnet-e4680fbe
    * Auto-assign Public IP: **Use subnet setting (Disable)**
    * Capacity Reservation: **Open**
    * IAM Role **EMR_EC2_DefaultRole**
    * Shutdown Behavior **Stop**
    * Monitoring **Checked** Enable CloudWatch detailed monitoring
    * Tenancy: **Shared**
    * Elastic Inference: **Not Checked**
  * Storage
    * 100GB Root /dev/sda1 snap-0cfb2ede9e021b4a1 (reduced from 1TB) 
      * GP SSD (gp2) 300 / 3000 IOPS, Delete on Termination
      * Not Encrypted
    * 1000GB EBS /dev/sdb snap-0bcc82862c83fb7d1
      * GP SSD (gp2) 300 / 3000 IOPS, Delete on Termination
      * Not Encrypted
    * 32GB Data EBS /dev/sdc 
      * GP SSD (gp2) 100 / 3000 IOPS; no Delete on Termination
      * Encrypted with KMS Key Alias = Default aws/ebs key alias
  * Security group
    * added **sce** sg-042047ec3d28c6ff1 
    * Type SSH, Protocol TCP, Port Range 22, Source 0.0.0.0/0 (open to world)
  * launch: downloaded new key pair **sceworker.pem** 
* EC2 for Bastion **sce bastion** **i-0291dffa465de737e**
  * Amazon Linux 2 AMI (HVM), SSD Volume Type - ami-032509850cf9ee54e (64-bit x86) / ami-00ced3122871a4921 (64-bit Arm)
  * **t2.medium**
  * Details
    * VPC **sce** **sce Public** 
    * IAM Role = **None**
    * CloudWatch **Enabled**
    * Tenancy = **Shared**
  * Storage
    * Root /dev/xvda snap-04359c6bb66cf4243 8GB gp2 IOPS 100 / 3000 Not Encrypted
  * Security group **sce** sg-042047ec3d28c6ff1
    * Inbound rules SSH TCP PortRange 22 Source 0.0.0.0/0
  * launch: downloaded new key pair **scebastion.pem**
  
### Lectroid demo config notes updated March 5 2019

* Restarted SCE Bastion and Worker from the AWS console: As a *different* admin User <flag>
  * Bastion is a t2.medium ($0.05/hr) at an elastic ip: Running Linux; with a VPC-internal private ip also
    * `ssh -i somepem.pem username@a.b.c.d`
      * The `sceworker.pem` file is here per the `sftp` sequence; but it required `chmod 400`
* Restarted SCE Worker from the console as well
  * This machine had a very expensive 1TB EBS root drive so I terminated it and began over
  * It is an m5a.large ($0.09/hr)) EC2; where from the console I see its private subnet ip address
    * ssh from local to bastion to worker is ok
    * worker login msg suggests `sudo apt-get update` plus `sudo shutdown -r now` 
    * Logged back in
      * should validate the drive space; and this is not an encrypted drive <flag> need to re-do 
      * need a lightweight version of Anaconda called Miniconda; plus make sure `jupyter` runs

```
wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

As this runs: Respond to the prompts with care; and continue with:

```
rm Miniconda3-latest-Linux-x86_64.sh
source ~/.bashrc
conda install ipython
conda install jupyter
conda create -n lectroid python=3.6
conda activate lectroid
(jupyter notebook --no-browser --port=8889) &
```

* The `create/activate` commands get us in the habit of treating a Python environment as a sub-framework of 
the generic general environment. <flag> What is still needed is steps to ensure this environment appears in
the notebook server interface. 


### Tunneling

* My **sce bastion** had no public ip address; so I assigned it an elastic ip **34.218.177.20**
  * Private: 10.0.0.35 on Public subnet
* Now from a bash shell command line

```
 $ ssh -i scebastion.pem ec2-user@34.218.177.20
 ```
 
 * Moving the ```sceworker.pem``` file to the bastion server
 
 ```
 $ sftp -i scebastion.pem ec2-user@34.218.177.20
 sftp> put sceworker.pem
 ```
 
  * Confirm by reconnection to sce bastion: Yes
  * Upgrade instances? Delete spare 1TB drive on Worker? Make a new AMI?
  * ssh ubuntu@54.70.247.255 -i scebastion.pem
    * from bastion:
      * `ssh -i sceprivate.pem ubuntu@10.0.1.234`
      * notice private subnet 10.0.1

      * `(jupyter notebook --no-browser --port=8889) &`
      * copy down very long token e.g.
        * `4109891ab3e0ec38c2aec9c427c8be11eda975ab2882a52a`
      * exit back to bastion
      * on bastion
        * `ssh -N -f -i sceprivate.pem -L localhost:7005:localhost:8889 ubuntu@10.0.1.234`
        * that's the pipe
        * exit to local
        * `ssh -N -f -i scebastion.pem -L localhost:7004:localhost:7005 ubuntu@54.70.247.255`
        * open browser, localhost:7004, enter token if requested

 
  

    
  
  
    
  
