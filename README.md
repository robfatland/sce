# sce
## secure computing environment builder walk-through

### go

* [reference page at cloudmaven](https://cloudmaven.github.io/documentation/aws_hipaa.html)
* designate HIPAA to both DLT and AWS 
* VPC **sce**
  * VPC characteristics
    * ID **vpc-74e0c60d** in Oregon
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
        * pl-68a54001 (com.amazonaws.us-west-2.s3, 54.231.160.0/19, 52.218.128.0/17, 52.92.32.0/22) Target vpce-e923e880
        * 0.0.0.0/0 Target NAT Gateway **nat-02b636be6988c3f83** 
    * **sce Second** rtb-0ae4b6cd6
      * Destinations
        * 10.0.0.0/16 Target local
        * pl-68a54001 (com.amazonaws.us-west-2.s3, 54.231.160.0/19, 52.218.128.0/17, 52.92.32.0/22)	Target vpce-e923e880
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
  
