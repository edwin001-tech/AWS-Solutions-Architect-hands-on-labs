# Securing your VPC using Public and Private Subnets

In this lab, you will design a VPC with a public subnet, a private subnet, and a Network Address Translation (NAT) device in the public subnet.

A NAT device enables instances in the private subnet to initiate outbound traffic to the Internet. This scenario is common when you have a public-facing web application while maintaining back-end servers that aren't publicly accessible.

A common example is a multi-tier website, with the web servers in a public subnet, and the database servers in a private subnet. You can set up security and routing allowing the web servers to communicate with the database servers. The instances in the public subnet can send outbound traffic directly to the Internet, whereas the instances in the private subnet cannot. The instances in the private subnet can access the Internet via the NAT Gateway in the public subnet. In this Lab, you will also increase the network security using a network access control list (NACL), which is an optional layer of security that acts as a firewall for controlling traffic in and out of a subnet. After completing this Lab, you might consider setting up network ACLs with rules similar to your security groups, in order to add an additional layer of security to your VPC.

## Introduction

Amazon Virtual Private Cloud (Amazon VPC) enables you to launch AWS resources into a virtual network you have defined. This virtual network closely resembles a traditional network that you would operate in your own data center with the benefits of using the scalable infrastructure of AWS. It is logically isolated from other virtual networks in the AWS cloud.

In this lab, you will create a new VPC using the AWS Management Console. Once created, you will create other EC2 and VPC resources mimicking a common two-tiered (front-end and back-end) architecture in the cloud.

### Instructions

1. Click Create VPC to begin creating a new VPC. A Create VPC dialog box is opened for you. Specify the following VPC details:

- Name tag: Enter cloudacademy-labs (This is the name for your VPC. A tag will be created with a key of Name and the value "cloudacademy-labs".)
- CIDR block: Enter 10.0.0.0/16 (This is a CIDR block from the private (non-publicly routable) IP address ranges as specified in RFC 1918.)
- Tenancy: Select Default (Dedicated tenancy ensures your instances run on single-tenant hardware. For the purposes of this Lab, the default is fine though.)

2. Create an Internet Gateway and associate it to a VPC.
   An Internet Gateway is a horizontally scaled, redundant, and highly available VPC component that allows communication between instances in your VPC and the Internet. It imposes no availability risks or bandwidth constraints on your network traffic. An Internet gateway serves two purposes:

- Provide a target in your VPC route tables for Internet-routable traffic
- Perform network address translation (NAT) for instances that have been assigned public IP addresses. (Note: It does not do this for instances with private IP addresses.)

3. Create a public subnet in your VPC.

- VPC ID: Select cloudacademy-labs
- Subnet name: Enter Public-A (This is the name for your subnet. A tag with a key of Name and the value "Public-A" is created)
- Availability Zone: Select us-west-2a from the drop-down menu
- CIDR block: Enter 10.0.20.0/24

4. Create a route table(PublicRouteTable)
   Configure the following route settings:

- Destination: Enter 0.0.0.0/0
- Target: Select Internet Gateway, then labs-gw

next, you will change the default route table of the public subnet to include the new route table.
Select the Public-A subnet and click the Route table tab
Click the Edit route table association button
Select PublicRouteTable from the Route table ID drop-down menu

This subnet will require a route to the internet, so the associated route table has now been configured to use PublicRouteTable to determine traffic rules.

5. Create a Bastion host
   A bastion host is typically a host that sits inside your public subnet for the purposes of SSH (and/or RDP) access. You can think of it as a host for gaining secure access to resources in your VPC from the public internet. Bastion hosts are sometimes referred to as jump servers, as you jump to one, then back out of it.

Once you access a bastion host (for example, by using SSH to log into it), in order to access other instances you must either set up SSH port forwarding or copy your SSH key material to the bastion host. The latter is not ideal for security reasons in a production environment. If you require Windows connectivity, then setting up Remote Desktop Gateway instead of SSH port forwarding is recommended. This lab step assumes SSH connectivity to Linux instances.

In this lab step, you will create an EC2 instance that will serve as both an observer instance that you can run various tests from and a bastion host.

Configure the following security group settings:

Assign a security group: Select Create a new security group

- Security group name: Enter SG-bastion
- Description: Enter SG for bastion host. SSH access only
- Type: SSH
- Protocol: TCP
- Port: 22
- Source: 0.0.0.0/0

In this lab step, you launched an EC2 instance with a public IP address in your public subnet that will be used as a bastion host.

6. Create a Private subnet
   A common use case for private subnets is to configure resources for a back-end tier, such as database servers that should not be accessible from the internet. However, you may eventually want these back-end database servers to access the internet for operating system updates or to be accessible by administrators via a bastion host.

Click Create Subnet and specify the following details:

- VPC ID: Select the cloudacademy-labs VPC from the drop-down menu
- Subnet name: Enter Private-A
- Availability Zone: Select us-west-2a
- CIDR block: Enter 10.0.10.0/24 as the CIDR block of your subnet

7. Create a route table(PrivateRouteTable) and configure the following

- Destination: Enter 0.0.0.0/0
- Target: Select Internet Gateway, then labs-gw
  **_ This is a temporary target value. Later in this lab, you will add a NAT device (gateway or instance) and update the Target for the PrivateRouteTable to the NAT device. _**
  This route will eventually send traffic originating from your private subnet and bound for the public internet, to a NAT device.
  next, associate Private-A subnet with PrivateRouteTable

8. Create a Network Access Control List for your private subnet.
   A Network Access Control List (NACL) is an optional layer of security that acts as a firewall for controlling traffic in and out of a subnet. The network access control list is a numbered list of rules, evaluated in order, to determine whether traffic is allowed in or out of any associated subnet. NACLs are stateless which means they cannot tell if a message is in response to a request that started inside the VPC, or if it is a request that started from the internet. Hence, a NACL is better suited for private subnets. For public subnets, using security groups is recommended without NACLs.
   Configure the following Network ACL settings:

- Name: Enter Private-NACL
- VPC: Select cloudacademy-labs from the drop-down menu
  Select Private-NACL from the Network ACLs list and click the Subnet associations tab
  Click Edit subnet associations and select the check box for the Private-A subnet to associate it with the network ACL

9.  Create inbound and outbound rules for your private Network Access Control List (NACL)
    For inbound rules, configure the following:

- Rule number: Enter 100
- Type: Select SSH
- Source: Enter 10.0.20.0/24
- Allow / Deny: Select Allow from the drop-down menu
  Add new rule
- Rule number: Enter 200
- Type: Select Custom TCP Rule
- Port Range: Enter 1024-65535
- Source: Enter 0.0.0.0/0
- Allow / Deny: Select Allow from the drop-down menu

This will allow return traffic for the outbound rules you will add shortly (the range is specified as 1024-65535 because these are the available ports and not reserved). This enables resources inside the subnet to receive responses to their outbound traffic.
**_Now that you've verified the inbound rules, you will move on to configure the outbound rules. Although the outbound IP addresses can be anything, the ports need to be 80 or 443. In short, operating system updates needed by instances in your private subnet could come from anywhere (0.0.0.0/0), but they will be downloaded over port 80 (HTTP) or 443 (HTTPS). You will need to add rules to account for each port._**

Click Add new rule and configure the following:

- Rule number: Enter 100
- Type: Select HTTP from the drop-down menu
- Destination: Enter 0.0.0.0/0
- Allow / Deny: Select Allow from the drop-down menu

For the second outbound rule, click Add new rule and configure the following:

- Rule number: Enter 200
- Type: Select HTTPS from the drop-down menu
- Destination: Enter 0.0.0.0/0
- Allow / Deny: Select Allow from the drop-down menu

For the third outbound rule, click Add new rule and configure the following:

- Rule number: Enter 300
- Type: Select Custom TCP from the drop-down menu
- Port Range: Enter 32768-61000
- Destination: Enter 10.0.20.0/24 (The CIDR block of your public subnet)
- Allow / Deny: Select Allow from the drop-down menu

10. Create an EC2 instance in a private subnet
    Once the instance is up and running, you will SSH into it from your local host, via a bastion host in the public subnet.

Configure Security Group, fill out accordingly:

- Select Create a new security group
- Security group name: Enter SG-Private
- Description: Enter Security group for private subnet instances. Accept SSH inbound requests from Bastion host only.
  Add the following rules:
- Type SSH; Protocol TCP; Port 22; Source Custom (Select the SecurityGroupOfBastionHost)
  Tip: If you don't recall the name of your bastion host's security group, leave the Source as Custom, and start typing "bastion". It will find the security group for you. (Example: SG-bastion)
- Type HTTPS; Protocol TCP; Port 443; Source Custom 10.0.20.0/24 (Public VPC CIDR)
  **_ Switch to the Inbound rules tab of the security group and locate the SSH rule. If the Source is not the security group of your bastion host, click Edit inbound rules and enter bastion for the Source. _**

Tighten down the bastion host's security group. Select the SG-bastion security group, switch to the Outbound rules tab, and click Edit outbound rules.Configure the following:

- Type: SSH
- Protocol: TCP
- Port: 22
- Destination: Select Custom and then enter the security Security group ID of SG-Private

11. Next, you will SSH into your bastion host, and enable ssh-agent forwarding so you can SSH (jump) to the private instance in your private subnet.
    Download the PEM SSH key file. Then `chmod 400 PEMfilename.pem`
    Since copying SSH private keys to a bastion instance is a security risk, you will enable SSH agent forwarding next. The ssh-add command can add private keys to the keychain application. Essentially, the private key will be used without having to copy it to the bastion host.

Enter `ssh-add -k PEMfilename.pem` to add private keys to the authentication agent and Verify the key was added by `ssh-add -L`
SSH into your bastion host using the authentication agent you just added:
`ssh -A ec2-user@BastionHostPublicIP`
SSH into the private instance running in your private subnet:
`ssh ec2-user@PrivateInstancePrivateIPaddress`
Enter the following command to install operating system updates:
`sudo yum update` insert emoji for timeout
Although the private instance security group is configured correctly, and you should have outbound access to the internet, it still timed out. The time out is caused by the private NACL denying inbound HTTP traffic. You will need Network Address Translation (NAT) to allow your private instance outgoing connectivity to the Internet, while at the same time blocking inbound traffic from the Internet. Once NAT is in place, you should be able to get package updates.

12. Launching a Network Address Translation (NAT) Gateway
    Create NAT gateway:

- Name: Enter NAT-GW
- Subnet: Select Public-A
- Connectivity type: Ensure Public is selected
  The Public connectivity type will allow this NAT Gateway the ability to access the public internet.
  Click Allocate Elastic IP next to the Elastic IP allocation ID.

**_ You will need to attach an Elastic IP address to your NAT Gateway. This allows it to be referenced by the route table responsible for routing outbound traffic from instances in the private subnet to the public internet. _**

This allocates an Elastic IP address for the NAT Gateway to use. When the NAT Gateway is created, this IP address will be attached to the NAT Gateway automatically.

Click Route tables and select PrivateRouteTable. Click Edit routes
Locate the 0.0.0.0/0 Destination route and clear the Target field by clicking the X. Begin typing NAT, then select NAT Gateway, then NAT-GW from the drop-down menu that appears:

**_You created a NAT Gateway to allow your instances in private subnet access to the public internet._**

13. Enter the following command to use the yum package manager to update the operating system packages on your private instance.
    `sudo yum update -y` :+1:

**_ Important! _** There are two important configurations worth mentioning again as to why this command should work in your lab environment:

- The private NACL has an Outbound Rule permitting HTTP (port 80) or HTTPS (port 443) access to anywhere on the internet (0.0.0.0/0)
- The security group for the NAT device allows HTTP/S access from any instance in the private subnet (that uses the private instance security group, which permits any destination as well)

You successfully performed an operating system update on an instance in a private subnet. To do so required access from your private instance, utilizing the route table of a private NACL and a NAT device and its security group.

#### Diagram for the lab
![Diagram](https://github.com/edwin001-tech/AWS-Solutions-Architect-hands-on-labs/blob/main/security/images/security.png)
