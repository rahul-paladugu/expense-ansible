# Expense Application - Ansible Deployment

Automated deployment of a three-tier expense management application using Ansible. This project automates the infrastructure provisioning, database setup, backend API deployment, and frontend web server configuration on AWS EC2 instances.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Deployment Steps](#deployment-steps)
- [Troubleshooting](#troubleshooting)
- [Future Improvements](#future-improvements)

## Overview

This Ansible playbook suite automates the deployment of an expense tracking application across multiple EC2 instances. The deployment is organized in phases:

1. **Phase 0** - Infrastructure provisioning (EC2 instances, Route53 DNS)
2. **Phase 1** - MySQL database setup
3. **Phase 2** - Node.js backend application deployment
4. **Phase 3** - Nginx frontend server setup

## Architecture

```
┌─────────────────────────────────────────────┐
│       Frontend (Nginx Server)               │
│   IP: expense.rscloudservices.icu           │
│   - Serves static assets                    │
│   - Proxies API requests to backend         │
└──────────────────┬──────────────────────────┘
                   │
       ┌───────────┴──────────────┐
       │                          │
       │                   ┌──────▼──────────┐
       │                   │ Backend Server  │
       │                   │ Node.js API     │
       │                   │ Port 8080       │
       │                   └─────┬───────────┘
       │                         │
       └─────────────┬───────────┘
                     │
       ┌─────────────▼─────────────┐
       │    MySQL Database         │
       │  mysql.rscloudservices.icu│
       │  Port 3306                │
       └───────────────────────────┘
```

## Prerequisites

### Control Machine (Where you run Ansible)
- Ansible 2.9+ installed
- AWS CLI configured with credentials
- Python 3.6+ with boto3 and botocore
- SSH key pair for EC2 access

### AWS Account
- EC2 instances (t3.micro or similar)
- Security group with appropriate inbound rules (22, 80, 443, 3306)
- Route53 hosted zone
- An existing AMI ID (e.g., Amazon Linux 2)

### Installation

```bash
# Install Ansible
pip3 install ansible

# Install AWS dependencies
pip3 install boto3 botocore

# Verify installation
ansible --version
```

## Quick Start

### 1. Update Variables in Playbooks

Edit `0-instances.yaml` and update:
```yaml
vars:
  servers:
    - mysql
    - backend
    - frontend
  instance_type: t3.micro
  image_id: ami-XXXXXXXXX        # Your AMI ID
  security_group_id: sg-XXXXXXXX # Your security group
  hosted_zone: your-domain.com   # Your domain
  hosted_zone_id: ZXXXXX         # Your hosted zone ID
```

### 2. Create Inventory File

Create/update `inventory.ini`:
```ini
[local]
localhost ansible_connection=local

[mysql]
mysql.rscloudservices.icu

[backend]
backend.rscloudservices.icu

[frontend]
frontend.rscloudservices.icu

[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/your-key.pem
ansible_python_interpreter=/usr/bin/python3
```

### 3. Run the Playbooks in Order

```bash
# Phase 0: Create infrastructure
ansible-playbook -i inventory.ini 0-instances.yaml

# Phase 1: Setup database
ansible-playbook -i inventory.ini 1-mysql.yaml

# Phase 2: Deploy backend
ansible-playbook -i inventory.ini 2-backend.yaml

# Phase 3: Deploy frontend
ansible-playbook -i inventory.ini 3-frontend.yaml
```

## Project Structure

```
expense-ansible/
├── 0-instances.yaml      # AWS EC2 instance provisioning
│   └── Creates 3 instances (mysql, backend, frontend)
│   └── Creates Route53 DNS records
│
├── 1-mysql.yaml          # MySQL database configuration
│   └── Installs mysql-server
│   └── Starts mysqld service
│   └── Sets root password
│
├── 2-backend.yaml        # Node.js backend deployment
│   └── Installs Node.js 20
│   └── Downloads application code from S3
│   └── Installs npm dependencies
│   └── Creates systemd service
│   └── Loads database schema
│
├── 3-frontend.yaml       # Nginx frontend server
│   └── Installs nginx
│   └── Downloads static assets from S3
│   └── Configures nginx reverse proxy
│
├── backend.service       # Systemd unit file for backend
│   └── Service configuration
│   └── Auto-restart on failure
│
├── expense.conf          # Nginx configuration
│   └── Reverse proxy setup
│   └── API routing rules
│
└── inventory.ini         # Host inventory
    └── Define target servers
    └── Connection parameters
```

## Configuration

### Database Configuration

In `1-mysql.yaml`:
```yaml
- name: configure admin credentials to debug
  ansible.builtin.shell:
    mysql_secure_installation --set-root-pass ExpenseApp@1
```

**⚠️ Note:** Root password is hardcoded. For production, use Ansible Vault:
```bash
# Create vault file
ansible-vault create group_vars/all/vault.yml

# Add vault_mysql_password variable
# Then reference it in playbook as: {{ vault_mysql_password }}

# Run playbook with vault
ansible-playbook -i inventory.ini 1-mysql.yaml --ask-vault-pass
```

### Backend Configuration

In `2-backend.yaml`:
- Downloads code from S3: `https://expense-joindevops.s3.us-east-1.amazonaws.com/expense-backend-v2.zip`
- Database host: `mysql.rscloudservices.icu`
- Database user: `root`
- Database password: `ExpenseApp@1` (hardcoded)

### Frontend Configuration

In `3-frontend.yaml`:
- Downloads code from S3: `https://expense-joindevops.s3.us-east-1.amazonaws.com/expense-frontend-v2.zip`
- Nginx serves on port 80
- Static files in: `/usr/share/nginx/html`

## Deployment Steps

### Phase 0: Infrastructure Provisioning

```bash
ansible-playbook -i inventory.ini 0-instances.yaml -v
```

This will:
- Create 3 EC2 instances (mysql, backend, frontend)
- Apply security groups
- Create Route53 DNS records
- Output instance IPs

### Phase 1: Database Setup

```bash
ansible-playbook -i inventory.ini 1-mysql.yaml -v
```

This will:
- Install MySQL server
- Start mysqld service
- Set root password to `ExpenseApp@1`

Verify:
```bash
ssh -i your-key.pem ec2-user@mysql.rscloudservices.icu
mysql -u root -pExpenseApp@1 -e "SHOW DATABASES;"
```

### Phase 2: Backend Deployment

```bash
ansible-playbook -i inventory.ini 2-backend.yaml -v
```

This will:
- Install Node.js 20
- Download backend code from S3
- Install npm dependencies
- Create `/app` directory with code
- Create `expense` system user
- Deploy and start backend service
- Load database schema

Verify:
```bash
ssh -i your-key.pem ec2-user@backend.rscloudservices.icu
curl http://localhost:8080/health
```

### Phase 3: Frontend Deployment

```bash
ansible-playbook -i inventory.ini 3-frontend.yaml -v
```

This will:
- Install nginx
- Download frontend assets from S3
- Configure nginx as reverse proxy
- Start nginx service

Verify:
```bash
curl http://expense.rscloudservices.icu
```

## Troubleshooting

### SSH Connection Issues

```bash
# Test connectivity
ansible all -i inventory.ini -m ping

# Check SSH access directly
ssh -i ~/.ssh/your-key.pem ec2-user@mysql.rscloudservices.icu

# Verify security group allows port 22
aws ec2 describe-security-groups --group-ids sg-XXXXXXXX
```

### MySQL Connection Issues

```bash
# Check if MySQL is running
ssh ec2-user@mysql.rscloudservices.icu
sudo systemctl status mysqld

# Check listening ports
sudo ss -tlnp | grep 3306

# Test connection
mysql -h mysql.rscloudservices.icu -u root -pExpenseApp@1
```

### Backend Service Not Starting

```bash
# Check service status
ssh ec2-user@backend.rscloudservices.icu
sudo systemctl status backend

# View logs
sudo journalctl -u backend -n 50

# Check if npm installed correctly
cd /app && npm list
```

### Frontend Not Loading

```bash
# Check nginx status
ssh ec2-user@frontend.rscloudservices.icu
sudo systemctl status nginx

# Check nginx logs
sudo tail -50 /var/log/nginx/error.log

# Verify files are extracted
ls -la /usr/share/nginx/html/
```

### DNS Resolution Issues

```bash
# Test DNS
nslookup mysql.rscloudservices.icu
nslookup expense.rscloudservices.icu

# If not resolving, check Route53
aws route53 list-resource-record-sets --hosted-zone-id Z050001923LY47PA0PTIR
```

## Security Considerations

### Current Implementation Issues

⚠️ **Password Hardcoding**: Passwords are hardcoded in playbooks. For production:

```bash
# Option 1: Use Ansible Vault
ansible-vault create credentials.yml
# Add: mysql_password: "secure_password"

# Option 2: Use environment variables
export MYSQL_PASSWORD="secure_password"
# Reference as: {{ lookup('env', 'MYSQL_PASSWORD') }}

# Option 3: AWS Secrets Manager
# Fetch secrets during playbook execution
```

### Recommendations

1. **Use Vault for Secrets**
   ```bash
   ansible-vault create group_vars/all/vault.yml
   ```

2. **Restrict Security Groups**
   - Only allow SSH (22) from your IP
   - Only allow 3306 from backend security group
   - Only allow 80/443 from load balancer

3. **SSH Key Management**
   - Never commit private keys to Git
   - Use `.gitignore` for `*.pem` files
   - Rotate keys regularly

4. **Database Access**
   - Create application-specific users (not root)
   - Limit permissions to specific databases
   - Use strong passwords

5. **S3 Access**
   - Use IAM roles for EC2 instances
   - Don't hardcode AWS credentials
   - Restrict S3 bucket access

## Playbook Details

### 0-instances.yaml

Creates EC2 instances and DNS records:
- Installs Python dependencies (boto3, botocore)
- Creates 3 instances with tags
- Saves instance data to JSON file
- Creates Route53 A records for each instance
- Maps frontend public IP to main domain

### 1-mysql.yaml

Sets up MySQL database:
- Installs mysql-server package
- Starts and enables mysqld
- Runs `mysql_secure_installation`

### 2-backend.yaml

Deploys Node.js backend:
- Enables Node.js 20 module
- Installs nodejs and mysql packages
- Creates `expense` system user
- Downloads code from S3
- Installs npm dependencies
- Copies systemd service file
- Loads database schema from `/app/schema/backend.sql`
- Starts backend service

### 3-frontend.yaml

Sets up Nginx frontend:
- Installs nginx package
- Removes default nginx content
- Downloads frontend assets from S3
- Extracts to `/usr/share/nginx/html`
- Copies nginx configuration
- Restarts nginx service

## Future Improvements

- [ ] Implement Ansible Vault for credential management
- [ ] Create separate playbooks for staging/production
- [ ] Add health checks and monitoring
- [ ] Implement automatic backups
- [ ] Use variables files for environment-specific configs
- [ ] Add rollback procedures
- [ ] Create idempotent database schema loading
- [ ] Implement load balancing
- [ ] Add SSL/TLS certificates (Let's Encrypt)
- [ ] Create pre-deployment testing tasks
- [ ] Add post-deployment verification steps

## Common Issues & Solutions

### Issue: `connection timed out` during playbook run

**Solution:**
- Verify security group allows SSH (port 22)
- Ensure instances are fully running
- Check network ACLs
- Verify SSH key has correct permissions: `chmod 600 ~/.ssh/key.pem`

### Issue: S3 download fails

**Solution:**
- Verify S3 bucket is public or EC2 has IAM role
- Check URLs are correct
- Ensure bucket exists in the region

### Issue: `mysql_secure_installation` command not found

**Solution:**
- The playbook assumes Linux with mysql-server
- On different distros, package names may vary
- Check available packages: `dnf search mysql`

### Issue: Playbook runs but service doesn't start

**Solution:**
- Check service logs: `journalctl -u backend -n 50`
- Verify dependencies are installed
- Check file permissions and ownership

## Running Specific Phases

To run only one playbook:

```bash
# Only database
ansible-playbook -i inventory.ini 1-mysql.yaml

# Only backend
ansible-playbook -i inventory.ini 2-backend.yaml
```

To re-run with more verbosity:

```bash
ansible-playbook -i inventory.ini playbook.yaml -vv
```

To do a dry-run (check mode):

```bash
ansible-playbook -i inventory.ini playbook.yaml --check
```

## Testing Connectivity

```bash
# Test all hosts
ansible all -i inventory.ini -m ping

# Test MySQL host
ansible mysql -i inventory.ini -m ping

# Run command on specific host
ansible mysql -i inventory.ini -m command -a "mysql --version"
```

## Maintenance

### Viewing Instance Status

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=expense" \
  --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,PrivateIpAddress,State.Name]' \
  --output table
```

### Stopping Instances

```bash
aws ec2 stop-instances --instance-ids i-XXXXXXXXX
```

### Terminating Resources

```bash
aws ec2 terminate-instances --instance-ids i-XXXXXXXXX
```

## Support & Contribution

For issues or improvements, open an issue on the repository.

## License

MIT License

---

**Note:** This project is designed for learning and development purposes. For production use, implement security best practices including credential management, SSL/TLS, monitoring, and backup procedures.
