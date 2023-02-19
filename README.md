# intrasonics_transmogrifier_task


<img width="1024" alt="Screenshot 2023-02-19 at 16 30 51" src="https://user-images.githubusercontent.com/104728608/219961551-35e4cb7a-e0dd-46da-b524-69d0bf253d1c.png">

---

## Bash <br>


<details markdown=1><summary markdown="span">Script</summary>

```
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

```
#!/bin/bash

crontab -l | { cat; echo "*/1 * * * * bash /home/ec2-user/script.sh"; } | crontab -

crontab -l | { cat; echo "*/1 * * * * tar -czvf /home/ec2-user/Archives/archive_$(hostname | cut -d '.' -f 1)_$(date +\%Y\%m\%d_\%H\%M\%S).tar.gz /home/ec2-user/Transmogrified/*"; } | crontab -

crontab -l | { cat; echo "0 0 * * 0 scp /home/ec2-user/Transmogrified/* user@remote.server:/path/to/remote/directory/"; } | crontab -
```
</details>

<details markdown=1><summary markdown="span">Pre-requisites to launch</summary>

```
```
</details>

---

## Python <br>

<details markdown=1><summary markdown="span">Script</summary>

```
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

```

```
</details>

<details markdown=1><summary markdown="span">Pre-requisites to launch</summary>

```

```
</details>

---

## Optional (creating the infrastructure to host and launch the script - combination of Terraform and Ansible)

<details markdown=1><summary markdown="span">Terraform script</summary>

```

```
</details>

<details markdown=1><summary markdown="span">Ansible script</summary>

```

```
</details>

<img width="1024" alt="Screenshot 2023-02-19 at 17 45 01" src="https://user-images.githubusercontent.com/104728608/219965610-eaa9b242-b6c2-439c-833c-800524c1d39f.png">

