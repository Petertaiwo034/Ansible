# Ansible
Ansible is agentless, it does not require a client software before it can manage a host. 
Ansible can be used for
1. UsersMGT
2. FileMGT
3. ServicesMGT
4. PackagesMGT
Ansible is an open source configuration management, deployment tool and also for provisioning tools maintained by Redhat. 
It can manage more than 50 server  of dbserver, webservers and appserver at tthe same time 

Ansible installation in ubuntu
sudo addsuser ansible
echo "ansible ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ansibe
sudo su - ansible
sudo apt-add-repository ppa:ansible/ansible
sudo apt install ansible -y

when you run ansible --version
You  will get the config file path= /etc/ansible/ansible.cfg
                   host file path= /etc/ansible/hosts
  Ansible concepts are:
1.  Ansible commands
         To check if serves are alive or not= ansible all -m ping ( if I want to check my dbserver is okay I will run ansible db -m ping)
         To check for memeory of the ansible server itself " ansible localhost -m command -a "free -m"
         To check in the webserver "ansible web -m command -a "df -h"
2. modueles
         ping = ansible all -m ping
         
         command=  ansible all -m command -a "ping"
         
         shell = ansible all -m shell -a "ping"
         
         service/systemd/systemctl/: ansible db -m service -a "name=ssd state=restarted"
                                     ansible db -m systemd -a "name=httpd state=absent"
                                     
         copy= ansible app -m copy -a "src=/home/ansible/app.war dest=/opt/tomcat/webapps/"
               ansible web -m copy -a "src=/home/ansible/index.html dest=/var/www/html/"
               
         yum/apt/package = ansible app -m yum -a "name=httpd state=present"
                           ansible app -m yum -a "name=httpd state=absent"
         run a"nsible-doc -i" to see all ansible modules 
         
3. Playbooks: it is configuration script written in yml. It contain plays and tasks those tasks will be executed in the hosts
you can vi into a yaml file and write a playbook
After you done you can check if the playbook you wrote is good by running " ansible-playbook apache.yml(name of the file you vi into) --syntax-check
if something was wrong with it you get an "error" message
To run the palybook use the command "ansible-playbook apache.yml(name of the file you vi into)"

apache.yml  
- hosts: db  
  become: true
  tasks: 
  - name: install apache
    yum: name=httpd state=present
  - name: start apache
    service: name=httpd state=started   
  - name: copy index file  
    copy: 
      src: index.html
      dest: /var/www/html/  
- hosts: all 
  become: true
  tasks:
  - name: install vim 
    package: 
      name: vim 
      state: present
- hosts: localhost 
  tasks: 
  - name: check system resources
    shell: df -h


How to Create Inventory file
IP Address ansible_user=ec2-user ansible_ssh_private_key_file=/tmp/ansible.pem
This is the way will write all our IP Address in the inventory file. if the instance you created is ubuntu then it will be ansible_user=ubuntu and make sure 
the key you will be using to created all the server will be the same
Change the permission of the file to ansible user using
run the command ls -l /etc/ansible/hosts to see who own the file
sudo chown ansible -R /etc/ansible/

Now vi into /etc/ansible/hosts and paste this all your host Ip 

[db]
    54.175.186.47 ansible_user=ec2-user  ansible_ssh_private_key_file=/tmp/ansible.pem 
    172.31.81.212 ansible_user=ec2-user  ansible_ssh_private_key_file=/tmp/ansible.pem

[web]
    172.31.87.249 ansible_user=ubuntu    ansible_ssh_private_key_file=/tmp/ansible.pem
    172.31.82.102 ansible_user=ec2-user  ansible_ssh_private_key_file=/tmp/ansible.pem
    54.84.32.131  ansible_user=ec2-user  ansible_ssh_private_key_file=/tmp/ansible.pem

[app]
    52.91.235.177 ansible_user=ubuntu    ansible_ssh_private_key_file=/tmp/ansible.pem
    172.31.87.249 ansible_user=ubuntu    ansible_ssh_private_key_file=/tmp/ansible.pem

you can also user a sever username and password
[k8s]
    172.31.87.249 ansible_user=kops    ansible_password=your password you created

You can also create a custom host file by just vi into host(this can be any name) and paste all your host IP address in there just like it's above
when you want to run any command using the custom host file pass "-i name of the custom file (-i host)"


vi  /etc/ansible/ansible.cfg
You will see a link in there to see the full file on github
Copy the file and paste in the vi /etc/ansible/ansible.cfg
Now search for the host key to seach without pressing the the I for insert write (/host_key) at the bottom of the file
Uncomment the line with the host key by deleting the # (pounds) commands
Now when you run ansible db -m ping it won't ask for permission any more
You have to create the ansible key in the /tmp
vi into /tmp/ansible.pem and paste the key in there
change the permission of the key using "sudo chmod 400 /tmp/ansible.pem

Note to run any thing in ansible that need elevated right and permission at th end of the command just pass "-b"


How to export ssh into which host manually/locally
you will run ssh-keygen to create the ssh
To create more secure key use the command "ssh-keygen -t ed25519 -c
ls -a  you will see .ssh, cd into .ssh and you will see the private and public sshh
To copy the ssh into which ip address then you will use the command "ssh-copy-id ec2-user@the ipaddress"
Instead of running the command this can be done using the playbook. You must have first included all your Ipaddresses and keys to the default host path and run the playbook.
172.31.87.249 ansible_user=ubuntu    ansible_ssh_private_key_file=/tmp/ansible.pem


After the playbook has been run successful, you can go back to the default hosts path and put # in from of the users and keys. You don't need the keys anymore because the public key from the .ssh has been deployed using the playbook to all server. 
172.31.87.249 #ansible_user=ubuntu    ansible_ssh_private_key_file=/tmp/ansible.pem

This is the playbook to deploy the key to all server so you can access them hence forth without passing key
- hosts: all
  become: true
  tasks:
  - name: Create Ansible User
    user:
      name: ansible
      create_home: true
      shell: /bin/bash
      comment: "Ansible Management Account"
      expires: -1
      password: "{{ 'DevOps@2020' | password_hash('sha512','A512') }}"
  - name: Deploy Local User SSH Key
    authorized_key:
      user: ansible
      state: present
      manage_dir: true
      key: "{{ lookup('file', '/home/ansible/.ssh/id_rsa.pub') }}"
  - name: Setup Sudo Access for Ansible User
    copy:
      dest: /etc/sudoers.d/ansible
      content: 'ansible ALL=(ALL) NOPASSWD: ALL' 
      validate: /usr/sbin/visudo -cf %s
  -  name: Disable Password Authentication
     lineinfile:
        dest=/etc/ssh/sshd_config
        regexp='^PasswordAuthentication'
        line="PasswordAuthentication no"
        state=present
        backup=yes
     notify:
       - restart ssh
  handlers:
  - name: restart ssh
    service:
      name=sshd
      state=restarted
     
     This will enable you to ssh into the server without using any key

      
      
4. Roles
5. Loops
6. Condition
7. Variable: This is use to pass a password in ansible. Foe example if using a password to access a user in the host file, you wont write the password itself out in the host file. It will be written in variable ed "{{password}}" and you will create a variable group_var directory in etc/ansible path using mkdir etc/ansible/group_var then create a file in the group_var by vi  etc/ansible/group_var/all_var write Password as it was written in the host file and the password in front of it e.g password= admin if "{{Name}}" was the variable written in the  host file then in the all_var file it will be written as Name = admin
To use imperative method to pass it
ansible kops -m ping -i hosts --extra-vars "password=admin"

using playbook
vars.yml 
---
- hosts: db
  vars:
    name: From Playbook
    password: dev@123
  tasks:
  - name:  demo vars
    debug:
      msg: "{{name}}"
  - name: vars demo2
    debug:
      msg: "{{password}}"


  ansible-playbook vars.yml --extra-vars "password=admin"
  ansible-playbook apache.yml  --extra-vars "password=admin"
---

Variables:
  1. runtime variables 
  2. playbook variables 
  3. host_vars variables 
  4. group_vars variables

ansible kops -m ping -i hosts --extra-vars "password=admin123"
 mkdir /etc/ansible/group_vars/
 mkdir /etc/ansible/db_vars/
 mkdir /etc/ansible/app_vars/
 mkdir /etc/ansible/all_vars/

ANSIBLE VAULT.:
=============== 
stores confidential or secrets  data.   

hashicorp VAULT.:

It is use to create and store secrets in ansible.

ansible-vault encrypt /etc/ansible/group_vars/all.yml (this will encrypt  the password in the all.yml file)

ansible-vault decrypt /etc/ansible/group_vars/all.yml

ansible-vault view /etc/ansible/group_vars/all.yml
ansible-vault edit /etc/ansible/group_vars/all.yml
ansible-vault rekey /etc/ansible/group_vars/all.yml
ansible-vault create  /etc/ansible/group_vars/all.yml

ansible k8s -m ping --ask-vault-pass
ansible k8s -m shell -a "kubectl get po " --ask-vault-pass (it can be run like this and this will ask you to enter the password encrypted)

ansible k8s -m ping --vault-password=admin ( You are passing the password directly here)
ansible-vault decrypt /etc/ansible/group_vars/all.yml --vault-password=vaultpass


Footer
© 2022 GitHub, Inc.
Footer navigation
Terms
