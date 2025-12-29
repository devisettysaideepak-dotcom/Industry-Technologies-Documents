# Cloud Architecture Guide: AWS, Google Cloud, and Microsoft Azure

## Amazon Web Services (AWS) Architecture

### Core AWS Components Overview

```
Internet Gateway
    ↓
VPC (Virtual Private Cloud)
    ↓
Availability Zones
    ↓
Subnets (Public/Private)
    ↓
Security Groups + NACLs
    ↓
EC2 Instances / Services
```

### Complete AWS Architecture Diagram

```
                    Internet
                       ↓
              [Route 53 (DNS)]
                       ↓
              [CloudFront (CDN)]
                       ↓
              [AWS WAF (Web Application Firewall)]
                       ↓
              [Internet Gateway (IGW)]
                       ↓
    ┌─────────────────────────────────────────────────┐
    │                VPC (10.0.0.0/16)                │
    │                                                 │
    │  [VPC Endpoints] ←→ [AWS Services (S3, DynamoDB)]│
    │                                                 │
    │  ┌─────────────────┐    ┌─────────────────┐    │
    │  │ Availability    │    │ Availability    │    │
    │  │ Zone A          │    │ Zone B          │    │
    │  │                 │    │                 │    │
    │  │ ┌─────────────┐ │    │ ┌─────────────┐ │    │
    │  │ │Public Subnet│ │    │ │Public Subnet│ │    │
    │  │ │10.0.1.0/24  │ │    │ │10.0.3.0/24  │ │    │
    │  │ │             │ │    │ │             │ │    │
    │  │ │[ALB/NLB/CLB]│ │    │ │[ALB/NLB/CLB]│ │    │
    │  │ │[NAT Gateway]│ │    │ │[NAT Gateway]│ │    │
    │  │ │[Bastion Host│ │    │ │[Elastic IP] │ │    │
    │  │ │/Jump Box]   │ │    │ │             │ │    │
    │  │ └─────────────┘ │    │ └─────────────┘ │    │
    │  │                 │    │                 │    │
    │  │ ┌─────────────┐ │    │ ┌─────────────┐ │    │
    │  │ │Private      │ │    │ │Private      │ │    │
    │  │ │Subnet       │ │    │ │Subnet       │ │    │
    │  │ │10.0.2.0/24  │ │    │ │10.0.4.0/24  │ │    │
    │  │ │             │ │    │ │             │ │    │
    │  │ │[EC2]        │ │    │ │[EC2]        │ │    │
    │  │ │[RDS]        │ │    │ │[RDS]        │ │    │
    │  │ │[Lambda]     │ │    │ │[Lambda]     │ │    │
    │  │ │[EFS/FSx]    │ │    │ │[EFS/FSx]    │ │    │
    │  │ └─────────────┘ │    │ └─────────────┘ │    │
    │  └─────────────────┘    └─────────────────┘    │
    │                                                 │
    │  [VPC Peering] ←→ [Other VPCs]                  │
    │  [Transit Gateway] ←→ [On-Premises via VPN/DX]  │
    │  [VPC Flow Logs] → [CloudWatch/S3]              │
    └─────────────────────────────────────────────────┘
                       ↓
              [Route Tables (Public/Private/Custom)]
                       ↓
              [Security Groups (Stateful)]
                       ↓
              [Network ACLs (Stateless)]
                       ↓
              [AWS Shield (DDoS Protection)]
```

### AWS Request Flow Example

**Complete Web Application Request Flow:**
```
1. User Request
   ↓
2. Route 53 (DNS) → Resolves domain to CloudFront/ALB IP
   ↓
3. CloudFront (CDN) → Cache check, edge location routing
   ↓
4. AWS WAF → Web application firewall filtering
   ↓
5. AWS Shield → DDoS protection
   ↓
6. Internet Gateway → Entry point to VPC
   ↓
7. Application Load Balancer (Public Subnet)
   ↓
8. Security Group Check → Allow HTTP/HTTPS (stateful)
   ↓
9. Network ACL Check → Subnet-level filtering (stateless)
   ↓
10. Route Table → Direct to Private Subnet
    ↓
11. EC2 Instance (Private Subnet)
    ↓
12. IAM Role Check → Verify permissions
    ↓
13. VPC Endpoint → Access AWS services (S3, DynamoDB) privately
    ↓
14. API Gateway → Route to Lambda/Services
    ↓
15. Lambda Function → Business logic
    ↓
16. RDS Database (Private Subnet)
    ↓
17. VPC Flow Logs → Log network traffic to CloudWatch
    ↓
18. Response back through same path
```

### AWS Core Components Explained

#### 1. VPC (Virtual Private Cloud)
```
Purpose: Isolated network environment in AWS cloud
CIDR Block: 10.0.0.0/16 (65,536 IP addresses)

Components:
- Subnets (divide VPC into smaller networks)
- Internet Gateway (internet access)
- NAT Gateway (outbound internet for private subnets)
- Route Tables (traffic routing rules)
- Security Groups (instance-level firewall)
- NACLs (subnet-level firewall)
```

#### 2. Subnets
```
Public Subnet:
- Has route to Internet Gateway
- Resources get public IP addresses
- Used for: Load Balancers, NAT Gateways, Bastion Hosts

Private Subnet:
- No direct internet access
- Uses NAT Gateway for outbound internet
- Used for: Application servers, Databases, Internal services
```

#### 3. Security Groups (Instance-level Firewall)
```javascript
// Example Security Group Rules
{
  "GroupName": "web-server-sg",
  "Rules": [
    {
      "Type": "Inbound",
      "Protocol": "HTTP",
      "Port": 80,
      "Source": "0.0.0.0/0"  // Allow from anywhere
    },
    {
      "Type": "Inbound", 
      "Protocol": "HTTPS",
      "Port": 443,
      "Source": "0.0.0.0/0"
    },
    {
      "Type": "Inbound",
      "Protocol": "SSH", 
      "Port": 22,
      "Source": "10.0.0.0/16"  // Only from VPC
    },
    {
      "Type": "Outbound",
      "Protocol": "All",
      "Port": "All", 
      "Destination": "0.0.0.0/0"  // Allow all outbound
    }
  ]
}
```

#### 4. Route Tables
```
Public Subnet Route Table:
Destination     Target
10.0.0.0/16    Local (VPC traffic)
0.0.0.0/0      Internet Gateway (internet traffic)

Private Subnet Route Table:
Destination     Target  
10.0.0.0/16    Local (VPC traffic)
0.0.0.0/0      NAT Gateway (outbound internet)
```

#### 5. IAM (Identity and Access Management)
```json
// Example IAM Policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow", 
      "Action": "dynamodb:*",
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/Users"
    }
  ]
}

// IAM Role for EC2
{
  "RoleName": "EC2-S3-Access-Role",
  "AssumeRolePolicyDocument": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }
}
```

#### 7. Additional AWS Networking Services

**Route 53 (DNS Service):**
```
Features:
- Domain registration and DNS hosting
- Health checks and failover
- Geolocation and latency-based routing
- Integration with AWS services

Example DNS Records:
A Record: example.com → 203.0.113.12
CNAME: www.example.com → example.com
ALIAS: example.com → ALB DNS name
```

**CloudFront (CDN):**
```
Global Content Delivery Network:
- Edge locations worldwide (200+ locations)
- Static and dynamic content caching
- SSL/TLS termination
- Origin failover
- Real-time logs and metrics

Cache Behaviors:
Path Pattern: /api/* → Origin: ALB
Path Pattern: /images/* → Origin: S3 bucket
Default: * → Origin: ALB
```

**AWS WAF (Web Application Firewall):**
```javascript
// Example WAF Rules
{
  "Rules": [
    {
      "Name": "SQLInjectionRule",
      "Statement": {
        "SqliMatchStatement": {
          "FieldToMatch": {
            "AllQueryArguments": {}
          }
        }
      },
      "Action": "Block"
    },
    {
      "Name": "RateLimitRule", 
      "Statement": {
        "RateBasedStatement": {
          "Limit": 2000,
          "AggregateKeyType": "IP"
        }
      },
      "Action": "Block"
    }
  ]
}
```

**VPC Endpoints:**
```
Gateway Endpoints (Free):
- S3: com.amazonaws.region.s3
- DynamoDB: com.amazonaws.region.dynamodb

Interface Endpoints (Paid):
- EC2: com.amazonaws.region.ec2
- Lambda: com.amazonaws.region.lambda
- SNS: com.amazonaws.region.sns

Benefits:
- Private connectivity to AWS services
- No internet gateway required
- Reduced data transfer costs
- Enhanced security
```

**Transit Gateway:**
```
Hub-and-spoke connectivity:
VPC A ←→ Transit Gateway ←→ VPC B
VPC C ←→ Transit Gateway ←→ On-premises (VPN/Direct Connect)

Features:
- Centralized routing
- Cross-region peering
- Multicast support
- Route table propagation
```

**Direct Connect:**
```
Dedicated network connection:
On-premises → Direct Connect Location → AWS Backbone

Benefits:
- Consistent network performance
- Reduced bandwidth costs
- Private connectivity
- Hybrid cloud architectures

Virtual Interfaces (VIFs):
- Private VIF: Access VPC resources
- Public VIF: Access AWS public services
- Transit VIF: Access multiple VPCs via Transit Gateway
```

**VPC Flow Logs:**
```
Network traffic monitoring:
- Capture IP traffic information
- Log to CloudWatch Logs, S3, or Kinesis
- Troubleshoot connectivity issues
- Security analysis

Log Format:
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes windowstart windowend action flowlogstatus
```

**Elastic Load Balancers:**
```
Application Load Balancer (ALB) - Layer 7:
- HTTP/HTTPS traffic
- Content-based routing
- WebSocket support
- HTTP/2 support

Network Load Balancer (NLB) - Layer 4:
- TCP/UDP traffic
- Ultra-high performance
- Static IP addresses
- Preserve source IP

Classic Load Balancer (CLB) - Legacy:
- Basic load balancing
- Layer 4 and 7
- Not recommended for new applications
```

---

## Google Cloud Platform (GCP) Architecture

### Core GCP Components Overview

```
Cloud Load Balancer
    ↓
VPC Network
    ↓
Regions & Zones
    ↓
Subnets
    ↓
Firewall Rules
    ↓
Compute Engine / Services
```

### Complete GCP Architecture Diagram

```
                    Internet
                       ↓
              [Cloud DNS]
                       ↓
              [Cloud CDN]
                       ↓
              [Cloud Armor (DDoS + WAF)]
                       ↓
              [Cloud Load Balancer (Global)]
                       ↓
    ┌─────────────────────────────────────────────────┐
    │            VPC Network (Global)                 │
    │                                                 │
    │  [Private Google Access] ←→ [Google APIs]       │
    │  [Private Service Connect] ←→ [Partner Services]│
    │                                                 │
    │  ┌─────────────────┐    ┌─────────────────┐    │
    │  │ Region: us-east1│    │ Region: us-west1│    │
    │  │                 │    │                 │    │
    │  │ ┌─────────────┐ │    │ ┌─────────────┐ │    │
    │  │ │Zone: us-    │ │    │ │Zone: us-    │ │    │
    │  │ │east1-a      │ │    │ │west1-a      │ │    │
    │  │ │             │ │    │ │             │ │    │
    │  │ │Subnet:      │ │    │ │Subnet:      │ │    │
    │  │ │10.1.0.0/24  │ │    │ │10.2.0.0/24  │ │    │
    │  │ │             │ │    │ │             │ │    │
    │  │ │[Compute     │ │    │ │[Compute     │ │    │
    │  │ │ Engine]     │ │    │ │ Engine]     │ │    │
    │  │ │[GKE]        │ │    │ │[GKE]        │ │    │
    │  │ │[Cloud SQL]  │ │    │ │[Cloud SQL]  │ │    │
    │  │ │[Cloud NAT]  │ │    │ │[Cloud NAT]  │ │    │
    │  │ └─────────────┘ │    │ └─────────────┘ │    │
    │  │                 │    │                 │    │
    │  │ ┌─────────────┐ │    │ ┌─────────────┐ │    │
    │  │ │Zone: us-    │ │    │ │Zone: us-    │ │    │
    │  │ │east1-b      │ │    │ │west1-b      │ │    │
    │  │ └─────────────┘ │    │ └─────────────┘ │    │
    │  └─────────────────┘    └─────────────────┘    │
    │                                                 │
    │  [VPC Peering] ←→ [Other VPCs]                  │
    │  [Cloud VPN/Interconnect] ←→ [On-Premises]      │
    │  [VPC Flow Logs] → [Cloud Logging]              │
    └─────────────────────────────────────────────────┘
                       ↓
              [Firewall Rules (Hierarchical)]
                       ↓
              [IAM & Service Accounts]
                       ↓
              [Cloud Security Command Center]
```

### GCP Request Flow Example

**Web Application Request Flow:**
```
1. User Request
   ↓
2. Cloud DNS → Resolves domain to Load Balancer IP
   ↓
3. Cloud Load Balancer (Global)
   ↓
4. Firewall Rules Check → Allow HTTP/HTTPS
   ↓
5. VPC Network Routing
   ↓
6. Compute Engine Instance
   ↓
7. Service Account Check → Verify permissions
   ↓
8. Cloud Functions/Cloud Run → Business logic
   ↓
9. Cloud SQL/Firestore → Database access
   ↓
10. Response back through same path
```

### GCP Core Components Explained

#### 1. VPC Network (Global)
```
Key Differences from AWS:
- Global by default (spans all regions)
- Subnets are regional (not zonal)
- No Internet Gateway concept
- Automatic routing between regions

Example VPC:
Name: my-vpc-network
Mode: Custom
Subnets:
  - us-east1: 10.1.0.0/24
  - us-west1: 10.2.0.0/24
  - europe-west1: 10.3.0.0/24
```

#### 2. Firewall Rules
```yaml
# Example Firewall Rules
- name: allow-http-https
  direction: INGRESS
  priority: 1000
  source-ranges: ['0.0.0.0/0']
  allowed:
    - IPProtocol: tcp
      ports: ['80', '443']
  target-tags: ['web-server']

- name: allow-ssh-internal
  direction: INGRESS
  priority: 1000
  source-ranges: ['10.0.0.0/8']
  allowed:
    - IPProtocol: tcp
      ports: ['22']
  target-tags: ['ssh-access']

- name: deny-all-ingress
  direction: INGRESS
  priority: 65534
  source-ranges: ['0.0.0.0/0']
  action: DENY
```

#### 3. IAM & Service Accounts
```json
// Service Account for Compute Engine
{
  "name": "projects/my-project/serviceAccounts/compute-sa@my-project.iam.gserviceaccount.com",
  "displayName": "Compute Engine Service Account"
}

// IAM Policy Binding
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "serviceAccount:compute-sa@my-project.iam.gserviceaccount.com"
      ]
    },
    {
      "role": "roles/cloudsql.client", 
      "members": [
        "serviceAccount:compute-sa@my-project.iam.gserviceaccount.com"
      ]
    }
  ]
}
```

#### 4. Load Balancing
```
GCP Load Balancer Types:

Global Load Balancers:
- HTTP(S) Load Balancer (Layer 7)
- SSL Proxy Load Balancer (Layer 4)
- TCP Proxy Load Balancer (Layer 4)

Regional Load Balancers:
- Network Load Balancer (Layer 4)
- Internal Load Balancer (Layer 4)

Features:
- Anycast IP addresses
- Auto-scaling backend services
- Health checks
- CDN integration
```

---

## Microsoft Azure Architecture

### Core Azure Components Overview

```
Azure Front Door / Application Gateway
    ↓
Virtual Network (VNet)
    ↓
Resource Groups
    ↓
Subnets
    ↓
Network Security Groups (NSGs)
    ↓
Virtual Machines / Services
```

### Complete Azure Architecture Diagram

```
                    Internet
                       ↓
              [Azure DNS]
                       ↓
              [Azure CDN]
                       ↓
              [Azure Front Door (Global)]
                       ↓
              [Azure DDoS Protection]
                       ↓
              [Application Gateway (Regional)]
                       ↓
    ┌─────────────────────────────────────────────────┐
    │         Resource Group: Production              │
    │                                                 │
    │  ┌─────────────────────────────────────────────┐│
    │  │        Virtual Network (VNet)               ││
    │  │        Address Space: 10.0.0.0/16          ││
    │  │                                             ││
    │  │ [Service Endpoints] ←→ [Azure Services]     ││
    │  │ [Private Endpoints] ←→ [PaaS Services]      ││
    │  │                                             ││
    │  │ ┌─────────────────┐  ┌─────────────────┐   ││
    │  │ │Subnet: Web      │  │Subnet: App      │   ││
    │  │ │10.0.1.0/24      │  │10.0.2.0/24      │   ││
    │  │ │                 │  │                 │   ││
    │  │ │[VM Scale Set]   │  │[VM Scale Set]   │   ││
    │  │ │[App Service]    │  │[Function App]   │   ││
    │  │ │[Azure Firewall]│  │[NAT Gateway]    │   ││
    │  │ └─────────────────┘  └─────────────────┘   ││
    │  │                                             ││
    │  │ ┌─────────────────┐  ┌─────────────────┐   ││
    │  │ │Subnet: Data     │  │Subnet: Mgmt     │   ││
    │  │ │10.0.3.0/24      │  │10.0.4.0/24      │   ││
    │  │ │                 │  │                 │   ││
    │  │ │[Azure SQL]      │  │[Bastion Host]   │   ││
    │  │ │[Cosmos DB]      │  │[Key Vault]      │   ││
    │  │ │[Storage Account]│  │[Log Analytics]  │   ││
    │  │ └─────────────────┘  └─────────────────┘   ││
    │  │                                             ││
    │  │ [VNet Peering] ←→ [Other VNets]             ││
    │  │ [VPN Gateway/ExpressRoute] ←→ [On-Premises] ││
    │  │ [NSG Flow Logs] → [Network Watcher]         ││
    │  └─────────────────────────────────────────────┘│
    └─────────────────────────────────────────────────┘
                       ↓
              [Network Security Groups (NSGs)]
                       ↓
              [Application Security Groups (ASGs)]
                       ↓
              [Azure Active Directory]
                       ↓
              [Azure Security Center]
```

### Azure Request Flow Example

**Web Application Request Flow:**
```
1. User Request
   ↓
2. Azure DNS → Resolves domain to Front Door IP
   ↓
3. Azure Front Door (Global CDN + Load Balancer)
   ↓
4. Application Gateway (Regional)
   ↓
5. Network Security Group Check → Allow HTTP/HTTPS
   ↓
6. Virtual Network Routing
   ↓
7. App Service / Virtual Machine
   ↓
8. Managed Identity Check → Verify permissions
   ↓
9. Azure Functions → Business logic
   ↓
10. Azure SQL Database / Cosmos DB
    ↓
11. Response back through same path
```

### Azure Core Components Explained

#### 1. Resource Groups & Virtual Networks
```
Resource Group: Logical container for resources
- Name: rg-production-eastus
- Location: East US
- Resources: VNet, VMs, Storage, Databases

Virtual Network (VNet):
- Address Space: 10.0.0.0/16
- Location: East US (regional)
- Subnets: Multiple subnets within VNet
- Peering: Connect to other VNets
```

#### 2. Network Security Groups (NSGs)
```json
// Example NSG Rules
{
  "securityRules": [
    {
      "name": "AllowHTTP",
      "priority": 100,
      "direction": "Inbound",
      "access": "Allow",
      "protocol": "Tcp",
      "sourcePortRange": "*",
      "destinationPortRange": "80",
      "sourceAddressPrefix": "*",
      "destinationAddressPrefix": "*"
    },
    {
      "name": "AllowHTTPS", 
      "priority": 110,
      "direction": "Inbound",
      "access": "Allow",
      "protocol": "Tcp",
      "sourcePortRange": "*",
      "destinationPortRange": "443",
      "sourceAddressPrefix": "*",
      "destinationAddressPrefix": "*"
    },
    {
      "name": "DenyAllInbound",
      "priority": 4096,
      "direction": "Inbound", 
      "access": "Deny",
      "protocol": "*",
      "sourcePortRange": "*",
      "destinationPortRange": "*",
      "sourceAddressPrefix": "*",
      "destinationAddressPrefix": "*"
    }
  ]
}
```

#### 3. Azure Active Directory (AAD) & Managed Identity
```json
// Managed Identity for App Service
{
  "type": "SystemAssigned",
  "principalId": "12345678-1234-1234-1234-123456789012",
  "tenantId": "87654321-4321-4321-4321-210987654321"
}

// Role Assignment
{
  "roleDefinitionId": "/subscriptions/{subscription-id}/providers/Microsoft.Authorization/roleDefinitions/ba92f5b4-2d11-453d-a403-e96b0029c9fe",
  "principalId": "12345678-1234-1234-1234-123456789012",
  "scope": "/subscriptions/{subscription-id}/resourceGroups/rg-production/providers/Microsoft.Storage/storageAccounts/mystorageaccount"
}
```

#### 4. Application Gateway & Front Door
```
Application Gateway (Regional):
- Layer 7 load balancer
- SSL termination
- Web Application Firewall (WAF)
- URL-based routing
- Cookie-based session affinity

Azure Front Door (Global):
- Global load balancer
- CDN capabilities
- DDoS protection
- SSL offloading
- Health probes
```

---

## Cloud Architecture Comparison

### Networking Comparison (Complete)

| Component | AWS | Google Cloud | Azure |
|-----------|-----|--------------|-------|
| **Virtual Network** | VPC (Regional) | VPC (Global) | VNet (Regional) |
| **Subnets** | AZ-specific | Regional | VNet-specific |
| **Internet Access** | Internet Gateway | Automatic | Automatic |
| **Firewall** | Security Groups + NACLs | Firewall Rules | NSGs + ASGs |
| **Load Balancer** | ALB/NLB/CLB | Cloud Load Balancer | Application Gateway/Load Balancer |
| **DNS** | Route 53 | Cloud DNS | Azure DNS |
| **CDN** | CloudFront | Cloud CDN | Azure CDN |
| **WAF** | AWS WAF | Cloud Armor | Azure WAF |
| **DDoS Protection** | AWS Shield | Cloud Armor | Azure DDoS Protection |
| **Private Connectivity** | VPC Endpoints | Private Google Access | Service/Private Endpoints |
| **Hybrid Connectivity** | Direct Connect + VPN | Cloud Interconnect + VPN | ExpressRoute + VPN Gateway |
| **Network Monitoring** | VPC Flow Logs | VPC Flow Logs | NSG Flow Logs + Network Watcher |
| **Multi-VPC/VNet** | VPC Peering + Transit Gateway | VPC Peering | VNet Peering + Virtual WAN |
| **NAT Service** | NAT Gateway/Instance | Cloud NAT | NAT Gateway |
| **Bastion Service** | EC2 Bastion Host | Identity-Aware Proxy | Azure Bastion |

### Identity & Access Management

| Feature | AWS IAM | Google Cloud IAM | Azure AAD |
|---------|---------|------------------|-----------|
| **Users** | IAM Users | Google Accounts | Azure AD Users |
| **Service Identity** | IAM Roles | Service Accounts | Managed Identity |
| **Permissions** | Policies (JSON) | Roles & Bindings | RBAC |
| **MFA** | Yes | Yes | Yes |
| **Federation** | SAML/OIDC | SAML/OIDC | SAML/OIDC |

### Compute Services

| Service Type | AWS | Google Cloud | Azure |
|--------------|-----|--------------|-------|
| **Virtual Machines** | EC2 | Compute Engine | Virtual Machines |
| **Containers** | ECS/EKS | GKE | AKS |
| **Serverless** | Lambda | Cloud Functions | Azure Functions |
| **App Platform** | Elastic Beanstalk | App Engine | App Service |

### Database Services

| Database Type | AWS | Google Cloud | Azure |
|---------------|-----|--------------|-------|
| **Relational** | RDS | Cloud SQL | Azure SQL Database |
| **NoSQL** | DynamoDB | Firestore/Bigtable | Cosmos DB |
| **Data Warehouse** | Redshift | BigQuery | Synapse Analytics |
| **Cache** | ElastiCache | Memorystore | Azure Cache for Redis |

## Best Practices for Each Cloud

### AWS Best Practices
```
1. Use multiple AZs for high availability
2. Implement least privilege IAM policies
3. Use VPC endpoints for AWS services
4. Enable CloudTrail for auditing
5. Use Auto Scaling Groups for elasticity
6. Implement proper tagging strategy
7. Use AWS Config for compliance
```

### Google Cloud Best Practices
```
1. Use regional persistent disks for HA
2. Implement service accounts with minimal permissions
3. Use Private Google Access for internal communication
4. Enable Cloud Audit Logs
5. Use managed instance groups for scaling
6. Implement proper labeling strategy
7. Use Security Command Center for monitoring
```

### Azure Best Practices
```
1. Use availability sets/zones for HA
2. Implement managed identities
3. Use service endpoints for Azure services
4. Enable Azure Monitor and Log Analytics
5. Use VM Scale Sets for elasticity
6. Implement proper resource tagging
7. Use Azure Security Center for compliance
```

## Migration Considerations

### Multi-Cloud Strategy
```
Reasons for Multi-Cloud:
- Avoid vendor lock-in
- Best-of-breed services
- Geographic requirements
- Cost optimization
- Risk mitigation

Challenges:
- Increased complexity
- Skills requirements
- Data transfer costs
- Security consistency
- Management overhead
```

### Cloud Selection Criteria
```
Technical Factors:
- Service availability
- Performance requirements
- Integration capabilities
- Compliance requirements

Business Factors:
- Cost structure
- Existing relationships
- Skills availability
- Support quality
- Future roadmap
```

This comprehensive guide provides the foundation for understanding how each major cloud provider structures their services and how data flows through their architectures. Each cloud has its strengths and is suitable for different use cases and organizational requirements.
