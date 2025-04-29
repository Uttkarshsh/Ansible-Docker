<div align="center">
    <img src="/assets/1615449048633.png">
</div>

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
https://github.com/user-attachments/assets/200a5052-63f9-4fbb-a3e1-41883533a7ad


```bash
sudo apt update
```


```bash
sudo apt install -y ansible openssh-client
```
![image](![1](https://github.com/user-attachments/assets/152cdc16-4762-498b-a2bb-200dc92ca864)
)
 
![image](![2](https://github.com/user-attachments/assets/318cdb90-081e-4a26-9313-401c6501f637)
)

### 2. Create Project Directory

```bash
mkdir -p ~/ansible-demo
cd ~/ansible-demo
mkdir -p .ssh
```

![image](![3](https://github.com/user-attachments/assets/5340fae0-980a-41d4-be9e-2cc82200f098)
)

### 3. Generate SSH Keys

```bash
# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ./.ssh/ansible_key -N ""

# Set proper permissions
chmod 700 .ssh
chmod 600 .ssh/ansible_key
chmod 644 .ssh/ansible_key.pub
```

![image](![4](https://github.com/user-attachments/assets/7ed8410e-7fc3-466d-a0f2-80fcc369297c)
)
![5](https://github.com/user-attachments/assets/9ebf3d0c-df25-452f-b4a3-c9f92858dcd5)


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

![image](![6](https://github.com/user-attachments/assets/aa2afa19-9f1b-4893-8948-2086f9f8e0b8)
)

### 5. Verify containers are up and running:

```bash
docker ps
```

![image](![7](https://github.com/user-attachments/assets/7e32ef37-6da7-4f4c-9840-ed3211222655)
)

### 6. Find IP of docker containers (optional ps~ we wont use this because of port mapping):

```bash
for i in {1..5}; do
  echo -e "\n IP if server${i}"
  docker inspect server${i} | grep IPAddress
done
```

![image](![8](https://github.com/user-attachments/assets/2b53de12-ac6d-4694-a704-69ebbfea973a)
)

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


![image](![9](![9](https://github.com/user-attachments/assets/4e4956b7-c90c-457a-b1cd-c478e30f272f)
)
)

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

![image](![10](![10](https://github.com/user-attachments/assets/0b2969fa-086b-4430-bbe7-769d6bc78f8d)
)
)

The file are created in the `ansible-demo` folder, and we can check them in the file explorer by searching ansible-demo
![image](![11](![11](https://github.com/user-attachments/assets/967c0b27-42d5-44ef-bc0d-63725c0d0b97)
)
)

### 9. Run Ansible Playbook

```bash
ansible-playbook -i inventory.ini playbook.yml
```

![image](![12](https://github.com/user-attachments/assets/5e73d5a3-237c-447b-8cb0-a5233329da76)
)
 
## Verification

### Manual Verification

```bash
# Check Python version
for i in {1..5}; do
    echo "Python version on server$i:"
    docker exec server$i python3 --version
done
```

![image](![13](https://github.com/user-attachments/assets/2625c582-2605-47ed-a11d-5843334fa2a1)
)

```bash
# Verify test file creation
for i in {1..5}; do
    echo "Contents on server$i:"
    docker exec server$i cat /root/test_file.txt
done
```

![image](![14](https://github.com/user-attachments/assets/9c2fe977-226c-4d04-ae21-38d274bfe58b)
)

```bash
# Check system information
for i in {1..5}; do
    echo "System info for server$i:"
    docker exec server$i uname -a
done
```

![image](![15](https://github.com/user-attachments/assets/2abdf7bf-561b-4ddd-b7f2-c5236f9d988b)
)

```bash
# Verify disk space
for i in {1..5}; do
    echo "Disk space on server$i:"
    docker exec server$i df -h
done
```

![image](![16](https://github.com/user-attachments/assets/c5599a97-1b80-4a65-a2dc-92599a19b07d)
)

### Ansible Verification

```bash
# Check file existence
ansible servers -i inventory.ini -m stat -a "path=/root/test_file.txt"
```

![image](![17](https://github.com/user-attachments/assets/9b363cbd-e8cc-47df-aa84-eeb274667e60)
)

```bash
# Check Python version
ansible servers -i inventory.ini -m command -a "python3 --version"
```

![image](![17](https://github.com/user-attachments/assets/a8ef4c7d-4314-434e-ad07-05404f0f20cd)
)

## Cleanup

To clean up the environment:

```bash
# Stop and remove containers
for i in {1..5}; do
    docker stop server$i
    docker rm server$i
done
```

![image](![19](https://github.com/user-attachments/assets/1bf49a46-f525-4e90-84dd-b057be54b15e)
)

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
