# sce
## secure computing environment builder walk-through

### go

* [reference page at cloudmaven](https://cloudmaven.github.io/documentation/aws_hipaa.html)
* designate HIPAA to both DLT and AWS 
* Assemble the following pieces (no EC2 just yet) in a Virtual Private Cloud (VPC)
* VPC **sce**
  * VPC characteristics
    * ID **vpc-74e0c60d** in Oregon (AZ = C on Network Interface; see below)
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
  * Flow Log?
  * S3 Bucket?
  * Roles? EMR_EC2_DefaultRole
  * Security Groups? 
    * added **sce** for **sce worker**
      * needs improvement 'open to the world'
  
Note: The private subnet with CIDR block 10.0.1.0/24 is home to the EC2 Worker; firewalled behind a NAT Gateway
that blocks traffic in such as *ssh*. The public subnet is for external access via the EC2 Bastion server, with
CIDR block 10.0.0.0/24. The public subnet connects to the internet via an Internet Gateway. It also hosts the 
NAT Gateway; so the NAT Gateway is not 'sequestered' on the private subnet. Both the Bastion and the NAT Gateway are 
on the public subnet but also have private subnet IP addresses. This is true of *any* resource on the public subnet: 
It always has a private ip address in the VPC as well. Public names resolve to pirvate addresses within the VPC as needed. 


### EC2 Worker and Bastion

* EC2 for Worker **sce worker** **i-09c35aea9ee6c15be**
  * Launched from AMI **moby-ami-test** = **ami-0cf27374** (JupyterHub pre-installed)
  * Instance Type **m5.large**
  * Configure Instance Details
    * On the above VPC, **sce Private** subnet-e4680fb3
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
  
### Tunnel work


    
  
  
    
  
