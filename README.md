Terraform: Deploy A Three-Tier Architecture in AWS

Infrastructure as Code (IaC)
The cloud gives us the ability to create our environments quickly, but the problem that arises is how to configure and manage the environments. Manually updating from the console may be acceptable for a small organization in a single region, but what if you have to create and maintain environments in multiple regions? Not only is it an inefficient use of time to create and maintain everything, but it’s also error-prone.

Imagine that you are asked to create an environment in a single Region. Not really a big deal and you are able to complete the task with relative ease. Now you need to do the exact same thing for five more Regions. Not only that, once you have completed the excruciatingly repetitive task, your leadership asks you to make a change that then needs to be applied to all Regions. This example is inefficiency at its finest.

Think about infrastructure as code as a scalable blueprint for your environment. It allows you to provision and configure your environments in a reliable and safe way. By using code to deploy your infrastructure you gain the ability to use the same tools as developers such as version control, testing, code reviews, and CI/CD.

Terraform
HashiCorp Terraform is a tool for building, changing, and versioning infrastructure that has an open-source and enterprise version. Terraform is cloud agnostic and can be used to create multi-cloud infrastructure. It allows IaC in a human readable language called HashiCorp Configuration Language (HCL).

Prerequisites
Install Terraform
Install the AWS CLI
Sign up for an AWS Account
Your preferred IDE (I used Visual Studio Code)
Notes Before Getting Started
I’ll be going through sections of my main.tf Terraform file, but if you’d just like to get to the full code, skip to the end or visit my GitHub.
This Terraform file does not follow best practice of DRY (Don’t Repeat Yourself) code. The best practice would be to use variables and modules rather than using a single file and hard coding.
I’ve just started using Terraform and in a future post will update this file to incorporate variables and modules.
A typical three-tier architecture uses a web, application and database layer. In this project, although I’ve created an application layer, I’m not deploying any instances in the application layer and have not created a Security Group for the application layer. If you decide to do so, you will need to modify some Security Group rules and create an application layer security group.
RDS has Multi-AZ set to true for high availability. If you’d like to save money be sure to set this to false.
The Future Updates As Promised
Terraform: Using Variables and Count
Alright, let’s get started.

Website Script
Create a new directory for this Terraform project.
Inside the new directory create a file named install_apache.sh and use the below code. This code will install an Apache web server on our instances and create a unique landing page for each so we can verify the Application Load Balancer is working.

Configure Provider
Providers are plugins that Terraform requires so that it can install and use for your Terraform configuration. Terraform offers the ability to use a variety of Providers, so it doesn’t make sense to use all of them for each file. We will declare our Provider as AWS.

Create a main.tf file and add each of the following sections to the main.tf file.
From the terminal in the Terraform directory containing install_apache.sh and main.tf run terraform init
Use the below code to set our Provider to AWS and set our Region to us-east-1.

Create VPC and Subnets
Our first Resource is creating our VPC with CIDR 10.0.0.0/16.
web-subnet-1 and web-subnet-2 resources create our web layer in two availability zones. Notice that we have map_public_ip_on_launch = true
application-subnet-1 and application-subnet-2 resources create our application layer in two availability zones. This will be a private subnet.
database-subnet-1 and database-subnet-2 resources create our database layer in two availability zones. This will be a private subnet.

Create Internet Gateway and Route Table
Our first resource block will create an Internet Gateway. We will need an Internet Gateway to allow our public subnets to connect to the Internet.
Just saying that our subnets are public does not make it so. We will need to create a route table and Associate our Web Layer subnets.
The web-rt route table creates a route in our VPC to our Internet Gateway for CIDR 0.0.0.0/0.
The next two blocks are associating web-subnet-1 and web-subnet-2 with the web-rt route table. I feel like you should be able to associate more than one subnet in a single block but I kept getting errors and wasn’t able to find any helpful documentation.
Note: We only create one new public route table and do not need to create a private route table. By default all subnets are associated with the default route table which is set to private by default.


Create Web Servers
webserver1 resource creates a Linux 2 EC2 instance in the us-east-1a Availability Zone.
ami is set to the ami id for the Linux 2 AMI for the us-east-1 Region. If using a different Region then you’d need to update.
vpc_security_group_ids is set to a not yet created Security Group, which will be created in the next section for our Application Load Balancer. Terraform doesn’t create infrastructure in order. It is smart enough to know what needs to be created before others (for the most part, I’ll talk more about this later).
user_data is used to boot strap our instance. Rather than type our the code directly, we will reference the install_apache.sh file we created earlier.
webserver2 is almost identical except that availability_zone is set to us-east-1b.

Create Security Groups
Create a Security Group named web-sg with inbound rule opening HTTP port 80 to CIDR 0.0.0.0/0 and allowing all outbound traffic.
Create a Security Group named webserver-sg with inbound rule opening HTTP port 80, but this time it’s not open to the world. Instead we are only allowing traffic from our web-sg Security Group.
Create a Security Group named database-sg with inbound rule opening MySQL port 3306 and once again we keep security tight by only allow the inbound traffic from the webserver-sg Security Group. We open outbound traffic to all the ephemeral ports.

Create Application Load Balancer
Create an external Application Load Balancer.
internal is set to false, making it an external Load Balancer.
load_balancer_type is set to application designating it an Application Load Balancer.
security_groups is set to our web-sg Security Group which allows access from the internet over port 80.
subnets is set to both of our web subnets. This designates where the ALB will send traffic and requires a minimum of two subnets in two different AZs.
2. Create an Application Load Balancer Target Group.

3. The aws_lib_target_group_attachment Resource attaches our instances to the Target Group. Note that I’ve added depends_on to both of these. I kept experiencing an issue where my instances kept showing as unhealthy in the Target Group because they weren’t done initializing. By setting the depends_on to their respective web server, the issue was resolved.

4. Add a listener on port 80 that forwards traffic to our Target Group.


Create RDS Instance
Create an MySQL RDS Instance. Some attributes to note:
db_subnet_group_name is a required field and is set to the aws_db_subnet_group.default.
instance_class is set to a db.t2.micro.
multi_az is set to true for high availability, but if you’d like to keep costs low, set this to false.
username & password will need to be changed.
vpc_secuiryt_group_ids is set to our database-sg Security Group.
2. Create a DB Subnet Group. subnet_ids identifies which subnets will be used by the Database.


Output
After your infrastructure completes, Output will print out the requested values.

We will use output to print out our ALB DNS so we can test our web servers.

Provision Infrastructure
If you didn’t do so earlier or you just want to do it again, from the terminal run terraform init .
Run terraform fmt . This ensures your formatting is correct and will modify the code for you to match.
Run terraform validate to ensure there are no syntax errors.
Run terraform plan to see what resources will be created.
Run terraform apply to create your infrastructure. Type Yes when prompted.
Testing
After your infrastructure has been created there should be an Output displayed on your terminal for the Application Load Balancer DNS Name.
Copy and paste (without quotations) into a new browser tab. Refresh the page to see the load balancer switch between the two instances.
Clean Up
To delete our infrastructure run terraform destroy . When prompted type Yes. This command will delete all the infrastructure that we created.
Congratulations on creating a Three-Tier AWS Architecture!# 3-tier-architecture-with-terraform-and-aws
3-tier-architecture-with-terraform-and-aws:
