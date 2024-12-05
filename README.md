# aws-autoscaling
# **Create a load-balanced web server with auto scaling**

# Overview
This tutorial walks you through the process of creating a web server which is managed by an auto scaling group. The auto scaling group uses a launch template to create the servers. The auto scaling group launches the servers into a target group. An internet-facing load balancer directs traffic to the target group.

Errors or corrections? Contact darkosloutions@gmail.com.

## Architecture Diagram
![](/images/autoscale-diagram-Page-1.drawio2.png)


## Create a VPC
A VPC is an isolated, private network you can create to run your workloads. You have complete control over your VPC when you create one.

1. Click on [Create VPC](https://eu-west-1.console.aws.amazon.com/vpc/home?region=eu-west-1#CreateVpc:createMode=vpcWithResources) from the VPC Dashboard to create a new VPC
2. Select **VPC and more**
3. Enter a name of your choice under **Auto-generate**
4. Choose a 10.0.0.0/16 IPV4 Cidr block
5. Number of Availability Zones (AZs) = "2"
6. Number of public subnets = "2"
7. NAT gateways ($) = "In 1 AZ"
8. Optional: VPC endpoints = "S3 Gateway"
9. Leave all other settings as default
10. Click Create VPC
    
Your configuration should match what is shown below:

[
](https://github.com/DarkoVibez/aws-autoscaling/blob/4b875faa27a432e510f484a7c05f70b866f9e7cd/Screenshot%202023-07-10%20163913.png) 

Create a Launch Template
The launch template will serve as the blueprint for creating the exact type of server we need to meet our web server demands. A launch template can be modified to create new versions when you need to change a config.

Click on Create launch template from the EC2 console to create a new launch template
Launch template name - required = "autoscale-webserver"
Check the Provide guidance to help me set up a template that I can use with EC2 Auto Scaling box
Under Application and OS Images (Amazon Machine Image) - required, choose Amazon Linux 2 AMI (HVM) - Kernel 5.10, SSD Volume Type
Instance type = "t2.micro"
Key pair - Create a new one or use existing key pair
Subnet - Don't include in launch template
Create security group = "autoscale-webserver-sg"
Allow SSH and HTTP traffic from 0.0.0.0/0 (Ignore the warning about security group. We will edit it later)
VPC - Select the VPC you created
Under Advanced network configuration, choose "Enable" under Auto-assign public IP
Under Storage, leave all other configuration as default and choose "gp3" for Volume type
Resource tags: Key: Name, Value: autoscale-webserver
Under Advanced details, scroll down to the User data section and enter the following lines of code exactly as shown (Works on Amazon Linux 2)
#!/bin/bash -ex
sudo su
yum -y update
yum install httpd -y
systemctl start httpd
systemctl enable httpd
systemctl status httpd
echo "<html>Hello World, welcome to my server</html>" > /var/www/html/index.html
systemctl restart httpd
amazon-linux-extras install epel -y
yum install stress -y
Your configuration should look like this:

      

Create Target Group
A target group will route requests to the web servers we create. Our load balancer will need this target group to know what set of servers to distribute traffic to. Our auto scaling group will also be associated with this target group so it launches our servers into the target group.

Click on Create target group from the EC2 console to create a target group
Choose a target type: "Instances"
Target group name: "autoscale-webserver"
Protocol: "HTTP"
Port: "80"
VPC: Select the VPC you created
Leave every other value on this page as default. Next
Register Targets: Leave as is.
Click Create target group


Create Load Balancer
An application load balancer acts as the entry point for traffic to our webservers. Instead of allowing users to access our application directly, we will use the load balancer to distribute traffic equally among our autoscaling group of web servers. This is better for load management, security and reliability of our application.

Click on Create load balancer from the EC2 console to create a load balancer
Type: "Application Load Balancer"
Scheme: "Internet-facing"
IP address type: "IPV4"
VPC: Select the VPC you created
Mappings: Check the box beside the two AZs listed
Subnet: For each AZ selected, choose the public subnet in the dropdown menu
At this point, go to the Security groups console and create a new security group for the load balancer. The inbound rule should allow HTTP traffic from anywhere.
Select this security group as the load balancer security group
Listeners and routing: Leave protocol and port as HTTP:80. Select the target group you created as target group
Leave every other config as default and click Create load balancer
    

Create Auto Scaling Group
The auto scaling group configures and controls how your application scales automatically in response to varying traffic situations.

Click on Create Auto Scaling group from the EC2 console to create an auto scaling group
Enter a name
Choose the launch template you created. Click Next
Select your webserver VPC created from the VPC step
Under Availability Zones and subnets, select the two public subnets in your VPC, in different AZs. Click Next
NB: Note that you can use the auto scaling group to override your instance type config from the launch template

Under Load balancing, choose the option "Attach to an existing load balancer"
Select Choose from your load balancer target groups
Select the target group you created
Select VPC Lattice service to attach: "No VPC Lattice service"
Additional health check types - optional: "Turn on Elastic Load Balancing health checks"
Leave every other config as default. Next
Group size: Desired: "2", Minimum: "1", Maximum: "4"
Scaling policies: "Target Tracking Policy"
Metric type: "Average CPU Utilization"
Target Value: "50%"
Add notifications - optional (Skipped)
Add tags - optional (Skipped)
Create Auto Scaling Group
Check your configuration below:

    

Immediately you create your autoscaling group, you should see two new instances getting created in the EC2 console. This is because we specified a desired count of 2. Also note that they are automatically placed one in each AZ to support high availability.



Test your web server
Click on one of the web servers, copy the public IP or DNS name and paste it in your browser. You should see the following content:



This means your apache web server is running and you are able to reach it from the internet.

Restrict web traffic to servers
With the current design, users are directly accessing our web server. We don't want that. That is why we created a load balancer. To restrict incoming HTTP traffic destined for our servers to only the load balancer, we need to update the web servers' security group to accept HTTP traffic from only our application load balancer. This means no request gets to our servers without first making it through the load balancer.

Edit web server security group
Go to the autoscale-webserver-sg security group and click on "Edit inbound rules".
Delete the existing HTTP rule.
Add a new HTTP rule. In the Source box, scroll down to select the security group of the load balancer. Save rules.
You have successfully restricted traffic going to the servers to the load balancer.
You should no longer be able to access your web server using the server IPs or DNS names. You should now be able to use the load balancer DNS name to access the servers. Test this out.


Live Autoscaling Test
We will now simulate a scenario of high CPU usage on our web server to allow the auto scaling group to respond.

We will SSH to our server and run a command to stress the server and this will raise CPU usage across our auto scaling group above 50%. This will make it respond by adding new servers until our maximum number of servers specified is reached.



Observe your EC2 console after running this command and you will see new instances being added by the auto scaling group to handle the simulated surge in traffic.

If you stopped or terminated the instance on which CPU usage has been simulated, instances will get terminated from your auto scaling group accordingly (scale-in).



Conclusion
You have built a load-balanced and highly available web application that auto scales out based on a target of CPU utilization.
