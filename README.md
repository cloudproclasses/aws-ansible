# AWS infrastructure with Ansible — beginner project

This project uses a local Ubuntu machine as the Ansible control node. It provisions a small Amazon Linux 2023 EC2 web server in the **default VPC** in the configured region, creates a security group, and installs Nginx over SSH.

## Resources created

- One EC2 instance (`t3.micro` by default)
- One security group (HTTP from the internet; SSH only from the controller's detected public IPv4)
- One encrypted 8 GiB gp3 root EBS volume (deleted with the instance)
- EC2 tags to identify project resources

It reuses the default VPC and one existing subnet. It does not create a VPC, subnet, Internet Gateway, or route table.

## Important cost and security notes

- AWS charges may apply for EC2 runtime, EBS, and public IPv4 usage. Free Tier eligibility and pricing vary by account and region. Check current pricing before deployment.
- The web page is public over HTTP. Do not put secrets or private data on it.
- SSH is restricted to the public IPv4 detected from the controller at deployment time. If that IP changes, update the security group rule or rerun deployment.
- Use a dedicated non-production AWS account or sandbox where possible.
- `destroy.yml` terminates instances tagged with this project's `Project` tag and deletes the named security group. Review the variables and AWS account/region before running it.
- The default VPC must exist and the selected subnet must provide public internet access.
- This demo uses an existing EC2 key pair. Do not commit the private key to source control.

## 1. Install prerequisites on Ubuntu

```bash
sudo apt update
sudo apt install -y ansible awscli python3-boto3 python3-botocore
```

Install the collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

Check:

```bash
ansible --version
aws --version
ansible-galaxy collection list
ansible localhost -m ping
```

## 2. Configure AWS authentication

The playbooks use your default AWS CLI credentials. Configure them once with an IAM access key:

```bash
aws configure
aws sts get-caller-identity
```

If `aws sts get-caller-identity` already works in your terminal, nothing more is needed. Ansible and the AWS CLI pick up the `[default]` credentials from `~/.aws/credentials` (or the `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` environment variables). The region comes from `aws_region` in `group_vars/all.yml`.

The identity needs permissions for EC2 describe/create/modify/terminate operations, security-group create/authorize/revoke/delete operations, and SSM `GetParameter` for the public AMI parameter. IAM policies may also require permissions for tagging, creating network interfaces, and using the selected key pair.

## 3. Create an EC2 key pair once

The playbook expects an existing EC2 key pair in the configured region. Create one if needed:

```bash
mkdir -p ~/.ssh
aws ec2 create-key-pair \
  --key-name ansible-demo-key \
  --query 'KeyMaterial' \
  --output text \
  --region ap-south-1 > ~/.ssh/ansible-demo-key.pem

chmod 400 ~/.ssh/ansible-demo-key.pem
```

If the key pair already exists, do not run the create command again. Set `ec2_key_name` and `private_key_path` in `group_vars/all.yml` to your existing key pair and matching private key. AWS does not let you download the private key again after creation.

## 4. Configure project variables

Edit `group_vars/all.yml`:

- `aws_region`: target region
- `ec2_key_name`: existing EC2 key pair name in that region
- `private_key_path`: local path to its private key

The AMI is resolved dynamically from the public AWS Systems Manager parameter for the latest Amazon Linux 2023 x86_64 image.

## 5. Deploy

From the project root:

```bash
ansible-playbook playbooks/deploy.yml
```

The first play runs on localhost and provisions AWS resources. It then adds the new instance to a temporary in-memory Ansible group named `webservers`. The second play connects over SSH, installs Nginx, publishes the page, and starts the service.

Open the public IP printed at the end:

```text
http://EC2_PUBLIC_IP/
```

## 6. Verify and learn idempotency

Run the same deployment command again:

```bash
ansible-playbook playbooks/deploy.yml
```

Ansible should reuse the tagged EC2 instance and existing security group rather than create duplicates. Some tasks may still report changes if AWS reports a difference or if the web page is updated.

For a preview of supported changes:

```bash
ansible-playbook playbooks/deploy.yml --check
```

Check mode is not a perfect simulation for every cloud module. Do not treat it as a guarantee that the real run will succeed.

## 7. Destroy the project resources

Review the AWS account (`aws sts get-caller-identity`), region, and project tag first. Then run:

```bash
ansible-playbook playbooks/destroy.yml
```

This terminates matching project instances and deletes the security group. Confirm in the EC2 console that the instance is terminated and the security group is gone.

## Ansible concepts demonstrated

- **Inventory:** `inventory/hosts.ini` defines localhost.
- **Playbook / plays:** YAML automation in `playbooks/deploy.yml`.
- **Tasks:** Individual operations inside each play.
- **Modules:** `amazon.aws.ec2_instance`, `amazon.aws.ec2_security_group`, `ansible.builtin.copy`, `ansible.builtin.dnf`, etc.
- **Variables:** Reusable settings in `group_vars/all.yml`.
- **Facts:** Disabled for provisioning; gathered on the EC2 host during configuration.
- **Register:** Captures AWS and command module results for later tasks.
- **Conditionals:** `when` prevents an empty termination operation.
- **Loops / filters:** Jinja `map` extracts instance IDs from discovered instances.
- **Handlers:** A changed home page notifies the Nginx restart handler.
- **Dynamic inventory:** `add_host` adds the newly created EC2 instance to the current run.
- **Privilege escalation:** `become: true` runs configuration tasks with sudo.
- **Idempotency:** `state: present`, `state: running`, package `state: present`, and service state describe the desired end state.
- **Tags:** AWS resource tags identify resources owned by this project.

## Troubleshooting

- **AccessDenied:** The IAM user/role in your AWS CLI credentials lacks one or more required AWS permissions.
- **No default VPC:** Create or choose a VPC and adapt the VPC/subnet discovery tasks.
- **InvalidKeyPair.NotFound:** The key pair name is absent in the selected region.
- **SSH timeout:** Check that the subnet has a route to an Internet Gateway, the instance has a public IP, and the security group allows TCP/22 from the controller's current public IP.
- **SSH permission denied:** Check that the private key matches the EC2 key pair and has mode `0400`.
- **Python/module errors:** Reinstall the collection and ensure Boto3/Botocore are available to the Python interpreter used by Ansible.
