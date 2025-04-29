
# Ansible Demo with Docker


This project demonstrates how to use Ansible to manage multiple Docker containers as target servers. It showcases Ansible's ability to automate server configuration and management across multiple hosts.

## Table of Contents

-   [Block Diagram](#block-diagram)
-   [Setup Instructions](#setup-instructions)
-   [Verification](#verification)
-   [Cleanup](#cleanup)
-   [Troubleshooting](#troubleshooting)

## Block Diagram

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|  WSL2 (Control   |     |  Docker Desktop  |     |  Docker          |
|   Node)          |     |                  |     |  Containers      |
|                  |     |                  |     |                  |
|  +------------+  |     |  +------------+  |     |  +------------+  |
|  |            |  |     |  |            |  |     |  |            |  |
|  |  Ansible   |  |     |  |  Docker    |  |     |  |  Server1   |  |
|  |  Playbook  |  |     |  |  Engine    |  |     |  |  (2221)    |  |
|  |            |  |     |  |            |  |     |  |            |  |
|  +------------+  |     |  +------------+  |     |  +------------+  |
|        |         |     |        |         |     |        |         |
|  +------------+  |     |  +------------+  |     |  +------------+  |
|  |            |  |     |  |            |  |     |  |            |  |
|  |  SSH Keys  |  |     |  |  Port      |  |     |  |  Server2   |  |
|  |            |  |     |  |  Mapping   |  |     |  |  (2222)    |  |
|  +------------+  |     |  +------------+  |     |  +------------+  |
|        |         |     |        |         |     |        |         |
|  +------------+  |     |  +------------+  |     |  +------------+  |
|  |            |  |     |  |            |  |     |  |            |  |
|  | Inventory  |  |     |  |  Network   |  |     |  |  Server3   |  |
|  | File       |  |     |  |  Bridge    |  |     |  |  (2223)    |  |
|  +------------+  |     |  +------------+  |     |  +------------+  |
|        |         |     |        |         |     |        |         |
+--------|---------+     +--------|---------+     +--------|---------+
         |                        |                        |
         |                        |                        |
         |                        |                        |
         +------------------------+------------------------+
                                  |
                                  v
                           +------------+
                           |            |
                           |  SSH       |
                           |  Protocol  |
                           |            |
                           +------------+
```

### Component Description

1. **WSL2 (Control Node)**

    - Runs Ansible playbook
    - Manages SSH keys
    - Contains inventory file
    - Executes commands

2. **Docker Desktop**

    - Manages container lifecycle
    - Handles port mapping
    - Provides network bridge
    - Manages container resources

3. **Docker Containers**

    - Run Ubuntu OS
    - Expose SSH ports
    - Execute Ansible tasks
    - Maintain consistent state

4. **SSH Protocol**
    - Secure communication
    - Key-based authentication
    - Port forwarding
    - Remote command execution

### Data Flow

1. Ansible playbook is executed on WSL2
2. SSH keys are used for authentication
3. Commands are sent through Docker's port mapping
4. Containers execute commands and return results
5. Ansible processes results and continues workflow

## Setup Instructions

### 1. Install Required Software

In WSL2, install Ansible and SSH client, and docker integration with WSL distros:

```bash
wsl --install
```

```bash
sudo apt update
```
![1](https://github.com/user-attachments/assets/8a6b82d9-2198-4b81-bb37-08b45278f351)


```bash
sudo apt install -y ansible openssh-client
```
![1](https://github.com/user-attachments/assets/e88e7084-46a3-4285-bb48-e259021b3d26)

![2](https://github.com/user-attachments/assets/53f3f8a3-c4ef-4778-a7fb-ab496d87b335)


### 2. Create Project Directory

```bash
mkdir -p ~/ansible-demo
cd ~/ansible-demo
mkdir -p .ssh
```

![3](https://github.com/user-attachments/assets/5dab7b5f-af55-47b8-98cc-8eea7748c81c)

### 3. Generate SSH Keys

```bash
# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ./.ssh/ansible_key -N ""

# Set proper permissions
chmod 700 .ssh
chmod 600 .ssh/ansible_key
chmod 644 .ssh/ansible_key.pub
```

![4](https://github.com/user-attachments/assets/5a430d70-dfc2-408a-8ff1-8f756b893f7e)

![5](https://github.com/user-attachments/assets/00cf6bbf-3ba7-4672-b926-208b9dbb1a11)



### 4. Create Docker Containers

```bash
# Create 5 Ubuntu containers with SSH port mapping
for i in {1..5}; do
    docker run -d \
        --name server$i \
        -p 222$i:22 \
        ubuntu sleep infinity
done

# Install required packages and configure SSH
for i in {1..5}; do
    # Install packages
    docker exec server$i bash -c "apt-get update && apt-get install -y openssh-server python3"

    # Configure SSH
    docker exec server$i bash -c "mkdir -p /root/.ssh"
    docker exec server$i bash -c "echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config"
    docker exec server$i bash -c "echo 'PubkeyAuthentication yes' >> /etc/ssh/sshd_config"
    docker exec server$i bash -c "echo 'PasswordAuthentication no' >> /etc/ssh/sshd_config"

    # Copy SSH keys
    docker cp ./.ssh/ansible_key.pub server$i:/root/.ssh/authorized_keys
    docker cp ./.ssh/ansible_key server$i:/root/.ssh/id_rsa

    # Set proper permissions and ownership
    docker exec server$i bash -c "chown -R root:root /root/.ssh"
    docker exec server$i bash -c "chmod 700 /root/.ssh"
    docker exec server$i bash -c "chmod 600 /root/.ssh/id_rsa"
    docker exec server$i bash -c "chmod 644 /root/.ssh/authorized_keys"

    # Start SSH service
    docker exec server$i bash -c "service ssh start"
done
```

![6](https://github.com/user-attachments/assets/f4748723-9762-4f96-a369-a5ead2121fcb)


### 5. Verify containers are up and running:

```bash
docker ps
```
![7](https://github.com/user-attachments/assets/bbafc1d7-3836-4862-934f-84d41b0e9812)


### 6. Find IP of docker containers (optional ps~ we wont use this because of port mapping):

```bash
for i in {1..5}; do
  echo -e "\n IP if server${i}"
  docker inspect server${i} | grep IPAddress
done
```

![8](https://github.com/user-attachments/assets/4a97c88b-425c-4c1f-81cd-7fec52160299)

### 7. Create Inventory File in the current directory

Create `inventory.ini`:

```bash
cat > inventory.ini
```

```ini
[servers]
server1 ansible_host=localhost ansible_port=2221
server2 ansible_host=localhost ansible_port=2222
server3 ansible_host=localhost ansible_port=2223
server4 ansible_host=localhost ansible_port=2224
server5 ansible_host=localhost ansible_port=2225

[servers:vars]
ansible_user=root
ansible_ssh_private_key_file=./.ssh/ansible_key
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```
Then press `Ctrl + D` to save and exit


![9](https://github.com/user-attachments/assets/66afaae4-f9f2-4958-875c-433ddbd688e0)


### 8. Create Ansible Playbook in the current directory

Create `playbook.yml`:

```bash
cat > playbook.yml
```

```yaml
---
- name: Configure multiple servers
  hosts: servers
  become: yes

  tasks:
      - name: Update apt package index
        apt:
            update_cache: yes

      - name: Install Python 3
        apt:
            name: python3
            state: latest

      - name: Create test file with content
        copy:
            dest: /root/test_file.txt
            content: |
                This is a test file created by Ansible
                Server name: {{ inventory_hostname }}
                Current date: {{ ansible_date_time.date }}

      - name: Display system information
        command: uname -a
        register: uname_output

      - name: Show disk space
        command: df -h
        register: disk_space

      - name: Print results
        debug:
            msg:
                - "System info: {{ uname_output.stdout }}"
                - "Disk space: {{ disk_space.stdout_lines }}"
```

Then press `Ctrl + D` to save and exit

![10](https://github.com/user-attachments/assets/76818079-e8d4-443e-80ca-720f37bcfe42)


The file are created in the `ansible-demo` folder, and we can check them in the file explorer by searching ansible-demo
![11](https://github.com/user-attachments/assets/a9a123ba-7d55-478a-9d6b-dd5ae5cff363)


### 9. Run Ansible Playbook

```bash
ansible-playbook -i inventory.ini playbook.yml
```

![12](https://github.com/user-attachments/assets/2429cee1-9e03-4fc4-83bc-bebe9d392ef0)

## Verification

### Manual Verification

```bash
# Check Python version
for i in {1..5}; do
    echo "Python version on server$i:"
    docker exec server$i python3 --version
done
```

![13](https://github.com/user-attachments/assets/49a85154-7f30-4b8e-903d-c1f27eb0f512)

```bash
# Verify test file creation
for i in {1..5}; do
    echo "Contents on server$i:"
    docker exec server$i cat /root/test_file.txt
done
```

![14](https://github.com/user-attachments/assets/303408b1-7d63-486a-9076-3d7e11d19bf2)


```bash
# Check system information
for i in {1..5}; do
    echo "System info for server$i:"
    docker exec server$i uname -a
done
```

![15](https://github.com/user-attachments/assets/60204a53-7ac5-40d5-a9c0-6412483b0df1)

```bash
# Verify disk space
for i in {1..5}; do
    echo "Disk space on server$i:"
    docker exec server$i df -h
done
```

![16](https://github.com/user-attachments/assets/3e4a7c19-bbe3-4192-ab67-f7da974c228f)

### Ansible Verification

```bash
# Check file existence
ansible servers -i inventory.ini -m stat -a "path=/root/test_file.txt"
```

![17](https://github.com/user-attachments/assets/8cebbaa2-3452-4fdd-bc54-4b6825e37613)


```bash
# Check Python version
ansible servers -i inventory.ini -m command -a "python3 --version"
```

![18](https://github.com/user-attachments/assets/9d208989-9a83-4a02-b489-c60440d5a8b1)


## Cleanup

To clean up the environment:

```bash
# Stop and remove containers
for i in {1..5}; do
    docker stop server$i
    docker rm server$i
done
```
![19](https://github.com/user-attachments/assets/4eb190b9-e367-4759-968e-da6903a32e93)



```bash
# Remove SSH keys
rm -rf .ssh
```

## Troubleshooting

### Common Issues and Solutions

1. **SSH Connection Issues**

    - Problem: Permission denied (publickey)
    - Solution: Ensure proper ownership and permissions of SSH keys

    ```bash
    docker exec server1 bash -c "chown -R root:root /root/.ssh"
    docker exec server1 bash -c "chmod 700 /root/.ssh"
    docker exec server1 bash -c "chmod 600 /root/.ssh/id_rsa"
    docker exec server1 bash -c "chmod 644 /root/.ssh/authorized_keys"
    ```

2. **SSH Service Not Running**

    - Problem: Connection refused
    - Solution: Start SSH service

    ```bash
    docker exec server1 service ssh start
    ```

3. **Permission Issues**
    - Problem: Cannot access files
    - Solution: Set proper permissions
    ```bash
    chmod 700 .ssh
    chmod 600 .ssh/ansible_key
    chmod 644 .ssh/ansible_key.pub
    ```
