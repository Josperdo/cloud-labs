# AWS Lab 1: VPC, Security Groups & IAM Fundamentals

Check box if done: []

## Overview
This lab mirrors Azure Lab 1 (networking) but on AWS. You'll build a VPC with subnets, configure security groups, and set up IAM roles/policies. This is foundational—everything in AWS runs *inside* a VPC.

**Estimated time**: 3–4 hours  
**Cost**: ~$0.50 (mostly free tier; tear down at the end)  
**Key difference from Azure**: AWS uses Security Groups (stateful firewall at instance level) instead of NSGs (can be at subnet or NIC level). Both do similar things, different mental models.

---

## Objectives
- Create VPC with public/private subnet architecture
- Configure Security Groups with inbound/outbound rules
- Understand VPC route tables and Internet Gateway
- Create IAM users, groups, and custom policies
- Connect concepts back to Azure equivalents

---

## Part 1: Create VPC and Subnets

### Step 1: Create VPC
1. Go to **AWS Console** → Search **VPC** → Click **VPCs** (left sidebar)
2. Click **Create VPC**
3. **VPC settings:**
   - **Name tag**: `lab1-vpc`
   - **IPv4 CIDR block**: `10.0.0.0/16`
   - Leave "IPv6 CIDR block" unchecked
   - **Tenancy**: `Default`
4. Click **Create VPC**

**Why 10.0.0.0/16?** Same as your Azure lab. Consistency helps you compare.

### Step 2: Create Internet Gateway
1. Still in VPC → Left sidebar → **Internet gateways** → **Create internet gateway**
2. **Name tag**: `lab1-igw`
3. Click **Create internet gateway**
4. After creation, you'll see a button **Attach to VPC**
5. Click it → Select `lab1-vpc` → Click **Attach internet gateway**

**What this does**: Internet Gateway = your entry point to the internet (equivalent to Azure's Public IP + routing).

### Step 3: Create Public Subnet
1. Left sidebar → **Subnets** → **Create subnet**
2. **Subnet settings:**
   - **VPC ID**: `lab1-vpc`
   - **Subnet name**: `public-subnet-1`
   - **Availability Zone**: `us-east-1a` (pick one; doesn't matter for lab)
   - **IPv4 CIDR block**: `10.0.1.0/24`
3. Click **Create subnet**

### Step 4: Create Private Subnet
1. Repeat Step 3:
   - **Subnet name**: `private-subnet-1`
   - **IPv4 CIDR block**: `10.0.2.0/24`
   - Same AZ as public subnet

**Why two subnets?** Public = webservers (accessible from internet). Private = databases (not accessible directly from internet). Same pattern as Azure Lab 1.

### Step 5: Configure Route Table for Public Subnet
1. Left sidebar → **Route tables** → Click the route table associated with your VPC
   - (You'll see one created automatically; it should say "Main" or have your VPC ID)
2. Click on it
3. Bottom section → **Routes** tab → **Edit routes**
4. Click **Add route**
   - **Destination**: `0.0.0.0/0` (all internet traffic)
   - **Target**: Select **Internet Gateway** → `lab1-igw`
5. Click **Save changes**

**What this does**: "Any traffic destined for the internet (0.0.0.0/0) goes through the Internet Gateway." This makes the route table suitable for public subnets.

### Step 6: Associate Route Table with Public Subnet
1. Still in the route table → **Subnet associations** tab
2. Click **Edit subnet associations**
3. Check `public-subnet-1`
4. Click **Save associations**

**Validation**: Public subnet is now associated with a route table that has an Internet Gateway route. Instances in this subnet can reach the internet.

### Step 7: Create Private Route Table (for database subnet)
1. **Route tables** → **Create route table**
2. **Name**: `private-rt`
3. **VPC**: `lab1-vpc`
4. Click **Create route table**
5. After creation, associate it with `private-subnet-1`:
   - Go to **Subnet associations** → **Edit subnet associations** → Check `private-subnet-1` → **Save**

**Don't add an Internet Gateway route to this one.** Private subnet = no direct internet access (database stays protected).

---

## Part 2: Create and Configure Security Groups

### Step 8: Create Security Group for Web Tier
1. Left sidebar → **Security groups** → **Create security group**
2. **Basic details:**
   - **Security group name**: `web-sg`
   - **Description**: `Security group for web servers`
   - **VPC**: `lab1-vpc`
3. Click **Create security group**

### Step 9: Add Inbound Rules to Web SG
1. After creation, you're in the security group details
2. **Inbound rules** tab → **Edit inbound rules**
3. Click **Add rule**
4. **Rule #1 (HTTP)**:
   - **Type**: `HTTP`
   - **Protocol**: `TCP`
   - **Port range**: `80`
   - **Source**: `0.0.0.0/0` (anyone on the internet)
5. Click **Add rule** again
6. **Rule #2 (HTTPS)**:
   - **Type**: `HTTPS`
   - **Protocol**: `TCP`
   - **Port range**: `443`
   - **Source**: `0.0.0.0/0`
7. Click **Add rule** again
8. **Rule #3 (SSH)**:
   - **Type**: `SSH`
   - **Protocol**: `TCP`
   - **Port range**: `22`
   - **Source**: `MY_IP` (AWS auto-fills your current IP) or manually enter your IP/32
9. Click **Save rules**

**Key insight**: Security Groups are *stateful*—if you allow inbound on port 80, responses are automatically allowed outbound. Outbound defaults to "allow all."

### Step 10: Create Security Group for Database Tier
1. **Create security group** again:
   - **Name**: `db-sg`
   - **VPC**: `lab1-vpc`
2. **Inbound rules**:
   - Add ONE rule: Type: `MySQL/Aurora`, Port: `3306`, Source: `web-sg` (select the security group, not a CIDR)
   - This means: "Only traffic from web-sg can access the database"
3. Click **Save rules**

**Why reference web-sg instead of a CIDR?** Because security groups can reference each other. This is powerful: "only instances in web-sg can access the database."

---

## Part 3: Understand VPC vs Subnet vs Security Group

### Step 11: Mental Model (Interview Question Practice)

**If someone asks: "What's the difference between a VPC and a subnet?"**

Your answer:
- **VPC** = Isolated network in AWS (like a region boundary). You define the IP space (10.0.0.0/16).
- **Subnet** = Divides the VPC into smaller IP ranges. Subnets are tied to Availability Zones. You can have a public subnet (with IGW route) or private subnet (no internet route).
- **Security Group** = Stateful firewall that protects instances. You attach it to an EC2 instance (or ENI).

**How does this compare to Azure?**
- Azure VNet ≈ AWS VPC
- Azure Subnet ≈ AWS Subnet (same concept)
- Azure NSG ≈ AWS Security Group (NSG can be at subnet level OR NIC level; SG is always at instance level)

---

## Part 4: Create IAM Users and Policies

### Step 12: Create IAM Users
1. Go to **IAM** → Left sidebar → **Users** → **Create user**
2. **Create User #1:**
   - **User name**: `prod-engineer`
   - **Console access**: Check **Provide user access to AWS Management Console**
   - **Console password**: `Autogenerated`
   - Uncheck **Require password change** (for lab purposes)
3. Click **Next**
4. **Set permissions**: 
   - Click **Attach policies directly**
   - Search `PowerUserAccess` → Check it
   - (PowerUser ≈ can do everything except manage IAM and billing)
5. Click **Create user**
6. Download the credentials CSV file (you'll need it to sign in as this user)

7. **Create User #2:**
   - **User name**: `db-admin`
   - **Console access**: Check
   - Follow same steps
   - Select policy: Search `RDSFullAccess` → check it
   - Click **Create user** and download credentials

8. **Create User #3:**
   - **User name**: `auditor`
   - **Console access**: Check
   - Select policy: Search `ReadOnlyAccess` → check it
   - Click **Create user** and download credentials

**What you've set up**:
- **prod-engineer**: Can do anything except IAM/billing (PowerUser)
- **db-admin**: Can only manage RDS (Database)
- **auditor**: Read-only (can view everything, change nothing)

### Step 13: Create IAM Group and Add Users
1. Left sidebar → **User groups** → **Create group**
2. **Group name**: `DevTeam`
3. **Add permissions**: Attach policy `AdministratorAccess` (for lab; in production, use least privilege)
4. Click **Create group**
5. After creation, click the group → **Add users** → Select `prod-engineer` and `db-admin` → **Add users**

**Validation**: DevTeam group now has 2 users.

### Step 14: Create Custom IAM Policy (Interview Exercise)
1. Left sidebar → **Policies** → **Create policy**
2. Click **JSON** tab
3. Paste this policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "s3:GetObject",
           "s3:ListBucket"
         ],
         "Resource": [
           "arn:aws:s3:::lab-bucket",
           "arn:aws:s3:::lab-bucket/*"
         ]
       }
     ]
   }
   ```
4. Click **Next**
5. **Policy name**: `S3ReadOnlyPolicy`
6. Click **Create policy**

**What this does**: This policy says "Allow reading from the S3 bucket called 'lab-bucket' only." It's an example of least privilege.

**Interview question this preps you for**: "How would you create a policy that allows a user to read S3 but not write or delete?"

### Step 15: Attach Custom Policy to User
1. Go to **Users** → Click `auditor`
2. **Add permissions** → **Attach policies directly**
3. Search `S3ReadOnlyPolicy` → Check it → Click **Next** → **Attach policy**

**Now auditor can**: Read-only everywhere (ReadOnlyAccess) PLUS specifically read S3.

---

## Part 5: Test and Validate

### Step 16: Sign Out and Test User Access
1. Top right → Click your account name → **Sign Out**
2. On the login page, click **Sign in as an IAM user**
3. **Account ID**: Your AWS account ID (you can find it in your main account's IAM dashboard)
4. **User name**: `auditor`
5. **Password**: From the CSV file you downloaded
6. Click **Sign in**

**What happens**: You're now signed in as the auditor user. Try navigating:
- **Can you see EC2?** Yes (ReadOnlyAccess)
- **Can you create an EC2 instance?** No (read-only)
- **Can you see S3?** Yes (ReadOnlyAccess + S3ReadOnlyPolicy)

7. Sign out → Sign in as `prod-engineer`
   - **Can you create resources?** Yes (PowerUserAccess)
   - **Can you modify IAM?** No (PowerUserAccess excludes IAM)

**Key learning**: IAM policies control what users can do. Different users have different scopes of access.

---

## Part 6: Understanding AWS IAM vs Azure RBAC

### Step 17: Compare to Azure (Mental Model)

| Concept | AWS | Azure |
|---------|-----|-------|
| **Access control** | IAM policies (JSON statements) | RBAC (role + scope) |
| **User/Group** | IAM Users + Groups | Entra ID users + Groups |
| **Permissions** | Policies attached to users/roles/groups | Roles assigned at scope (Sub, RG, Resource) |
| **Scope** | Actions on resources (ARNs) | Scope-based (subscription, RG, resource) |
| **Managed vs Custom** | Managed policies + customer-managed policies | Built-in roles + custom roles |

**Key difference**: 
- **AWS**: "User gets Policy X, which allows Action Y on Resource Z"
- **Azure**: "User gets Role X at Scope Y, which allows them to do Z"

Both achieve least privilege. Different mental models.

---

## Part 7: Cleanup

1. **Delete IAM users**: 
   - Go to **IAM** → **Users** → Select each user → **Delete**

2. **Delete IAM group**:
   - **User groups** → Select `DevTeam` → **Delete**

3. **Delete custom policy**:
   - **Policies** → Search `S3ReadOnlyPolicy` → **Delete**

4. **Delete Security Groups**:
   - Go to **VPC** → **Security groups** → Select `web-sg` and `db-sg` → **Delete**
   - (You may need to delete EC2 instances first if you created any; for this lab, you didn't)

5. **Delete Subnets**:
   - **Subnets** → Select `public-subnet-1` and `private-subnet-1` → **Delete**

6. **Detach Internet Gateway**:
   - **Internet gateways** → Click `lab1-igw` → **Detach from VPC** → Confirm → **Delete internet gateway**

7. **Delete VPC**:
   - **VPCs** → Select `lab1-vpc` → **Delete VPC**
   - (This auto-deletes route tables)

---

## Key Concepts to Understand

| Concept | What It Does |
|---------|-------------|
| **VPC** | Isolated network in AWS; defines IP space and subnets |
| **Subnet** | Divides VPC into smaller IP ranges; tied to one Availability Zone |
| **Internet Gateway** | Entry/exit point for traffic to/from the internet |
| **Route Table** | Defines where traffic goes (e.g., 0.0.0.0/0 → IGW for internet) |
| **Security Group** | Stateful firewall at instance level; allows/denies traffic |
| **NACL** | Network ACL; stateless firewall at subnet level (advanced; not covered here) |
| **IAM User** | Individual person who accesses AWS |
| **IAM Group** | Collection of users with shared permissions |
| **IAM Policy** | JSON document defining allowed actions on resources |
| **IAM Role** | Identity that can be assumed by services (e.g., EC2 → S3) |

---

## Interview Prep: Common Questions

**Q: "What's the difference between a Security Group and a NACL?"**
A: Security Groups are stateful (allow inbound = auto-allow outbound response). NACLs are stateless (you define both inbound AND outbound). SGs are instance-level; NACLs are subnet-level. For most use cases, SGs are sufficient.

**Q: "How would you restrict database access to only web servers?"**
A: Create a Security Group for the database with an inbound rule that references the web server Security Group (by ID). This way, only traffic from web-sg can reach the database.

**Q: "What's the principle of least privilege in AWS IAM?"**
A: Only grant users the minimum permissions they need. Example: auditor gets ReadOnlyAccess (not full Admin). db-admin gets RDSFullAccess (not EC2 or S3 access).

**Q: "What's the difference between AWS IAM and Azure RBAC?"**
A: Both achieve access control but differ in model. AWS uses policies (JSON statements on resources). Azure uses role-scope pairs (role + where it applies). AWS is more granular; Azure is more straightforward.

---

## Next Steps
- Deploy an EC2 instance in the public subnet (Lab 2)
- Create an RDS database in the private subnet and access it from the public subnet
- Implement VPC peering (connect two VPCs)
- Set up a NAT Gateway to allow private subnet instances to reach the internet
