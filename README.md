# aws-production-3tier-cicd

AWS CICD Pipeline :
It is a major project where most of the things are AWS based
—> Production Grade Project
We are going to use CodeBuild,CodeDeploy,CodePipeline

—> Deploy Production-Grade Full stack application
 
Overview:
(Refer To Diagram)
-> We’ll be working under AWS - account
-> Everything happens under isolated network called VPC
-> Within VPC we have subnets —> we need about 8 subnets for this project
-> Consider Application as a 3 tier application —> DB,Backend and Frontend
-> Apart from that we’ll have servers for bastion host to forward traffic from public to private subnets we’ll require gateway helps(intermediary servers)
-> These help us to create resources in the private subnets without directly exposing them to the internet
->  DB,Backend and Frontend are in the private subnet(private subnet where IG is not attached Internet Gateway)
-> To make this app accessible to the users we’ll have public subnets to internally forward traffic to the users
-> We’ll have VPC and within which we’ll have multiple subnets
*VPC is regional specific while subnets are zone specific
—> 2 bastion host —> intermediary server(External users can access the internal app in the private subnet using it)
—> The 2 bastion host are public subnets while all the below are private subnets
—> 2 servers for frontend (web-layer)
—> 2 for backend (app-layer)
—> 2 for database 
We’ll have each above in 2 different zones (say 1a and 1b)
Setting up ASG for webservers


*The tech stack can be anything
*Both backend apps connect to a single db while the other db (in other zone) acts as a backup
-> this is because db is only one ->as data is consistent

—> We’ll have 2 load balancers -> external and internal load balancer 
*using external LB the traffic goes to frontend and using the internal LB the traffic is distributed to backend subnets
Route table and stuff used

—> To collect logs of application we can have/use CloudWatch logs 
—> We can also have cloud watch alarms and sns  for notifications

-> We’ll also may be use CloudFront or Route53 for dns resolution

People can request to access our application

*We have Jenkins Pipelines so far , but this project is a complete project that will be using only AWS services 
—> We use CodePipeline for above

*The pattern remains same and can be done later 

Documentation of Steps :
Create 1 VPC (Subnets - 2 public and 6 private subnets )
Security Groups(multiple inbound and outbound rules)
Create s3 bucket for the code
IAM roles
Create RDS MySql Database
Create Application Server for backend
Create Webserver for frontend
Create External Load Balancer for webservers
Create Internal Load Balancer for Application Server
Setup the ASG (Auto Scaling Groups) for webservers
Setup the ASG (Auto Scaling Groups) for app servers
Parameter store for storing secrets(passwords)
Create s3 Private bucket for Artifacts(reusable files)(jar files)

Configuring Code Build for Application Server
Configuring Code Deploy for Application Server
Configuring AWS CodePipeline for Application Server

Configuring Code Build for Web Server
Configuring Code Deploy for Web Server
Configuring AWS CodePipeline for Web Server

Build Yaml files where Code build process is specified (specifications.yml) to specify the Cinfigurations

Once App is launched Setup HTTPS —> Setup Route53(if possible)

=========================================================
Nat gateway is in the public subnet always
*Always delete the resources to avoid bills
NAT Gateway is associated with elastic IP therefor it leads to bill generation

SG needs to be configured in order to configure the availability of web-private-subnet
We’ll create 6 security groups for different purposes
1 for Jump server
1 web ALB
1 app server
1 web App server
1 app private (internal) LB
1 db server

=========================================================
Operations Performed :

Create A VPC
Create 8 subnets - 2 public and 6 private 
Create IGW and attach it to the VPC
Create Nat Gateway in public subnet 1a and allocate elastic public ip
Create 2 route tables public-rt and private-rt
In public-rt attach IGW and associate public subnets
In private-rt attach NGW and associate the 6 private subnets

*Select public subnets —> action —> subnet settings —> Enable-Auto assign public IP

Now as stated above creation of 6 SG is done
Attach and provide inbound rules as per diagram

Create an EC2 instance Jump-Server in newly created VPC and attach JUMP server Sg to it
Create 3 IAM roles for EC2, AWS-CodeBuild, AWS-CodeDeploy and AWS-CodePipeline

For Ec2
Type - Ec2
Permissions = CloudWtachLogsFullAccess, CloudWatchAgentServerPolicy, AmazonEc2RoleForAWSCodeDeploy
Name = multi-tier-ec2role

=========================================================
Operations Performed :

~ Create 2 more roles :
Multi-tier-codebuild-role
—>AmazonS3FullAccess (for artifacts)
—>AmazonCodeBuildAdminAccess
—>AmazonSSMReadOnlyAccess (For Parameter Store)
—>CloudWatchLogsfullaccess

Multi-tier-codedeploy-role
—>AWSCodeDeployRole
—>CloudWatchLogsFullAccess
—>CloudWatcgAgentServerPolicy

Create a SubnetGroup and configure the private db subnets 1a and 1b to it

Create a database of MYSQL FreeTier A to Z ,username = admin , password = any of choice
Public access ; No
Security Group : DB-SG
Encryption: yes
Also VPC must be : multi-tier-project

Now get connected to Jump-Server 
Using Ai tools install my sql client
And using the endpoint of created db connect to the database locally using VM
Use command
mysql -u <endpoint> -u admin -p
Enter the password:

Now We are required to create a database and a user
Create two tables author and books 

—> We are done with db and jump server now we need to configure the Application Tier and Web Tier
Application Tier Setup:
-> Create launch Template for application tier for ec2 instance
-> Create empty target group
-> Create Internal Application Load Balancer
-> Create Auto Scaling Group

Setup for Web Tier
-> Create launch Template for application tier for ec2 instance
-> Create empty target group
-> Create External Load Balancer
-> Creae Auto Scaling Group

Application Tier (Middle Tier)
============================================
Launch Template
Name : application-tier-LT
AMI:Amazon Linux 2023
Type : t2.micro
SG : App-SG
Role: multi-tier-ec2-role
User Data : Node.js,PM2,MariaDB client,CloudWatch,CodeDeploy agent

Target Group
Name:app-tier-tg
Protocol:HTTP,Port:3200
Health Check: /health
Internal ALB
Name : app-tier-internal-alb
Scheme:Internal
Subnets: Private App Subnets
Security Group : AppALB-SG

ASG
Name : application-tier-ASG
LT:application-tier-LT
Min: 2, Max:4, Desired:2
Attach to app-tier-tg
Scaling Policy: AVG CPU 70%

Web/Presentation tier (Front tier):
============================================
Launch Template
Name: web-tier-Lt
AMI:Amazon-Linux 2023
Type:t2.micro
SG : Web-SG
Role: Multi-tier-Ec2-role
UserData : Nginx , CloudWatch , CodeDeploy , proxy to App tier ALB

Target Group
Name : web-tier-tg
Protocol : HTTP ,Port : 80
Heath Check : /health

External ALB
Name : web-tier-external-ALB
Scheme : Internet-Facing
Subnets : Public Web Subnets
Security Group : WebALB-SG

We’ll be using a parameter for user data  log group name is : node-app-logs-backend
At the time of machine creation it has been initiated and requires to be created as :
Serach CloudWatch —> Logs —> Logs group —> We have many created as per user data but we are required to create a missing one
Create Log group —> node-app-logs-backend —>Create 

It will be the place where all the logs will come and be collected

Now,We are required a buildspec.yml where configurations such as db-passwords,hosts,port no.,db-user,db-name and so on will be added 
To do so follow as :
Search System Manager —> To manage and View AWS resources
—>Application Tools —> Parameter Store —> Create Parameter (It’s a good way to manage the credentials)
—> 1 for DB passwords —> 1 for host(db location) —> 1 for port no. —>1 for db-user —>1 for db-name

*Only Core Knowledge is sufficient due to tools variation 




