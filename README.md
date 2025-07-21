# Ansible Web Server Deployment

This repository contains an Ansible playbook designed to deploy and secure a basic NGINX web server on an Ubuntu 22.04 LTS EC2 instance. The playbook is structured according to Ansible best practices and aims to be idempotent, ensuring consistent configuration upon repeated runs.

## 🎯 Project Goal

The primary goal of this project is to automate the setup of a web server, including:
* Installation and custom configuration of NGINX.
* Serving web content from `/opt/static-sites`.
* Implementing Uncomplicated Firewall (UFW) rules.
* Creating a dedicated `webapp` user for web content ownership and NGINX processes.
* Hardening SSH security.
* Configuring automatic security updates and Fail2Ban for brute-force protection.
* Validating the deployment with an automated HTTP test.

## 📦 Project Structure

The project follows a standard Ansible directory structure:

ansible-webserver/
├── inventory/
│   └── hosts.ini             # Defines target hosts (EC2 instance)
├── playbooks/
│   └── site.yml              # Main playbook orchestrating roles
├── roles/
│   ├── common/               # Basic system setup (apt update, upgrade, webapp user creation)
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── ...
│   ├── ufw/                  # Uncomplicated Firewall configuration
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── ...
│   ├── nginx/                # NGINX installation and configuration
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── templates/
│   │   │   ├── nginx.conf.j2
│   │   │   └── index.html.j2
│   │   └── ...
│   ├── ssh_security/         # SSH hardening (disable root login, enforce key-only)
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── ...
│   └── security_updates/     # Automatic updates and Fail2Ban installation
│       ├── tasks/
│       │   └── main.yml
│       ├── files/
│       │   └── jail.local
│       └── ...
└── ansible.cfg               # Ansible configuration file
└── README.md                 # This documentation


## ✨ Features Implemented

* **Idempotency**: The playbook is designed to be idempotent, meaning running it multiple times will not cause unintended changes if the system is already in the desired state.
* **NGINX Deployment**:
    * Installs NGINX.
    * Uses a custom `nginx.conf` template for `sites-available/default` to replace the default configuration.
    * Configures NGINX to serve content from `/opt/static-sites`.
    * Sets NGINX worker processes to run under the `webapp` user.
    * Deploys a dynamic `index.html` template.
* **Firewall (UFW)**:
    * Installs and enables UFW.
    * Sets a default deny policy for incoming connections.
    * Explicitly allows SSH (port 22) and HTTP (port 80).
* **User and Permissions**:
    * Creates a dedicated `webapp` system user.
    * Ensures web files and the NGINX process run under the `webapp` user.
* **SSH Security**:
    * Disables `PermitRootLogin` in `sshd_config`.
    * Enforces `PasswordAuthentication no` (key-only login).
    * *Note*: The `UsePAM no` setting was found to cause authentication issues on Ubuntu and has been excluded from the `ssh_security` role.
* **Updates and Protection**:
    * Enables automatic security updates using `unattended-upgrades`.
    * Installs and configures `fail2ban` for brute-force attack protection (specifically for SSH).
* **Functionality Validation**:
    * Includes a `post_tasks` section to perform an HTTP `uri` test on the deployed web server.
    * Asserts that the web server returns a 200 status code and the expected content.

## ⚙️ Requirements

* **Ansible**: Installed on your control machine (e.g., WSL Ubuntu).
* **Target VM**: An Ubuntu 22.04 LTS EC2 instance (or similar Linux VM) with public IP access.
* **SSH Key**: An AWS `.pem` key (e.g., `aws.pem`) associated with the EC2 instance, correctly placed and permissioned on your control machine (`~/.ssh/aws.pem` with `chmod 400`).
* **AWS Security Group**: Ensure inbound rules for SSH (port 22) and HTTP (port 80) are open from your source IP or `0.0.0.0/0`.

## 🚀 How to Run the Playbook

1.  **Clone the Repository**:
    ```bash
    git clone https://github.com/IndraVasek/Web-App-Deployment.git
    cd ansible-webserver
    ```
    (Replace `https://github.com/your-username/ansible-webserver.git` with your actual repository URL).

2.  **Prepare SSH Key**:
    Ensure your AWS private key (`.pem` file) is in your `~/.ssh/` directory on your Ansible control machine (WSL in this case) and has the correct permissions:
    ```bash
    cp /mnt/c/Users/minad/SSH/AWS-Key.pem ~/.ssh/aws.pem # Adjust path as needed
    chmod 400 ~/.ssh/aws.pem
    ```

3.  **Configure Inventory (`inventory/hosts.ini`)**:
    Update the `inventory/hosts.ini` file with the public IP address of your EC2 instance. Ensure the `ansible_user` is `ubuntu` and `ansible_ssh_private_key_file` points to your key.

    Example `inventory/hosts.ini`:
    ```ini
    [webservers]
    your_ec2_public_ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/aws.pem
    ```
    (Replace `your_ec2_public_ip` with your actual EC2 instance's public IP address).

4.  **Run the Playbook**:
    Execute the playbook from the root of the `ansible-webserver` directory:
    ```bash
    ansible-playbook -i inventory/hosts.ini playbooks/site.yml
    ```

## ✅ Validation

Upon successful completion of the playbook, you can:
* Open your web browser and navigate to `http://your_ec2_public_ip`. You should see the custom "Hello from Ansible NGINX!" page.
* Verify SSH security by attempting to SSH as `root` (which should now be denied) or with a password (which should also be denied).
* Check UFW status on the EC2 instance: `sudo ufw status verbose`.
* Check Fail2Ban status on the EC2 instance: `sudo systemctl status fail2ban`.

## 🛠️ Troubleshooting

* **`UNREACHABLE! => {"msg": "Failed to connect to the host via ssh: Permission denied (publickey)."}`**:
    * Ensure your `~/.ssh/aws.pem` has `chmod 400` permissions.
    * Verify the correct `ansible_user=ubuntu` and `ansible_ssh_private_key_file` in `inventory/hosts.ini`.
    * If using WSL, ensure your `.pem` key is in your WSL filesystem (e.g., `~/.ssh/aws.pem`), not a Windows path (`C:\...`).
    * If the issue arose after `ssh_security` role, check `/etc/ssh/sshd_config` on the EC2 instance (via EC2 Instance Connect) and ensure `UsePAM yes` (or is commented out).
* **"Took too long to respond" in browser / Port 80 inaccessible**:
    * This usually indicates a firewall issue *outside* the EC2 instance. Check your AWS EC2 instance's **Security Group** inbound rules. Ensure HTTP (port 80) is open from your IP or `0.0.0.0/0`.
* **NGINX service failed to start/restart**:
    * SSH into your EC2 instance (via Instance Connect if needed) and run `sudo nginx -t` to check for syntax errors in NGINX configuration files.
    * Check NGINX service logs: `sudo systemctl status nginx.service` or `sudo journalctl -xeu nginx.service`.

---
