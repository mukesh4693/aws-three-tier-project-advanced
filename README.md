# AWS Three-Tier Web Application

## Project Overview

This project demonstrates the deployment of a highly available three-tier web application on AWS. It uses Amazon VPC, EC2, Application Load Balancer, Auto Scaling, and Amazon RDS MySQL.

The architecture separates the presentation, application, and database layers to improve security, scalability, and maintainability.

## Architecture

**Architecture components:**
- Amazon VPC
- Public and private subnets across two Availability Zones
- Internet Gateway
- NAT Gateway
- Amazon EC2
- Application Load Balancer (ALB)
- EC2 Auto Scaling Group
- Amazon RDS MySQL
- Security Groups
- Route 53 (optional)

## Architecture Flow

1. Users access the application through the ALB DNS name or a domain configured in Route 53.
2. The Application Load Balancer receives HTTP requests on port 80.
3. The ALB forwards requests to healthy EC2 instances registered in the target group.
4. The EC2 instances run the Flask application on port 5000.
5. Flask connects to Amazon RDS MySQL on port 3306.
6. The database returns the requested records to Flask.
7. Flask returns the response to the user through the ALB.


## Deployment Steps

### 1. Create the VPC

1. Open the **Amazon VPC** console.
2. Choose **Your VPCs** → **Create VPC**.
3. Select **VPC only**.
4. Enter a name such as `three-tier-vpc`.
5. Set the IPv4 CIDR to `10.0.0.0/16` (or your chosen non-overlapping range).
6. Create the VPC.
7. Create or attach an Internet Gateway, then attach it to the VPC.

### 2. Create the Six Subnets

1. Open **Subnets** in the VPC console.
2. Create the two public web subnets, one in each AZ.
3. Create the two private app subnets, one in each AZ.
4. Create the two private database subnets, one in each AZ.
5. Use the CIDR plan in the [Network Design](#network-design) table, adjusting it if needed.
6. Enable auto-assign public IPv4 addresses only for the public subnets if your design requires it. Private app and DB subnets should not auto-assign public IPs.

### 3. Create and Associate Route Tables

1. Create `public-rt`, add the default route to the Internet Gateway, and associate both web subnets.
2. Create `app-rt-1` and `app-rt-2`.
3. Create one NAT Gateway in each public AZ and allocate an Elastic IP to each.
4. Add a default route in each app route table to the NAT Gateway in the same AZ.
5. Associate each app route table with its corresponding app subnet.
6. Create `db-rt` with only the local VPC route and associate both DB subnets.
7. Verify all subnet associations and routes.

### 4. Create Security Groups

1. Open **EC2** → **Security Groups**.
2. Create the five security groups listed in [Security Groups](#security-groups), selecting the project VPC.
3. Configure inbound rules using the tables above.
4. Attach each group to the correct load balancer, EC2 instance, or RDS database.
5. Verify the source security group references and listening ports before testing.

### 5. Configure Route 53

1. Open **Amazon Route 53**.
2. Create a **Public hosted zone** for a domain you own.
3. Copy the hosted zone's assigned name servers.
4. Open the domain registrar's DNS/name-server settings.
5. Replace the existing name servers with the Route 53 name servers.
6. Wait for the registrar changes and DNS delegation to propagate.
7. Later, create an alias record that points your application hostname to the Web ALB.

You must own or control the domain to update its name-server delegation.

### 6. Request and Validate an ACM Certificate

1. Open **AWS Certificate Manager (ACM)** in the same AWS Region as the ALB.
2. Choose **Request a certificate** → **Request a public certificate**.
3. Enter the domain name, for example `app.example.com`. Add any required subject alternative names.
4. Choose **DNS validation**.
5. Request the certificate.
6. Use the CNAME record ACM provides. If the hosted zone is in Route 53, create the suggested record there.
7. Wait until the certificate status is **Issued**.
8. Attach the certificate to the ALB HTTPS listener.

For an ALB, the ACM certificate must be in the same Region as the load balancer. If you also use HTTPS on an internal ALB, configure its listener and certificate separately as required.

### 7. Create the RDS MySQL Database

#### Create a DB subnet group

1. Open **Amazon RDS** → **Subnet groups**.
2. Create a DB subnet group and select the project VPC.
3. Add `db-subnet-1` and `db-subnet-2`, which must be in different AZs.
4. Save the subnet group.

## 1. VPC Configuration

| Setting | Value |
|---|---|
| VPC Name | ThreeTier-VPC |
| CIDR Block | 10.0.0.0/16 |
| Availability Zones | AZ-1 and AZ-2 |
| Public Subnets | Two |
| Private Application Subnets | Two |
| Private Database Subnets | Two |
| Internet Gateway | ThreeTier-IGW |
| NAT Gateway | ThreeTier-NAT |

### Subnet Design

| Subnet | CIDR | Purpose |
|---|---|---|
| Public-A | 10.0.1.0/24 | ALB and public resources |
| Public-B | 10.0.2.0/24 | ALB and public resources |
| App-A | 10.0.3.0/24 | Private application EC2 |
| App-B | 10.0.4.0/24 | Private application EC2 |
| DB-A | 10.0.5.0/24 | RDS subnet group |
| DB-B | 10.0.6.0/24 | RDS subnet group |

### Route tables

#### Public route table

Create `public-rt` and associate it with both web subnets.

| Destination | Target |
|---|---|
| VPC CIDR (`10.0.0.0/16`) | `local` (created automatically) |
| `0.0.0.0/0` | Internet Gateway |

#### Private application route tables

Use a separate route table for each app subnet. This design uses one NAT Gateway in each AZ.

| Route table | Subnet association | Destination | Target |
|---|---|---|---|
| `app-rt-1` | `app-subnet-1` | `0.0.0.0/0` | NAT Gateway 1 |
| `app-rt-2` | `app-subnet-2` | `0.0.0.0/0` | NAT Gateway 2 |

Each private route table also has the automatically created local VPC route.

#### Database route table

Create a database route table and associate it with both DB subnets.

| Destination | Target |
|---|---|
| VPC CIDR (`10.0.0.0/16`) | `local` |

RDS does not normally need a NAT Gateway for application database traffic. Add outbound routes only if a documented requirement calls for them; RDS maintenance is managed by the service.

---

## Security Groups

Create five security groups. Use security-group references as sources wherever possible instead of opening traffic to broad CIDR ranges.

> **Important:** The rules below describe the intended traffic path. For a production deployment, avoid public SSH access. Prefer AWS Systems Manager Session Manager or a tightly restricted administrative access path.

### 1. `WebServer-SG`

Attach to the web EC2 instances.

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| SSH | TCP | 22 | Restricted administrator IP or management path | Administration, if needed |
| HTTP | TCP | 80 | `Web-ALB-SG` | Web traffic from the ALB |
| HTTPS | TCP | 443 | `Web-ALB-SG` | HTTPS traffic from the ALB, if used on instances |

If users connect through the Web ALB, do not open ports 80/443 on the web instances to the entire internet. If SSH is required, restrict it to a known administrator IP or use Session Manager.

### 2. `Web-ALB-SG`

Attach to the internet-facing Web ALB.

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| HTTP | TCP | 80 | `0.0.0.0/0` | Public HTTP listener / redirect |
| HTTPS | TCP | 443 | `0.0.0.0/0` | Public HTTPS listener |

For IPv6-enabled public access, add the appropriate IPv6 source range as well. Configure the HTTPS listener with the ACM certificate.

### 3. `AppServer-SG`

Attach to the Flask application EC2 instances.

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| Custom TCP | TCP | 5000 | `App-ALB-SG` | Flask API traffic from the internal ALB |
| SSH | TCP | 22 | Restricted management source | Administration, if required |

Do not open the Flask port to the internet. The app instances should receive application traffic only from the internal App ALB. If you administer the app instances through the web servers, use a carefully restricted management path; Session Manager is preferable.

### 4. `App-ALB-SG`

Attach to the internal App ALB.

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| Custom TCP | TCP | 5000 | `WebServer-SG` | Requests from the web tier |
| HTTPS | TCP | 443 | `WebServer-SG` | Only if the internal ALB uses HTTPS |

An internal ALB should be configured as **internal**, not internet-facing. Allow only the traffic needed by the web tier.

### 5. `DB-SG`

Attach to the RDS database.

| Type | Protocol | Port | Source | Purpose |
|---|---|---:|---|---|
| MySQL/Aurora | TCP | 3306 | `AppServer-SG` | MySQL connections from app instances |

Set RDS **Public access** to **No**. Do not allow MySQL port 3306 from `0.0.0.0/0`.

### Security group traffic flow

```text
Internet
   |
   v
Web-ALB-SG (80/443)
   |
   v
WebServer-SG (web listener port)
   |
   v
App-ALB-SG (5000, or configured HTTPS)
   |
   v
AppServer-SG (5000)
   |
   v
DB-SG (3306)
   |
   v
RDS MySQL
```

Security groups are stateful. Ensure the inbound rules on each destination allow the required source and port. Also confirm that network ACLs and application listeners do not block the traffic.

## 2. Target Group Configuration

| Setting | Value |
|---|---|
| Target Group Name | ThreeTier-TG |
| Target Type | Instances |
| Protocol | HTTP |
| Port | 5000 |
| VPC | ThreeTier-VPC |
| Protocol Version | HTTP1 |
| Health Check Protocol | HTTP |
| Health Check Path | /health |

The target group distributes incoming requests to registered EC2 instances and monitors their health.

### Flask Health Check

```python
@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok"})
```

## 3. Application Load Balancer

| Setting | Value |
|---|---|
| Name | ThreeTier-ALB |
| Scheme | Internet-facing |
| IP Address Type | IPv4 |
| VPC | ThreeTier-VPC |
| Availability Zones | AZ-1 and AZ-2 |
| Subnets | Both public subnets |
| Security Group | ThreeTier-ALB-SG |

### Listener Configuration

| Setting | Value |
|---|---|
| Protocol | HTTP |
| Port | 80 |
| Default Action | Forward to ThreeTier-TG |

The ALB distributes incoming HTTP traffic between healthy application instances.

## 4. Amazon RDS MySQL Configuration

### Database Configuration

| Setting | Value |
|---|---|
| Engine | MySQL |
| Version | Available supported version |
| Template | Free tier, if available |
| DB Instance Identifier | ThreeTier-MySQL |
| Master Username | admin |
| Master Password | Strong password |
| DB Instance Class | Smallest eligible class |
| Availability | Single-AZ |
| Storage | Minimum permitted |
| Storage Autoscaling | Disabled for cost-controlled lab |

### Database Connectivity

| Setting | Value |
|---|---|
| VPC | ThreeTier-VPC |
| DB Subnet Group | ThreeTier-DB-Subnet-Group |
| Subnets | ThreeTier-DB-A and ThreeTier-DB-B |
| Public Access | No |
| Security Group | ThreeTier-DB-SG |
| Port | 3306 |

The database is deployed in private subnets and is not directly accessible from the internet.

## 5. EC2 Launch Template

| Setting | Value |
|---|---|
| AMI | Ubuntu Server LTS |
| Instance Type | Smallest eligible type |
| Key Pair | As required for SSH access |
| Security Group | ThreeTier-APP-SG |
| IAM Instance Profile | ThreeTier-EC2-Role |
| Network Interface | Selected by Auto Scaling |
| Auto-assign Public IP | Disabled |

The launch template defines the configuration used to launch application EC2 instances.

## 6. Auto Scaling Group

### Configuration

| Setting | Value |
|---|---|
| Name | ThreeTier-ASG |
| VPC | ThreeTier-VPC |
| Subnets | ThreeTier-App-A and ThreeTier-App-B |
| Launch Template | ThreeTier-Launch-Template |
| Desired Capacity | 2 |
| Minimum Capacity | 2 |
| Maximum Capacity | 4 |

### Scaling Policy

- Policy Type: Target tracking
- Metric: Average CPU utilization
- Target Value: 60%
- Instance Warm-up: 180 seconds

### Load Balancing

1. Attach to an existing load balancer.
2. Select the existing target group.
3. Choose ThreeTier-TG.
4. Enable Elastic Load Balancing health checks.

The Auto Scaling Group maintains the configured capacity and can launch or terminate instances based on scaling policies.

## 7. Security Group Configuration

| Security Group | Protocol | Port | Source |
|---|---|---|---|
| ThreeTier-ALB-SG | HTTP | 80 | 0.0.0.0/0 |
| ThreeTier-APP-SG | HTTP | 5000 | ThreeTier-ALB-SG |
| ThreeTier-DB-SG | MySQL | 3306 | ThreeTier-APP-SG |
| ThreeTier-APP-SG | SSH | 22 | Your IP, if required |

Security groups restrict communication between the application layers. SSH access should be restricted to trusted addresses.

## 8. Route 53 (Optional)

Amazon Route 53 can be used to connect a custom domain to the Application Load Balancer.

1. Register or use an existing domain.
2. Create a hosted zone.
3. Create an Alias A record.
4. Select the Application Load Balancer as the alias target.
5. Access the application through the configured domain.

For HTTPS, configure an SSL/TLS certificate using AWS Certificate Manager and an HTTPS listener on the ALB.

## 9. Testing the Architecture

### Test 1: Auto Scaling Group

1. Navigate to EC2 → Auto Scaling Groups.
2. Select ThreeTier-ASG.
3. Verify that the desired capacity is two.
4. Confirm that instances are running in both application subnets.

### Test 2: Target Group Health

1. Navigate to EC2 → Target Groups.
2. Select ThreeTier-TG.
3. Open the Targets tab.
4. Verify that the registered EC2 instances have Healthy status.

### Test 3: Application Load Balancer

1. Navigate to EC2 → Load Balancers.
2. Select ThreeTier-ALB.
3. Copy the DNS name.
4. Open the DNS name in a browser using HTTP.

Example:

```text
http://YOUR-ALB-DNS
```

### Test 4: Database Connectivity

Open the student API endpoint through the ALB.

```text
http://YOUR-ALB-DNS/api/students
```

The API should return the student records retrieved from Amazon RDS.

Example response:

```json
{
  "success": true,
  "count": 2,
  "students": [
    {
      "id": 1,
      "username": "Arun",
      "email": "arun@example.com"
    },
    {
      "id": 2,
      "username": "Ravi",
      "email": "ravi@example.com"
    }
  ]
}
```

## 10. Key AWS Services Used

| AWS Service | Purpose |
|---|---|
| Amazon VPC | Isolated network for the application |
| Public Subnets | Host internet-facing resources |
| Private Subnets | Isolate application and database resources |
| Internet Gateway | Internet connectivity for public resources |
| NAT Gateway | Outbound internet access for private resources |
| Amazon EC2 | Runs the application |
| Application Load Balancer | Distributes incoming traffic |
| Target Group | Registers and monitors application instances |
| Auto Scaling | Automatically adjusts instance capacity |
| Amazon RDS | Managed MySQL database |
| Security Groups | Control network traffic |
| Route 53 | Domain name resolution |
| IAM | Secure AWS permissions |

## 11. Project Benefits

- Three-tier architecture with separated application layers.
- Load balancing across multiple application instances.
- Automatic scaling based on CPU utilization.
- Private database access.
- Health checks to detect unhealthy application instances.
- Network-level access control using security groups.
- DNS integration through Route 53.
- Infrastructure designed for availability across two Availability Zones.

## 12. Future Improvements

- Add HTTPS using AWS Certificate Manager.
- Add AWS WAF to protect the application.
- Configure CloudWatch alarms and SNS notifications.
- Automate deployments using Jenkins.
- Use Terraform for infrastructure provisioning.
- Store application secrets in AWS Secrets Manager.
- Configure automated database backups.

## Conclusion

This project demonstrates how to deploy a three-tier web application on AWS using EC2, Application Load Balancer, Auto Scaling, and Amazon RDS. It provides hands-on experience with AWS networking, application deployment, database connectivity, security, and scalability.

**Note:** Instance availability, free-tier eligibility, and AWS pricing depend on the selected region and current AWS account terms.
![Uploading image.png…]()
