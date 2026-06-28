# Day 12 — Container Networking & Infrastructure as Code

## ECS Networking

### Launch Types

| Type | Network Model | When to Use |
|------|--------------|-------------|
| **Fargate** | Serverless (`awsvpc`) | No servers to manage |
| **EC2** | `awsvpc`, `bridge`, `host` | Need control over instance |

### awsvpc Network Mode (Recommended)

Each task/container gets its **own ENI** with a **private IP** from your VPC. This is the closest to EC2 networking.

```
ECS Task
  └── ENI (10.0.1.50)
       ├── Security Group: sg-app
       ├── Private IP: 10.0.1.50
       └── Route Table: rtb-private
```

### Create ECS Service with awsvpc

```bash
aws ecs create-service \
  --cluster prod-cluster \
  --service-name web-app \
  --task-definition web-td:1 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-app-1a", "subnet-app-1b"],
      "securityGroups": ["sg-app"],
      "assignPublicIp": "DISABLED"
    }
  }' \
  --load-balancers '[
    {
      "targetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:xxx:targetgroup/web-tg/xxx",
      "containerName": "web",
      "containerPort": 80
    }
  ]'
```

### Key Network Facts

- **Each task = one ENI** — subnets need enough IPs for your task count
- **Security Groups** apply per task (not per host)
- **Dynamic port mapping**: with ALB, containers can use random host ports
- **Service discovery**: use AWS Cloud Map (`aws servicediscovery`) for DNS-based service discovery
- **VPC Endpoints**: required for Fargate tasks to pull images from ECR (otherwise NAT needed)

## EKS Networking

### Amazon VPC CNI (Default)

Each pod gets an **IP from the VPC subnet** — no overlay network, no NAT.

```
Worker Node (EC2)
  ├── eth0 (primary ENI: 10.0.1.50)
  ├── eth1 (secondary ENI: 10.0.1.51)
  │    ├── pod-A (10.0.1.51)
  │    └── pod-B (10.0.1.51)
  └── eth2 (secondary ENI: 10.0.1.52)
       └── pod-C (10.0.1.52)
```

### How It Works

1. VPC CNI plugin allocates **secondary IPs** to worker node ENIs
2. Each secondary IP = one pod
3. Pods communicate directly via VPC router — no encapsulation
4. Security Groups can be applied per pod (using Security Groups for Pods)

### Network Considerations

| Factor | EKS Impact |
|--------|-----------|
| **IP exhaustion** | Each pod uses a VPC IP — plan subnet sizes accordingly |
| **Node ENI limit** | Instance type determines max ENIs and IPs per ENI |
| **Pod density** | More pods per node = more ENIs/IPs needed |
| **Security Groups for Pods** | Assign SGs to individual pods (not just nodes) |

### Security Groups for Pods

```bash
# Enable on EKS cluster
aws eks create-cluster \
  --name prod \
  --resources-vpc-config '{
    "subnetIds": ["subnet-1a", "subnet-1b"],
    "securityGroupIds": ["sg-cluster"]
  }' \
  --kubernetes-network-config 'serviceIpv4Cidr=10.100.0.0/16'
```

## Infrastructure as Code (IaC)

### Why Network Admins Need IaC

| Without IaC | With IaC |
|-------------|----------|
| Click through Console | Single `aws cloudformation deploy` |
| Manual route table entries | Route tables defined in code |
| Document changes in wiki | Changes tracked in Git |
| Drift is common | Config can detect drift |
| Repeatability is hard | Every deploy is identical |

### CloudFormation

Define VPC infrastructure as YAML/JSON templates.

### Example: VPC with Public and Private Subnets

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Basic VPC with public and private subnets

Parameters:
  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
  EnvironmentName:
    Type: String
    Default: prod

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-vpc

  PublicSubnetA:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: !Select [0, !Cidr [!Ref VpcCIDR, 4, 8]]
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true

  InternetGateway:
    Type: AWS::EC2::InternetGateway
  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref InternetGateway

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: IGWAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  PublicSubnetARouteAssoc:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetA
      RouteTableId: !Ref PublicRouteTable

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
  PublicSubnetA:
    Description: Public Subnet A
    Value: !Ref PublicSubnetA
```

### Deploy with CLI

```bash
aws cloudformation deploy \
  --template-file vpc.yaml \
  --stack-name prod-network \
  --parameter-overrides VpcCIDR=10.0.0.0/16 EnvironmentName=prod \
  --capabilities CAPABILITY_NAMED_IAM
```

### Terraform (HashiCorp)

Equivalent in Terraform:

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.environment}-vpc"
  }
}

resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(aws_vpc.main.cidr_block, 8, 0)
  availability_zone       = "${data.aws_region.current.name}a"
  map_public_ip_on_launch = true
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.gw.id
}
```

### CDK (AWS Cloud Development Kit)

Define infrastructure with real programming languages:

```typescript
import * as ec2 from 'aws-cdk-lib/aws-ec2';

const vpc = new ec2.Vpc(this, 'ProdVPC', {
  ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
  maxAzs: 2,
  subnetConfiguration: [
    { name: 'Public',  subnetType: ec2.SubnetType.PUBLIC },
    { name: 'Private', subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
  ],
});
```

### IaC Best Practices for Network Admins

| Practice | Why |
|----------|-----|
| **Version control** | Git-track all network templates |
| **Parameterize CIDRs** | Don't hardcode — use parameters/variables |
| **Stack outputs** | Export VPC/Subnet IDs for other stacks to reference |
| **Change sets** | Preview changes before applying (CloudFormation) |
| **Plan** | `terraform plan` before `terraform apply` |
| **State management** | Store Terraform state in S3 with DynamoDB locking |
| **StackSets** | Deploy network stacks across multiple accounts/regions |

### CLI Commands

```bash
# CloudFormation
aws cloudformation deploy
aws cloudformation create-change-set
aws cloudformation describe-stacks
aws cloudformation delete-stack

# ECS
aws ecs create-service
aws ecs describe-services

# EKS
aws eks create-cluster
aws eks update-kubeconfig
```

## Console

**ECS → Clusters** — manage tasks, services, networking
**EKS → Clusters** — manage Kubernetes networking
**CloudFormation → Stacks** — create, update, delete
**CloudFormation → StackSets** — multi-account deployments

## Gotchas

- **ECS Fargate ENI limits**: each task gets an ENI — subnets must have available IPs
- **EKS IP exhaustion**: plan /21 or larger subnets for EKS worker nodes
- **VPC CNI vs Calico/Cilium**: third-party CNIs may use overlay (better IP utilization)
- **CloudFormation stack updates**: if a resource doesn't support updates, you must recreate
- **Terraform state**: store remotely (S3) — never commit `terraform.tfstate` to git
- **IaC doesn't replace Config**: use CloudFormation/ Terraform to provision, Config to audit
- **StackSets**: cross-account deployments require Organizations + RAM
- **CDK**: synthesizes to CloudFormation — same limits apply
