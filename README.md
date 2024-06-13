# CS312Final

## Requirements
The following requirements are needed to run the ansible-playbook:
- Amazon CLI
  - amazon.aws
- AWS Academy Learner Lab
- Ansible
- boto3
- git

(see resources section for information on requirements)

## Overview
This assignment's purpose was to automatically deploy a Minecraft server. I decided to use Ansible for this assignment. The playbook contains two plays: deploying the EC2 instance and setting up the Minecraft Server.

### Playbook Play 1 Explanation
```
- name: Create Minecraft Server on AWS
  hosts: localhost
  gather_facts: yes

  vars:
    aws_region: "us-west-2"
    key_name: "minecraft"
    public_key_path: "~/.ssh/minecraft.pub"
    security_group: "minecraft"
    instance_type: "t3.small"
    ami_id: "ami-0b20a6f09484773af"
    minecraft_port: "25565"

  tasks:
    - name: Create AWS Key Pair
      amazon.aws.ec2_key:
        name: "{{ key_name }}"
        key_material: "{{ lookup('file', public_key_path) }}"
        region: "{{ aws_region }}"
      register: keypair

    - name: Create Security Group
      amazon.aws.ec2_security_group:
        name: "{{ security_group }}"
        description: "Secuirty Group for Minecraft Server"
        region: "{{ aws_region }}"
        rules:
          - proto: tcp
            ports:
              - "{{ minecraft_port }}"
              - 22
            cidr_ip: "0.0.0.0/0"

    - name: Launch EC2 Instance
      amazon.aws.ec2_instance:
        name: "Minecraft Server"
        region: "{{ aws_region }}"
        key_name: "{{ key_name }}"
        instance_type: "{{ instance_type }}"
        image_id: "{{ ami_id }}"
        wait: yes
        security_group: "{{ security_group }}"
        network:
          assign_public_ip: true
      tags: ['create_ec2']
      register: ec2

    - name: Add Minecraft Server to Hosts
      ansible.builtin.add_host:
        name: "{{ ec2.instances[0].public_ip_address }}"
        groups: Minecraft_Server
```
#### Create AWS Key Pair
This task creates a key pair using the *minecraft* ssh key within the region so that Ansible can communicate with the EC2 instance once the instance is created.

#### Create Security Group
This task creates the security group that the EC2 instance will use. By default, there is no access to an EC2 instance, so in this security group, we open ports 22 and 25565. Port 22 is used for ssh so that Ansible can do its magic, and port 25565 is used for Minecraft so that external users can play on the server.

#### Launch EC2 Instance
This task launches the EC2 Instance using the key pair and security group created in the previous tasks. Along with the key pair and security group, we tell AWS what image the server will use and that the EC2 instance will need a public IP address.

#### Add Minecraft Server to Hosts
This task takes the EC2 instance's public IP address and adds it to the Minecraft_Server group for use in the next play.

### Playbook Play 2 Explanation
```
- name: Setup Minecraft Server
  hosts: Minecraft_Server
  user: ec2-user
  become: true
  become_user: root
  gather_facts: no

  tasks:
    - name: Wait for EC2 Instnace to Start
      ansible.builtin.wait_for_connection:
        timeout: 600

    - name: Update EC2 Instance
      dnf:
        name: "*"
        state: latest

    - name: Install Java
      dnf:
        name: "java-21-amazon-corretto-devel"
        state: latest

    - name: Install Cron
      dnf:
        name: "cronie"
        state: latest

    - name: Enable and Start Cron
      ansible.builtin.systemd:
        name: crond.service
        enabled: yes
        state: started

    - name: Create Cron Job to Start Minecraft Server
      ansible.builtin.cron:
        name: "Start Minecraft Server"
        special_time: reboot
        job: "cd /minecraft-server && java -Xmx1G -Xms1G -jar /minecraft-server/server.jar nogui"

    - name: Create /minecraft-server Directory
      ansible.builtin.file:
        path: /minecraft-server
        state: directory
        mode: "7777"

    - name: Download Minecraft Server
      ansible.builtin.get_url:
        url: https://piston-data.mojang.com/v1/objects/145ff0858209bcfc164859ba735d4199aafa1eea/server.jar
        dest: /minecraft-server/server.jar

    - name: Initialize Minecraft Server
      ansible.builtin.command: java -Xmx1G -Xms1G -jar server.jar nogui
      args:
        chdir: /minecraft-server

    - name: Accept Minecraft Server EULA
      ansible.builtin.lineinfile:
        path: /minecraft-server/eula.txt
        regexp: '^eula='
        line: eula=true

    - name: Reboot Server
      ansible.builtin.reboot:
        reboot_timeout: 600
        pre_reboot_delay: 0
        post_reboot_delay: 30
```
#### Wait for EC2 Instance to Start
This task makes the play wait until the EC2 instance has fully started to avoid timing out on the following tasks.

#### Update EC2 Instance
This task updates all the packages on the EC2 instance. It is included for best practice but all the packages should already be up to date since this is a fresh install using Amazon Linux

#### Install Java
This task installs Java 21, which is what Minecraft needs to run, onto the server

#### Install Cron
This task installs cron so that we can automagically start the Minecraft server if the EC2 instance is restarted

#### Enable and Start Cron
By default, cron is not enabled or running after being installed. This task enables cron to start on startup and starts cron.

#### Create Cron Job to Start Minecraft Server
This task creates the cron job to start the Minecraft server if the EC2 instance is rebooted

#### Create /minecraft-server Directory
To keep our system organized a minecraft-server directory is created to hold the Minecraft server files.

#### Download Minecraft Server
This task installs the minecraft server (v1.20.6) into the minecraft-server dierectory previously created.

#### Initialize Minecraft Server
This task runs the server.jar for the first time which will create the configuration files and provide the eula we will need to accept.

#### Accept Minecraft Server EULA
This task changes the status of the eula.txt from false to true (accepting the eula) so that the server will run.

#### Reboot Server
This task reboots the server, and the Minecraft server will start automatically after the reboot, thanks to the cron job.

## Tutorial
This tutorial assumes you are on a Linux machine.
1. Set up AWS environment variables  
*note: AWS environment variables are located in your learner lab. Go to the **AWS Details** tab and click **Show** on **AWS CLI:***
```
$ export AWS_ACCESS_KEY_ID={your access key here}
$ export AWS_SECRET_ACCESS_KEY={your secret access key here}
$ export AWS_SESSION_TOKEN={your session access key here}
```
2. Clone the repo  
*note: the following command makes your current directory the repo*  
```
$ git clone https://github.com/rmkirk21/CS312Final .
```
3. Create your SSH key (note: name the key *minecraft*)
```
$ cd ~/.ssh
$ ssh-keygen
```
4. Run the playbook  
*note: run the following command in the directory you put the repo*  
```
$ ansible-playbook -i inventory.ini playbook.yml
```
5. After the playbook finishes the public IP of the Minecraft server can be found in the **Play Recap** or you can find the public IP in the AWS Management Console.

## Resources
The following is a list of resources used for this project:
- Amazon CLI - https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
- amazon.aws - https://galaxy.ansible.com/ui/repo/published/amazon/aws/docs/
- Ansible - https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html
- boto3 - https://boto3.amazonaws.com/v1/documentation/api/latest/guide/quickstart.html
- Minecraft Server - https://www.minecraft.net/en-us/download/server
- git - https://git-scm.com/book/en/v2/Getting-Started-Installing-Git
