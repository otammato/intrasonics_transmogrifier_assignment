# intrasonics_transmogrifier_assignment (monitoring solution)

## Objectives:<br>
Monitor the existing solution. The recently added features make the existing solution expected to behave in the following way: 
-  Independent processes will be started up to serve individual client requests. These should all be started by the transmogrifier user. 
-  Each of these processes may run for an extended period, depending on how much work is required to serve their specific request. This workload is expected to be highly variable. 
-  A dedicated Transmogrified/ folder has been created on each server, which these processes will occasionally write to. 

You have been asked to create a solution which monitors the following at regular intervals:<br>
-  Processes currently running on the machine. 
-  Contents of the Transmogrified/ folder. 
Write a script in the language of your choosing, which will be used to achieve the above. 

<img width="1024" alt="Screenshot 2023-02-19 at 16 30 51" src="https://user-images.githubusercontent.com/104728608/219961551-35e4cb7a-e0dd-46da-b524-69d0bf253d1c.png">

## Description of the logic of the created script:
The solution involves:
 
 #### 1. Design of the ETL process up to the point of loading the files onto the master server.
&nbsp;&nbsp;&nbsp;&nbsp;Since there were no specific requirements for the ETL process or a representation layer, the process is only designed up to the point of &nbsp;&nbsp;&nbsp;&nbsp;loading the files onto the master server. Instead, the data could be loaded into a Data Warehouse or a database and then connected to &nbsp;&nbsp;&nbsp;&nbsp;a representation layer, allowing for the creation of dashboards using tools such as MS Power BI, Tableau, or AWS Quicksight or some &nbsp;&nbsp;&nbsp;&nbsp;specific monitoring tools.

 #### 2. Design and develop the main script
- to monitor and log the processes and content of the Transmogrified/ folder to .txt files 
- to place the created log files at the same (for simplicity) Transmogrified/ folder
 #### 3. Design and develop the script for launching the following cron tasks:
- at regular intervals launch the main script
- at regular intervals create .tar archives of Transmogrified/ folder's content and place them to the Archives/ folder
- at regular intervals backup the .tar archives to a "master" server managed by a DevOps
 #### 4. [Optionally] Terraform + Ansible infrastructure to test the script


## Assumptions:
- The task can be done by any one of the tools being used (Bash or Python or Terraform or Ansible). I am demonstrating different variations and combinations of them for demo purposes.
- To keep things work smoothly, I am giving resources maximum rights. In an actual production environment, I would apply the principle of least privilege.
- To keep things simple, I am creating EC2 instances in the default VPC. In an actual production environment, I would create the separate VPC, subnets, route tables, IGW, NACLs and would apply additional level of the network and systems hardening.
- All the necessary functionality can be easily accomplished by utilizing a dedicated monitoring tool such as AWS CloudWatch. I am using scripting for demonstration purposes only.
- Since there were no specific requirements for the ETL process or a representation layer, the process is only designed up to the point of loading the files onto the master server. Instead, the data could be loaded into a Data Warehouse or a database and then connected to a representation layer, allowing for the creation of dashboards using tools such as MS Power BI, Tableau, or AWS Quicksight or some specific monitoring tools.

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
        echo "Transmogrified/ folder created, please relaunch the script"
        files_list=()

        # Set the owner and group for Transmogrified/ folder and give read, write, and execute permissions recursively
        sudo chown -R transmogrifier:ec2-user Transmogrified/
        sudo chmod -R g+rwx Transmogrified/
        sudo chmod -R o-rx Transmogrified/
    fi
    
    # Check if Archives/ exists and create it if not
    if [ ! -d "Archives/" ]; then
        # Create the Archives/ folder
        mkdir "Archives/"
        echo "Archives/ folder created, please relaunch the script"

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
# this scenario launches if "transmogrifier" does not exist and creates the user and adds him in ec2-user group
else
    sudo useradd "$user" 
    sudo usermod -a -G ec2-user "$user"
    echo ""$user" created, please create a password to the user and relaunch the script"
fi
```

</details>

<details markdown=1><summary markdown="span">Cron script</summary>

``` sh
#!/bin/bash

# This block adds a cron job that executes the "script.sh" file every minute, increase the interval if needed
crontab -l | { cat; echo "*/1 * * * * bash /home/ec2-user/script.sh"; } | crontab -

# This block adds a cron job that creates a compressed archive of the files in the "/home/ec2-user/Transmogrified" directory every minute
# The archive file is named based on the current hostname and timestamp, increase the interval if needed
crontab -l | { cat; echo "*/1 * * * * tar -czvf /home/ec2-user/Archives/archive_$(hostname | cut -d '.' -f 1)_$(date +\%Y\%m\%d_\%H\%M\%S).tar.gz /home/ec2-user/Transmogrified/*"; } | crontab -

# This block adds a cron job that copies all the files in the "/home/ec2-user/Transmogrified" directory to a remote server every Sunday at midnight
# The destination directory on the remote server is specified as "/path/to/remote/directory/"
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
        print("Transmogrified/ folder created, please relaunch the script")
        files_list = []

        # Set the owner and group for Transmogrified/ folder and give read, write, and execute permissions recursively
        os.system("sudo chown -R transmogrifier:ec2-user Transmogrified/")
        os.system("sudo chmod -R g+rwx Transmogrified/")
        os.system("sudo chmod -R o-rx Transmogrified/")
    
    # Check if Archives/ exists and create it if not
    if not os.path.isdir("Archives/"):
        # Create the Archives/ folder
        os.makedirs("Archives/")
        print("Archives/ folder created, please relaunch the script")

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
    print(f"{user} created, please create a password to the user and relaunch the script")
```
</details>

<details markdown=1><summary markdown="span">Cron script</summary>

``` python3
#!/usr/bin/env python

import subprocess

# Add cron job to run script.sh every minute, increase the interval if needed
subprocess.run(['bash', '-c', 'echo "$(crontab -l ; echo \'*/1 * * * * bash /home/ec2-user/script.sh\') | crontab -"'])

# Add cron job to archive files every minute, increase the interval if needed
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

## Optional (Automatically create the infrastructure and launch the scripts - combination of Terraform, Ansible and Bash)

<details markdown=1><summary markdown="span">Terraform script</summary>

``` tf
# Terraform script that configures AWS provider for us-east-1 region with shared 
# credentials and config files, and launches an EC2 instance with the specified AMI, 
# instance type, and subnet, and attaches the security group.
# It also creates an "inventory" file for the following Ansible playbook and IPs as output

# for the simplicity I used the default VPC

# Configure AWS provider for us-east-1 region with shared credentials and config files
provider "aws" {
  region                    = "us-east-1"
  shared_config_files       = ["/home/ec2-user/.aws/config"]
  shared_credentials_files  = ["/home/ec2-user/.aws/credentials"]
}

# Retrieve availability zones in the specified state
data "aws_availability_zones" "available" {
  state = "available"
}

# Retrieve the latest Amazon Linux 2 AMI ID from AWS Systems Manager Parameter Store
data "aws_ssm_parameter" "current-ami" {
  name = "/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2"
}

# Create a default VPC with DNS hostnames and support enabled
resource "aws_default_vpc" "default" {
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = {
    Name = "Default VPC"
  }
}

# Create a subnet in the default VPC with a specific CIDR block and availability zone
resource "aws_subnet" "subnet_1" {
  vpc_id     = aws_default_vpc.default.id
  cidr_block = "172.31.98.128/25"  # change this to your desired CIDR block
  availability_zone = data.aws_availability_zones.available.names[0]  # use the first available AZ

  tags = {
    Name = "ansible-subnet_1"
  }
}

# Create a security group allowing inbound SSH and HTTP traffic and all outbound traffic
resource "aws_security_group" "ec2_security_group" {
  name        = "ec2-slave-security-group"
  description = "Allow SSH and HTTP access"
  vpc_id      = aws_default_vpc.default.id

  ingress {
    from_port   = 22 # for the system hardening never use the default port in production, set up another port in your system and then change it here
    to_port     = 22 # for the system hardening never use the default port in production, set up another port in your system and then change it here
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # change this to your IP for security
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # change this to your IP for security
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Launch an EC2 instance with the specified AMI, instance type, and subnet, and attach the security group
resource "aws_instance" "ansible_slave" {
  count = 3
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

# Create a local file containing the public IP addresses of the launched EC2 instances
resource "local_file" "slaves_ips" {
    content = format("%s\n%s\n%s",
  aws_instance.ansible_slave.*.public_ip[0],
  aws_instance.ansible_slave.*.public_ip[1],
  aws_instance.ansible_slave.*.public_ip[2]
  )
    filename = "inventory"
}


## this is to create only one slave_instance
# resource "local_file" "slaves_ips" {
#     content = aws_instance.ansible_slave.*.public_ip[0]

#   filename = "inventory"
# }

# indicate the IP's of the created controlled instances after completion
output "slaves_ips" {
  value = ["${aws_instance.ansible_slave.*.public_ip}"]
}
```
</details>

<details markdown=1><summary markdown="span">Ansible playbook</summary>

``` yml
# Ansible playbook to create user and folder on slaves
# and schedule script to run every minute, and archive files
# for transfer to a master server

---

# Define hosts, privilege escalation, and gather facts
- name: Create user and folder on slaves
  hosts: all
  become: true
  gather_facts: true

  # Define variables to be used later
  vars:
    user: transmogrifier

  # Define tasks to be performed
  tasks:
    # Create a user for running the script
    - name: Create transmogrifier user
      user:
        name: "{{ user }}"
        group: "ec2-user"
        state: present

    # Create the directory for storing the files to be processed
    - name: Create Transmogrified folder
      file:
        path: /home/ec2-user/Transmogrified
        state: directory
        owner: "{{ user }}"
        group: "ec2-user"
        mode: '0777' # this is only for testing purposes and should be limited in production
        
    # Create a directory to store archive files
    - name: Create a Tar folder for archives
      file:
        path: /home/ec2-user/Archives
        state: directory
        owner: "{{ user }}"
        group: "ec2-user"
        mode: '0777' # this is only for testing purposes and should be limited in production

    # Copy the script to the slave servers
    - name: Copy transmogrifier script to slave
      copy:
        src: /home/ec2-user/environment/Terraform/script.sh
        dest: /home/ec2-user/script.sh
        owner: "ec2-user"
        group: "ec2-user"
        mode: '0777' # this is only for testing purposes and should be limited in production
        
    # Schedule the script to run every minute
    - name: Create cron task to run script every minute
      ansible.builtin.cron:
        user: "ec2-user"
        name: "Run script every minute"
        minute: "*/1"
        job: "bash /home/ec2-user/script.sh"    
        
    # Schedule a weekly archive of the processed files
    - name: Create weekly tar archive of Transmogrified folder
      ansible.builtin.cron:
        user: "ec2-user"
        name: "Weekly tar archive of Transmogrified folder"
        job: "tar -czvf /home/ec2-user/Archives/archive_$(hostname | cut -d '.' -f 1)_$(date '+%Y%m%d_%H%M%S').tar.gz /home/ec2-user/Transmogrified/*"
        minute: "*/1"
        hour: "*"
        day: "*"
        month: "*"
        weekday: "*"

    # Transfer the monthly archives to the master server
    - name: Move monthly archives to the master server
      ansible.builtin.cron:
        user: "ec2-user"
        name: "Monthly archive transfer to master server"
        job: "rsync -avz /home/ec2-user/Archives/*.tar.gz root@ec2-3-80-47-11.compute-1.amazonaws.com"
        day: 1
        
    # Execute the script on the slave servers
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

<details markdown=1><summary markdown="span">Extra commands which might be useful</summary>

``` sh
# create key pairs on a master machine 
ssh-keygen -t rsa -b 2048

# import public key into the ec2 console
aws ec2 import-key-pair --key-name "test_delete" --public-key-material fileb://~/.ssh/test_delete.pem

# install ansible on a master machine
sudo yum update -y
sudo amazon-linux-extras install ansible2 -y
```
</details>

---


<img width="1024" alt="Screenshot 2023-02-19 at 20 11 21" src="https://user-images.githubusercontent.com/104728608/219972747-c9334130-8beb-4837-b3f1-84464838d4a3.png">
<br>

---

## Testing / Conclusion

Launching scripts:
<p align="center" >
  <img width="1024" alt="Screenshot 2023-02-20 at 09 16 28" src="https://user-images.githubusercontent.com/104728608/220063636-6c548dba-9bce-41f3-911c-add1c1d7fc2f.png">
</p>

<p align="center" >
  <img width="1024" alt="Screenshot 2023-02-20 at 09 23 10" src="https://user-images.githubusercontent.com/104728608/220065309-fb66a565-fe8d-4ff5-a4da-e520a90f93d7.png">
</p>

<p align="center" >
  <img width="1024" alt="Screenshot 2023-02-19 at 23 32 09" src="https://user-images.githubusercontent.com/104728608/219982081-395a37b1-e446-4f5d-a5ff-f76f139cd506.png">
</p>

Creating Terraform Resources:
<p align="center" >
  <img width="1024" alt="Screenshot 2023-02-20 at 09 28 08" src="https://user-images.githubusercontent.com/104728608/220066373-37c041a2-725f-4023-8b81-ada2e478dbfd.png">
</p>


Launching Ansible playbook:
<p align="center" >
  <img width="1024" alt="Screenshot 2023-02-19 at 23 57 12" src="https://user-images.githubusercontent.com/104728608/219983498-5b75b3df-0aab-424d-87a5-2db8885b38cc.png">
</p>





