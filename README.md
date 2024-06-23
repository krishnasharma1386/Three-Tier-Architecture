Here's a detailed, step-by-step guide to create a three-tier architecture using AWS services such as CloudFormation, Elastic Load Balancers, Auto Scaling Groups, and a Bastion Host:

---

## Step 1: Create Base Network for Three-Tier Architecture

### Create Network Configuration with CloudFormation

1. **Open AWS Management Console**: Navigate to the CloudFormation service.
2. **Create Stack**: Click on **Create stack** and select **With new resources (standard)**.
3. **Upload VPC.yaml**: Use the provided `VPC.yaml` file to configure the network.
4. **Parameters**:
    - Name the VPC: `ds-vpc`
    - Configure **2 Public Subnets** and **2 Private Subnets**.
    - Add **Internet Gateway** and **NAT Gateway**.
    - Create **2 Route Tables** and configure routes.

![Create Stack](images/cloudformation-create-stack.png)

### VPC.yaml Example
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  VPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: '10.0.0.0/16'
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
      Tags:
        - Key: Name
          Value: ds-vpc

  InternetGateway:
    Type: 'AWS::EC2::InternetGateway'
    Properties:
      Tags:
        - Key: Name
          Value: ds-igw

  AttachGateway:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnetA:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.1.0/24'
      AvailabilityZone: !Select [0, !GetAZs '']

  PublicSubnetB:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.2.0/24'
      AvailabilityZone: !Select [1, !GetAZs '']

  PrivateSubnetA:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.3.0/24'
      AvailabilityZone: !Select [0, !GetAZs '']

  PrivateSubnetB:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.4.0/24'
      AvailabilityZone: !Select [1, !GetAZs '']

  NatGateway:
    Type: 'AWS::EC2::NatGateway'
    Properties:
      AllocationId: !GetAtt EIPForNAT.AllocationId
      SubnetId: !Ref PublicSubnetA

  EIPForNAT:
    Type: 'AWS::EC2::EIP'
    Properties:
      Domain: vpc

  PublicRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref VPC

  PrivateRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref VPC

  PublicRoute:
    Type: 'AWS::EC2::Route'
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId: !Ref InternetGateway

  PrivateRoute:
    Type: 'AWS::EC2::Route'
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: '0.0.0.0/0'
      NatGatewayId: !Ref NatGateway

  PublicSubnetARouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PublicSubnetA
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetBRouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PublicSubnetB
      RouteTableId: !Ref PublicRouteTable

  PrivateSubnetARouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PrivateSubnetA
      RouteTableId: !Ref PrivateRouteTable

  PrivateSubnetBRouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref PrivateSubnetB
      RouteTableId: !Ref PrivateRouteTable

Outputs:
  VPCId:
    Description: The VPC Id
    Value: !Ref VPC
```

### Verify Creation
- **Check VPC and Subnets**: Ensure the VPC and subnets are created under the VPC section.
- **Check Internet and NAT Gateways**: Verify their existence under the respective sections.

![VPC Configuration](images/vpc-configuration.png)

---

## Step 2: Create Elastic Load Balancers

### Create Frontend (Internet-Facing) Load Balancer

1. **Open AWS Management Console**: Navigate to **EC2** > **Load Balancers**.
2. **Create Load Balancer**: Click **Create Load Balancer** > **Application Load Balancer**.
3. **Configure**:
    - **Name**: `frontend-lb`
    - **Scheme**: `internet-facing`
    - **Listeners**: Add HTTP (port 80).
    - **Availability Zones**: Select the two public subnets from your VPC.

![Frontend Load Balancer](images/frontend-lb-configuration.png)

### Create Backend (Internal) Load Balancer

1. **Repeat Steps**: Repeat the above steps but choose **internal** for the scheme.
    - **Name**: `backend-lb`
    - **Scheme**: `internal`
    - **Listeners**: Add HTTP (port 3000).
    - **Availability Zones**: Select the two private subnets from your VPC.

![Backend Load Balancer](images/backend-lb-configuration.png)

### Create Target Groups

1. **Target Groups**:
    - **Frontend Target Group**:
        - **Name**: `frontend-tg`
        - **Target Type**: `instance`
        - **Protocol**: HTTP, Port: 80
        - **Health Check**: `/`
    - **Backend Target Group**:
        - **Name**: `backend-tg`
        - **Target Type**: `instance`
        - **Protocol**: HTTP, Port: 3000
        - **Health Check**: `/health`

![Target Groups](images/target-groups.png)

### Register Targets Later

- **Skip Registration**: You will register instances in the auto-scaling setup.

![Register Targets](images/register-targets.png)

---

## Step 3: Auto Scaling Group

### Create Launch Configuration

1. **Open AWS Management Console**: Navigate to **Auto Scaling** > **Launch Configurations**.
2. **Create Launch Configuration**:
    - **Name**: `frontend-lc`
    - **AMI**: Choose an appropriate AMI for your application.
    - **Instance Type**: e.g., `t2.micro`
    - **User Data**: Install dependencies and start the application.
    - **Security Group**: Create or use one that allows necessary ports.

![Launch Configuration](images/launch-configuration.png)

### Create Auto Scaling Group

1. **Auto Scaling Groups**: Go to the Auto Scaling Groups page.
2. **Create Group**:
    - **Launch Configuration**: Select `frontend-lc`.
    - **VPC**: Choose `ds-vpc`.
    - **Subnets**: Select both public subnets for the frontend and both private subnets for the backend.
    - **Scaling Policies**:
        - **Add Instance**: When CPU >= 80%.
        - **Remove Instance**: When CPU <= 50% (optional).

![Auto Scaling Group](images/auto-scaling-group.png)

### Register Targets in Load Balancers

- **Frontend**: Register frontend instances to `frontend-tg`.
- **Backend**: Register backend instances to `backend-tg`.

![Register Instances](images/register-instances.png)

---

## Step 4: Bastion Host

### Create Bastion Host

1. **Open AWS Management Console**: Navigate to **EC2** > **Instances**.
2. **Launch Instance**:
    - **AMI**: Choose a minimal Linux-based AMI.
    - **Instance Type**: `t2.micro`
    - **Subnet**: `sn-public-subnet-A`
    - **Public IP**: Enable.

![Bastion Host](images/bastion-host.png)

### Configure Security Groups

1. **Bastion Host Security Group**:
    - **Inbound Rules**: Allow SSH (port 22) from your trusted IPs.
2. **Private Instances Security Group**:
    - **Inbound Rules**: Allow SSH (port 22) from Bastion Host's security group.

![Security Group](images/security-group.png)

### SSH to Bastion and Private Instances

- **SSH to Bastion**: Use the provided key pair.
- **SSH to Private Instances**: From the bastion host, SSH into the private instances.

```bash
ssh
