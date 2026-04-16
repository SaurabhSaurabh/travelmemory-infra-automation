# 🚀 TravelMemory Deployment (Terraform + Ansible)

📄 Source:
This project demonstrates end-to-end infrastructure provisioning and application deployment using:

Terraform → Infrastructure as Code (AWS)
Ansible → Configuration Management
MERN Stack App → TravelMemory

🔗 MERN App Used:
https://github.com/UnpredictablePrashant/TravelMemory

-----

# 📌 Project Overview

This project automates:

AWS infrastructure setup (VPC, EC2, networking)
Secure multi-tier architecture (public + private subnet)
Application deployment using Ansible
Bastion-based secure access to private resources
 
 # travelmemory-deployment-ansible-terraform
Project Statement
MERN Application to use: https://github.com/UnpredictablePrashant/TravelMemory    
Tasks:  
Part 1: Infrastructure Setup with Terraform
1. AWS Setup and Terraform Initialization:
   - Configure AWS CLI and authenticate with your AWS account.
   - Initialize a new Terraform project targeting AWS.
2. VPC and Network Configuration:
   - Create an AWS VPC with two subnets: one public and one private.
   - Set up an Internet Gateway and a NAT Gateway.
   - Configure route tables for both subnets.
3. EC2 Instance Provisioning:
   - Launch two EC2 instances: one in the public subnet (for the web server) and another in the private subnet (for the database).
   - Ensure both instances are accessible via SSH (public instance only accessible from your IP).
4. Security Groups and IAM Roles:
   - Create necessary security groups for web and database servers.
   - Set up IAM roles for EC2 instances with required permissions.
5. Resource Output:
   - Output the public IP of the web server EC2 instance.
  

# Project solution:
🏗️ Architecture

```
Internet
   ↓
[Internet Gateway]
   ↓
[Public Subnet]
   → EC2 (Frontend + Backend - Node.js MERN App)
   ↓ (via NAT Gateway)
[Private Subnet]
   → EC2 (MongoDB Database)

```

# ⚙️ Part 1: Infrastructure Setup (Terraform)

## 🔹 Step 1: AWS Setup

AWS Setup + Terraform Init  
 
```aws configure```
<img width="975" height="183" alt="image" src="https://github.com/user-attachments/assets/bd4ffcce-ccc3-46df-943b-a04280b492c9" />

📁 Terraform Project Structure

```
terraform-mern/
│── main.tf
│── variables.tf
│── outputs.tf
│── provider.tf
```

<img width="975" height="404" alt="image" src="https://github.com/user-attachments/assets/8b9f30fc-2d81-4f34-a3e6-1b74cb17618a" />   


Provider.tf   
<img width="975" height="323" alt="image" src="https://github.com/user-attachments/assets/f450f4ac-f2e7-4513-a8ca-946ea88566e8" />   



Step: main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "6.39.0"
    }
  }
}

### vpc
resource "aws_vpc" "mern_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "mern-vpc"
    name = "mern-vpc"
  }
}

### Public Subnet
resource "aws_subnet" "public_subnet" {
  vpc_id                  = aws_vpc.mern_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone = "ap-south-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "mern-public-subnet"
  }
  
}

### private subnet
resource "aws_subnet" "private_subnet" {
  vpc_id     = aws_vpc.mern_vpc.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "ap-south-1a"
   tags = {
    Name = "mern-private-subnet"
  }
}

### internet gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.mern_vpc.id
   tags = {
    Name = "mern-igw"
  }
}

### Route Table (Public)
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.mern_vpc.id
   tags = {
    name = "mern-public-rt"
  }
}

resource "aws_route" "public_route" {
  route_table_id         = aws_route_table.public_rt.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

### NAT Gateway (for private subnet)
resource "aws_eip" "nat_eip" {
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_subnet.id
   tags = {
    Name = "mern-natgw"
  }
}

### Private Route Table
resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.mern_vpc.id
   tags = {
    name = "mern-private-rt"
  }
}

resource "aws_route" "private_route" {
  route_table_id         = aws_route_table.private_rt.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat.id
}

resource "aws_route_table_association" "private_assoc" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.private_rt.id
}


### Security groups
#### frontend-sg creation
resource "aws_security_group" "web_sg" {
  vpc_id = aws_vpc.mern_vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

   tags = {
    name = "mern-frontend-sg"
  }
}

#### database sg creation:
resource "aws_security_group" "db_sg" {
  vpc_id = aws_vpc.mern_vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 3001
    to_port     = 3001
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    name = "mern-backend-sg"
  }
}

### ec2 instance creation:
#### frontend ec2 instance
resource "aws_instance" "frontend" {
  ami           = "ami-05d2d839d4f73aafb" # Amazon Linux
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_subnet.id
  key_name      = "mern-key"

  associate_public_ip_address = true

  vpc_security_group_ids = [aws_security_group.web_sg.id]
  tags = {
    name = "mern-frontend"
  }
}

#### backend ec2 instance creation
resource "aws_instance" "backend" {
  ami           = "ami-05d2d839d4f73aafb"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.private_subnet.id
  key_name      = "mern-key"

  vpc_security_group_ids = [aws_security_group.db_sg.id]
  tags = {
    name = "mern-backend"
  }
}

### iam role
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Effect = "Allow",
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

Outputs.tf
```
output "frontend_public_ip" {
  value = aws_instance.frontend.public_ip
}
output "backend_public_ip" {
  value = aws_instance.backend.public_ip

}
```

Folder structure
<img width="880" height="432" alt="image" src="https://github.com/user-attachments/assets/e477e73f-c544-48ca-bbf6-95d2011633c1" />

## 🔹 Key Resources Created
   🌐 Networking
   VPC → 10.0.0.0/16
   Public Subnet → 10.0.1.0/24
   Private Subnet → 10.0.2.0/24
   Internet Gateway
   NAT Gateway
## 🔐 Security
   Web Security Group (ports: 22, 80, 443, 3000)
   DB Security Group (ports: 22, 3001)
   💻 Compute
   Public EC2 → Frontend + Backend
   Private EC2 → MongoDB

## 🔹 Terraform Commands
```
terraform init
terraform plan
terraform apply
```
## 📤 Outputs
```
output "frontend_public_ip"  
output "backend_public_ip"  
```

# ⚙️ Part 2: Configuration Management (Ansible)
## 🖥️ Setup (WSL / Ubuntu)

```
1.	wsl --install -d Ubuntu
2.	sudo apt update
3.	sudo apt install ansible -y
4.	ansible –version
```
   
<img width="975" height="610" alt="image" src="https://github.com/user-attachments/assets/3436dbba-5ae0-464a-8a95-48a301eaaee1" />

<img width="975" height="585" alt="image" src="https://github.com/user-attachments/assets/d02c7225-26d6-4e46-8306-fba7519d9c7a" />

<img width="975" height="190" alt="image" src="https://github.com/user-attachments/assets/064f7b02-cee8-47b8-9a0a-c5cbefb24779" />


## 📁 Inventory File (hosts.ini)
### Creating hosts.ini
#### [web]
   ```bastion ansible_host=13.233.92.171 ansible_user=ubuntu ansible_ssh_private_key_file=~/mern-key.pem```

##### [db]
``` private1 ansible_host=10.0.2.158 ansible_user=ubuntu ansible_ssh_private_key_file=~/mern-key.pem ansible_ssh_common_args='-o ProxyCommand="ssh -i ~/mern-key.pem ubuntu@13.233.92.171 -W %h:%p"' ```

Creating db.yml for database mongodb configuration on aws instance in private subnet
- name: Setup MongoDB Server
  hosts: db
  become: yes

  vars:
    mongo_admin_user: admin
    mongo_admin_password: password123   # ⚠️ change in real projects

  tasks:

    - name: Update apt cache
      apt:
        update_cache: yes

    #Install dependencies
    - name: Install required packages
      apt:
        name:
          - gnupg
          - curl
        state: present

    #### Add MongoDB GPG key and official repo only on supported distro (jammy)
    - name: Import MongoDB GPG key
      shell: |
        curl -fsSL https://pgp.mongodb.com/server-6.0.asc | \
        gpg -o /usr/share/keyrings/mongodb-server-6.0.gpg --dearmor
      args:
        creates: /usr/share/keyrings/mongodb-server-6.0.gpg

    - name: Add MongoDB repository
      copy:
        dest: /etc/apt/sources.list.d/mongodb-org-6.0.list
        content: |
          deb [ arch=amd64 signed-by=/usr/share/keyrings/mongodb-server-6.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse


    - name: Update apt cache again
      apt:
        update_cache: yes

   #### Install MongoDB: use official mongodb-org for jammy, otherwise install distro package
    - name: Install MongoDB
      apt:
        name: mongodb-org
        state: present


       

    - name: Enable and start MongoDB service
      systemd:
        name: mongod
        state: started
        enabled: yes

    #### Create admin user using mongosh if available, otherwise mongo
    - name: Create MongoDB Admin User
      shell: |
        if command -v mongosh >/dev/null 2>&1; then
          mongosh --eval "db = db.getSiblingDB('admin'); if (!db.getUser('{{ mongo_admin_user }}')) { db.createUser({ user: '{{ mongo_admin_user }}', pwd: '{{ mongo_admin_password }}', roles: [{ role: 'root', db: 'admin' }] }); }"
        elif command -v mongo >/dev/null 2>&1; then
          mongo --eval "db = db.getSiblingDB('admin'); if (!db.getUser('{{ mongo_admin_user }}')) { db.createUser({ user: '{{ mongo_admin_user }}', pwd: '{{ mongo_admin_password }}', roles: [{ role: 'root', db: 'admin' }] }); }"
        else
          echo "No mongo shell available" >&2
          exit 1
        fi
      args:
        executable: /bin/bash

    - name: Allow remote access in MongoDB config
      replace:
        path: /etc/mongod.conf
        regexp: '^  bindIp: 127.0.0.1'
        replace: '  bindIp: 0.0.0.0'

 

    #### Firewall setup
    - name: Install UFW
      apt:
        name: ufw
        state: present

    - name: Enable UFW
      ufw:
        state: enabled
        policy: allow

    - name: Allow required ports
      ufw:
        rule: allow
        port: "{{ item }}"
      loop:
        - 22
        - 80
        - 443
        - 3000
        - 3001
        - 27017

    #### SSH security
    - name: Disable root SSH login
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
        backup: yes   

    - name: Restart SSH service   
      service:    
        name: ssh    
        state: restarted     


## 🌐 Application Setup (web.yml) - Creating web.yml for ec2 instance in public subnet      
   
   - name: Setup Web Server  
     hosts: web  
     become: yes  
  
     vars:  
       app_dir: /home/ubuntu/app  
       frontend_dir: "{{ app_dir }}/frontend"  
       backend_dir: "{{ app_dir }}/backend"  
       backend_url: "http://13.206.107.143:3001"  

     tasks: 
   
       ####  BASIC SETUP
   
       ##### - name: Update apt cache  
       #####   apt:   
       #####     update_cache: yes   
   
       - name: Install required packages  
         apt:  
           name:  
             - curl  
             - git  
           state: present  
   
       - name: Add NodeSource repo  
         shell: curl -fsSL https://deb.nodesource.com/setup_18.x | bash -  
         args:   
           executable: /bin/bash  
    
       - name: Install Node.js
         apt:
           name: nodejs
           state: present

       #### CLONE CODE
       - name: Clone Repository
         git:
           repo: "https://github.com/SaurabhSaurabh/TravelMemory.git"
           dest: "{{ app_dir }}"
           force: yes
   
       - name: Set ownership to ubuntu
         file:
           path: "{{ app_dir }}"
           owner: ubuntu
           group: ubuntu
           recurse: yes

       #### PM2 SETUP 
       - name: Install PM2 globally
         npm:
           name: pm2
           global: yes
         become: yes
   
       - name: Set npm global bin path manually
         set_fact:
           npm_global_bin: "/usr/bin"

       ####  BACKEND 
   
       - name: Install Backend Dependencies
         command: npm install
         args:
           chdir: "{{ backend_dir }}"
         become_user: ubuntu

       #### - name: Stop existing backend
       ####   shell: |
       ####    pm2 PATH={{ npm_global_bin.stdout }}:$PATH pm2 delete backend || true
       ####   become_user: ubuntu

       - name: Start Backend
         command: pm2 start index.js --name backend
         args:
           chdir: "{{ backend_dir }}"
         become_user: ubuntu

       ####  FRONTEND 
   
       - name: Install Frontend Dependencies
         command: npm install
         args:
           chdir: "{{ frontend_dir }}"
         become_user: ubuntu

       #### - name: Create frontend .env file
       ####   copy:
       ####     dest: "{{ frontend_dir }}/.env"
       ####     content: |
       ####       REACT_APP_API_URL={{ backend_url }}
       ####   become_user: ubuntu

       - name: Build Frontend
         command: npm run build
         args:
           chdir: "{{ frontend_dir }}"
         become_user: ubuntu
   
       - name: Install serve globally
         npm:
           name: serve
           global: yes
         become: yes

       #### - name: Stop existing frontend
       ####   shell: |
       ####     pm2 PATH={{ npm_global_bin.stdout }}:$PATH pm2 delete frontend || true
       ####   become_user: ubuntu

       - name: Start Frontend
         command: pm2 start npx --name frontend -- serve -s /home/ubuntu/app/frontend/build -l 3000
         args:
           chdir: "{{ frontend_dir }}"
         become_user: ubuntu

       ####  PM2 SAVE 
       - name: Save PM2 process list
         command: pm2 save
         become_user: ubuntu
   
       - name: Setup PM2 startup
         command: pm2 startup systemd -u ubuntu --hp /home/ubuntu





Terraform commands:
1.	terraform init
2.	terraform plan
3.	terraform apply

<img width="975" height="423" alt="image" src="https://github.com/user-attachments/assets/06cbdaf6-e47b-4ef9-b635-a37d4962cc64" />

<img width="975" height="762" alt="image" src="https://github.com/user-attachments/assets/d67ba92d-9dc8-43b0-93c9-74b27233eb3a" />

<img width="811" height="533" alt="image" src="https://github.com/user-attachments/assets/096cede1-2ffb-4fbd-83fc-c0f27e4fc576" />

<img width="975" height="949" alt="image" src="https://github.com/user-attachments/assets/fc44459e-0eea-4be9-bb55-75731167b694" />

<img width="932" height="339" alt="image" src="https://github.com/user-attachments/assets/e414e92a-14ef-45a1-b365-ff7fc2836088" />

<img width="929" height="374" alt="image" src="https://github.com/user-attachments/assets/60001887-10f7-48b1-89ee-657e9a722ecb" />

<img width="975" height="254" alt="image" src="https://github.com/user-attachments/assets/ff8a5691-f81d-496e-b070-ef1707c78fef" />

<img width="975" height="216" alt="image" src="https://github.com/user-attachments/assets/38754c89-1517-437a-8201-64a2707150cd" />

<img width="975" height="197" alt="image" src="https://github.com/user-attachments/assets/eb53c8c8-cbc2-445a-8438-480025a29049" />

<img width="975" height="282" alt="image" src="https://github.com/user-attachments/assets/6679e54e-68ac-42ae-aadf-ec94b2bc0928" />



Connecting to ec2 instance in public subnet and configuration as a bastion host
<img width="975" height="820" alt="image" src="https://github.com/user-attachments/assets/72ba629c-29a5-4a80-8b49-ea37ddfa8554" />


Copying the content of mern-key.pem private key into the bastion host
Touch mern-key.pm
<img width="842" height="106" alt="image" src="https://github.com/user-attachments/assets/14715770-abb9-4d99-b9d9-55c11a5ce6a8" />
<img width="939" height="1292" alt="image" src="https://github.com/user-attachments/assets/1b99f321-0bc9-4ba8-ae9f-06b5134394ca" />


Connecting from bastion host to ec2 instance in private subnet
<img width="866" height="998" alt="image" src="https://github.com/user-attachments/assets/da739f24-6748-4956-bbfd-99a840552466" />

Now my ansible files are
Db.yml – this will create install mongodb in ec2 instance in private subnet
Web.yml :- this will configure frontend and backend in ec2 instance in public subnet

Ansible commands
1.	ansible-playbook -i hosts.ini db.yml
2.	ansible-playbook -i hosts.ini web.yml

Do below steps before running db.yml
/mnt/c/herovired/travelmemory-deployment-ansible-terraform/travelmemory-deployment-ansible-terraform/ansible
cp ./mern-key.pem ~/
chmod 400 mern-key.pem
Change RED (public ip of ec2, deployed in public subnet) and green(private ip of ec2, deployed in private subnet)
<img width="975" height="141" alt="image" src="https://github.com/user-attachments/assets/591f0881-f9d7-4cab-8ea6-62c875984b19" />
<img width="865" height="426" alt="image" src="https://github.com/user-attachments/assets/64630570-6618-47fe-8f84-24e25740fd62" />
<img width="975" height="576" alt="image" src="https://github.com/user-attachments/assets/ffc45c19-1987-4c22-931d-d5f35b2d9e1c" />
<img width="975" height="747" alt="image" src="https://github.com/user-attachments/assets/8da758b1-c5f0-4f7a-8531-b171b30e517f" />
<img width="975" height="542" alt="image" src="https://github.com/user-attachments/assets/c3b3f29a-2195-4714-8565-8cac6b53aed3" />



Testing from the browser:

http://52.66.152.69:3000/addexperience
<img width="868" height="502" alt="image" src="https://github.com/user-attachments/assets/e248446b-8b4c-4712-9f77-ac488ca4ef2a" />


http://52.66.152.69:3000
<img width="975" height="459" alt="image" src="https://github.com/user-attachments/assets/68282c3c-8a7f-4930-9c7c-0bb9fc863dc0" />

http://52.66.152.69:3001/trip/

<img width="975" height="563" alt="image" src="https://github.com/user-attachments/assets/1161e23f-b0d0-43c7-8421-5a0c2db62a8e" />













