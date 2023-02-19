# intrasonics_transmogrifier_assignment


<img width="1024" alt="Screenshot 2023-02-19 at 16 30 51" src="https://user-images.githubusercontent.com/104728608/219961551-35e4cb7a-e0dd-46da-b524-69d0bf253d1c.png">

---

## Objectives:<br>
The recently added functionality is expected to behave in the following way: 
-  Independent processes will be started up to serve individual client requests. These should all be started by the transmogrifier user. 
-  Each of these processes may run for an extended period, depending on how much work is required to serve their specific request. This workload is expected to be highly variable. 
-  A dedicated Transmogrified/ folder has been created on each server, which these processes will occasionally write to. 

You have been asked to create a solution which monitors the following at regular intervals:<br>
-  Processes currently running on the machine. 
-  Contents of the Transmogrified/ folder. 
Write a script in the language of your choosing, which will be used to achieve the above. 

## Description of the created script's logic:
The solution involves:
1. The main script
- to regularly monitor and log processes and content of the Transmogrified/ folder to .txt files 
- to place the created log files at the same (for simplicity) Transmogrified/ folder
2. The script launching these cron tasks:
- at regular intervals launch the main script
- at regular intervals create .tar archives and place them to Archives/ folder
- at regular intervals backup the .tar archives to a "master" server managed by a DevOps
3. [Optionally] Terraform + Ansible infrastructure to test the script


## Assumptions:
1. The task can be done by any one of the tools being used (Bash or Python or Terraform or Ansible). I am demonstrating different variations and combinations of them for demo purposes.
2. To keep things work smoothly, I am giving resources maximum rights. In an actual production environment, I would apply the principle of least privilege.
3. To keep things simple, I am creating EC2 instances in the default VPC. In an actual production environment, I would create the separate VPC, subnets, route tables, IGW, NACLs and would apply additional level of the network and systems hardening.
4. All the necessary functionality can be easily accomplished by utilizing a dedicated monitoring tool such as AWS CloudWatch. I am using scripting for demonstration purposes only.

---

## Bash <br>


<details markdown=1><summary markdown="span">Script</summary>

``` sh
#!/bin/bash

user="transmogrifier"
all_users=($(getent passwd | cut -d: -f1))

if [[ " ${all_users[@]} " =~ " ${user} " ]]; then
     # Check if the Transmogrified/ folder exists
    if [ -d "Transmogrified/" ]; then
        # Get the list of files in the Transmogrified/ folder
        files_list=()
        for filename in Transmogrified/*; do
            files_list+=("$filename")
        done
    else
        # Create the Transmogrified/ folder and set files_list to an empty array
        mkdir "Transmogrified/"
        echo "Transmogrified/ folder created"
        files_list=()

        # Set the owner and group for Transmogrified/ folder and give read, write, and execute permissions recursively
        sudo chown -R transmogrifier:ec2-user Transmogrified/
        sudo chmod -R g+rwx Transmogrified/
        sudo chmod -R o-rx Transmogrified/
    fi
    
    # Check if Archives/ exists and create it if not
    if [ ! -d "Archives/" ]; then
        # Create the Archives/ folder and set files_list to an empty array
        mkdir "Archives/"
        echo "Archives/ folder created"

        # Set the owner and group for Archives/ folder and give read, write, and execute permissions recursively
        sudo chown -R transmogrifier:ec2-user Archives/
        sudo chmod -R g+rwx Archives/
        sudo chmod -R o-rx Archives/
    fi

    # Get the list of running processes for user 'transmogrifier'
    process_list=()
    while read -r pid name username; do
        if [[ $username == $user ]]; then
            process_list+=("$pid $name $username")
        fi
    done < <(ps -eo pid,comm,user)

    # Print the lists to the console
    echo "Transmogrified files:"
    printf '%s\n' "${files_list[@]}"
    echo "Running processes for user 'transmogrifier':"
    printf '%s\n' "${process_list[@]}"

    # Save the lists to a file in Transmogrified/ directory
    file_prefix=$(date '+%Y-%m-%d_%H-%M-%S')
    {
        echo "Transmogrified files:"
        printf '%s\n' "${files_list[@]}"
        echo ""
        echo "Running processes for user 'transmogrifier':"
        printf '%s\n' "${process_list[@]}"
    } > "Transmogrified/${file_prefix}_file_and_process_list.txt"

    echo "Saved file and process lists to file: Transmogrified/${file_prefix}_files_and_processes_list.txt"
else
    sudo useradd "$user" 
    sudo usermod -a -G ec2-user "$user"
    echo ""$user" created, please create a password to the user"
fi
```

</details>

<details markdown=1><summary markdown="span">Cron script</summary>

``` sh
#!/bin/bash

crontab -l | { cat; echo "*/1 * * * * bash /home/ec2-user/script.sh"; } | crontab -

crontab -l | { cat; echo "*/1 * * * * tar -czvf /home/ec2-user/Archives/archive_$(hostname | cut -d '.' -f 1)_$(date +\%Y\%m\%d_\%H\%M\%S).tar.gz /home/ec2-user/Transmogrified/*"; } | crontab -

crontab -l | { cat; echo "0 0 * * 0 scp /home/ec2-user/Transmogrified/* user@remote.server:/path/to/remote/directory/"; } | crontab -
```
</details>

<details markdown=1><summary markdown="span">Pre-requisites to launch</summary>

``` sh
sudo chmod +x script.sh && sudo chmod +x script_cron.sh
sudo bash script.sh && sudo bash script_cron.sh
```
</details>

---

## Python <br>

<details markdown=1><summary markdown="span">Script</summary>

``` python3
#!/usr/bin/env python

import os
import subprocess
from datetime import datetime

user = "transmogrifier"
all_users = subprocess.check_output(["getent", "passwd"]).decode("utf-8").split("\n")
all_users = [u.split(":")[0] for u in all_users if u]

if user in all_users:
    # Check if the Transmogrified/ folder exists
    if os.path.isdir("Transmogrified/"):
        # Get the list of files in the Transmogrified/ folder
        files_list = [f for f in os.listdir("Transmogrified/") if os.path.isfile(os.path.join("Transmogrified/", f))]
    else:
        # Create the Transmogrified/ folder and set files_list to an empty array
        os.makedirs("Transmogrified/")
        print("Transmogrified/ folder created")
        files_list = []

        # Set the owner and group for Transmogrified/ folder and give read, write, and execute permissions recursively
        os.system("sudo chown -R transmogrifier:ec2-user Transmogrified/")
        os.system("sudo chmod -R g+rwx Transmogrified/")
        os.system("sudo chmod -R o-rx Transmogrified/")
    
    # Check if Archives/ exists and create it if not
    if not os.path.isdir("Archives/"):
        # Create the Archives/ folder and set files_list to an empty array
        os.makedirs("Archives/")
        print("Archives/ folder created")

        # Set the owner and group for Archives/ folder and give read, write, and execute permissions recursively
        os.system("sudo chown -R transmogrifier:ec2-user Archives/")
        os.system("sudo chmod -R g+rwx Archives/")
        os.system("sudo chmod -R o-rx Archives/")
    
    # Get the list of running processes for user 'transmogrifier'
    process_list = []
    ps_output = subprocess.check_output(["ps", "-eo", "pid,comm,user"]).decode("utf-8")
    for line in ps_output.split("\n")[1:]:
        if not line.strip():
            continue
        pid, name, username = line.split()
        if username == user:
            process_list.append(f"{pid} {name} {username}")
    
    # Print the lists to the console
    print("Transmogrified files:")
    print("\n".join(files_list))
    print("Running processes for user 'transmogrifier':")
    print("\n".join(process_list))
    
    # Save the lists to a file in Transmogrified/ directory
    file_prefix = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    with open(f"Transmogrified/{file_prefix}_file_and_process_list.txt", "w") as f:
        f.write("Transmogrified files:\n")
        f.write("\n".join(files_list) + "\n\n")
        f.write("Running processes for user 'transmogrifier':\n")
        f.write("\n".join(process_list) + "\n")
    print(f"Saved file and process lists to file: Transmogrified/{file_prefix}_file_and_process_list.txt")
else:
    os.system(f"sudo useradd {user}")
    os.system(f"sudo usermod -a -G ec2-user {user}")
    print(f"{user} created, please create a password to the user")
```
</details>

<details markdown=1><summary markdown="span">Cron script</summary>

``` python3
#!/usr/bin/env python

import subprocess

# Add cron job to run script.sh every minute
subprocess.run(['bash', '-c', 'echo "$(crontab -l ; echo \'*/1 * * * * bash /home/ec2-user/script.sh\') | crontab -"'])

# Add cron job to archive files every minute
subprocess.run(['bash', '-c', 'echo "$(crontab -l ; echo \'*/1 * * * * tar -czvf /home/ec2-user/Archives/archive_$(hostname | cut -d \'.\' -f 1)_$(date +\%Y\%m\%d_\%H\%M\%S).tar.gz /home/ec2-user/Transmogrified/*\') | crontab -"'])

# Add cron job to copy files to remote server every Sunday at midnight
subprocess.run(['bash', '-c', 'echo "$(crontab -l ; echo \'0 0 * * 0 scp /home/ec2-user/Transmogrified/* user@remote.server:/path/to/remote/directory/\') | crontab -"'])
```
</details>

<details markdown=1><summary markdown="span">Pre-requisites to launch</summary>

``` python3
sudo chmod +x script.py && sudo chmod +x script_cron.py

sudo python3 script.py && sudo python3 script_cron.py
```
</details>

---

## Optional (creating the infrastructure to host and launch the script - combination of Terraform and Ansible)

<details markdown=1><summary markdown="span">Terraform script</summary>

``` tf
provider "aws" {
  region                    = "us-east-1"
  shared_config_files       = ["/home/ec2-user/.aws/config"]
  shared_credentials_files  = ["/home/ec2-user/.aws/credentials"]
}


data "aws_availability_zones" "available" {
  state = "available"
}
data "aws_ssm_parameter" "current-ami" {
  name = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}


resource "aws_default_vpc" "default" {
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name = "Default VPC"
  }
}

resource "aws_subnet" "subnet_1" {
  vpc_id     = aws_default_vpc.default.id
  # cidr_block = "10.0.1.0/24"
  cidr_block = "172.31.98.128/25"
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "ansible-subnet_1"
  }
}

resource "aws_security_group" "ec2_security_group" {
  name        = "ec2-slave-security-group"
  description = "Allow ssh and http access"
  vpc_id      = aws_default_vpc.default.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for sake of security!!
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # change this to your ip for sake of security!!
  }
  

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"] 
  }
}

resource "aws_instance" "ansible_slave" {
  count = 1
  ami           = data.aws_ssm_parameter.current-ami.value
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.subnet_1.id
  vpc_security_group_ids = [aws_security_group.ec2_security_group.id]
  associate_public_ip_address = true
  key_name      = "test_delete"

  tags = {
    Name = "slave_instance${count.index + 1}"
  }
}

resource "local_file" "slaves_ips" {
    content = format("%s\n%s\n%s",
  aws_instance.ansible_slave.*.public_ip[0],
  aws_instance.ansible_slave.*.public_ip[1],
  aws_instance.ansible_slave.*.public_ip[2]
)

## this is to create only one slave_instance
# resource "local_file" "slaves_ips" {
#     content = aws_instance.ansible_slave.*.public_ip[0]

#   filename = "inventory"
# }


output "slaves_ips" {
  value = ["${aws_instance.ansible_slave.*.public_ip}"]
}
```
</details>

<details markdown=1><summary markdown="span">Ansible playbook</summary>

``` yml
---
- name: Create user and folder on slaves
  hosts: all
  become: true
  gather_facts: true

  vars:
    user: transmogrifier
    week_number: "{{ ansible_date_time.week_number }}"

  tasks:
    - name: Create transmogrifier user
      user:
        name: "{{ user }}"
        group: "ec2-user"
        state: present

    - name: Create Transmogrified folder
      file:
        path: /home/ec2-user/Transmogrified
        state: directory
        owner: "{{ user }}"
        group: "ec2-user"
        mode: '0777'
        
    - name: Create a Tar folder for archives
      file:
        path: /home/ec2-user/Archives
        state: directory
        owner: "{{ user }}"
        group: "ec2-user"
        mode: '0777'

    - name: Copy transmogrifier script to slave
      copy:
        src: /home/ec2-user/environment/Terraform/script.sh
        dest: /home/ec2-user/script.sh
        owner: "ec2-user"
        group: "ec2-user"
        mode: '0777'
        
    - name: Create cron task to run script every minute
      ansible.builtin.cron:
        user: "ec2-user"
        name: "Run script every minute"
        minute: "*/1"
        job: "bash /home/ec2-user/script.sh"    
        
        
    - name: Create weekly tar archive of Transmogrified folder
      ansible.builtin.cron:
        user: "ec2-user"
        name: "Weekly tar archive of Transmogrified folder"
        job: "tar -czvf /home/ec2-user/Archives/archive_$(hostname | cut -d '.' -f 1)_$(date +\%Y\%m\%d_\%H\%M\%S).tar.gz /home/ec2-user/Transmogrified/*"
        minute: "*/1"
        hour: "*"
        day: "*"
        month: "*"
        weekday: "*"

    - name: Move monthly archives to the master server
      ansible.builtin.cron:
        user: "ec2-user"
        name: "Monthly archive transfer to master server"
        job: "rsync -avz /home/ec2-user/Archives/*.tar.gz root@ec2-3-80-47-11.compute-1.amazonaws.com"
        day: 1
        
    

    - name: Execute transmogrifier script on the slaves
      become: true
      become_user: root
      shell: "/home/ec2-user/script.sh"
```
</details>

<details markdown=1><summary markdown="span">Ansible .cfg</summary>

``` yml
[defaults]
remote_user = ec2-user 
inventory = inventory 
private_key_file = ~/.ssh/test_delete.pem
```
</details>

<details markdown=1><summary markdown="span">Launching</summary>

``` tf
# terraform template:
terraform init
terraform validate
terraform plan
terraform apply
terraform destroy

# ansible:
# ad-hoc command to launch the script:
ansible all -m script -a "/home/ec2-user/environment/Terraform/script.sh &" --become --become-user=ec2-user
# ad-hoc command to ping all the instances
ansible all --key-file ~/.ssh/test_delete.pem -i inventory -m ping -u ec2-user

# launch the playbook:
ansible-playbook playbook.yml
```
</details>

<details markdown=1><summary markdown="span">Extra info</summary>

``` sh
1. create key pairs on a master machine 
    ssh-keygen -t rsa -b 2048

2. import public key into the ec2 console
    aws ec2 import-key-pair --key-name "test_delete" --public-key-material fileb://~/.ssh/test_delete.pem

3. install ansible on a master machine
    sudo yum update -y
    sudo amazon-linux-extras install ansible2 -y
```
</details>

---


<img width="1024" alt="Screenshot 2023-02-19 at 20 11 21" src="https://user-images.githubusercontent.com/104728608/219972747-c9334130-8beb-4837-b3f1-84464838d4a3.png">
