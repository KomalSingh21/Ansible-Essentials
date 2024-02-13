
Ansible Lab Steps
=================
Login to AWS Console

## Lab 1: Installation and Configuration of Ansible

Launch a RHEL 9 instance in us-east-1. 
Choose t2.micro. 
In security group, allow SSH (22) and HTTP (80) for all incoming traffic. 
Add Tag Name: Ansible-ControlNode

Once the EC2 is up & running, SSH into one of it and set the hostname as 'Control-Node'. 
```
sudo hostnamectl set-hostname Control-Node
```
or you can type 'bash' and open another shell which shows new hostname.

Update the package repository with latest available versions
```
sudo yum check-update
```

Install latest version of Python. 
```
sudo yum install python3-pip -y 
```
```
python3 --version
```
```
sudo pip3 install --upgrade pip
```

Install awscli, boto, boto3 and ansible
Boto/Boto3 are AWS SDK which will be needed while accessing AWS APIs
```
sudo pip3 install awscli boto boto3
```
```
sudo pip3 install ansible==4.10.0
```
```
pip show ansible
```

Authorize aws credentials
```
aws configure
```
#### Enter the Credentials as below. Example:
| **Access Key ID** | **Secret Access Key** |
| ----------------- | --------------------- |
| AKIAXMWJXSSHRD27T6SC | H4Vh0U5oenKfmJ/+FEUcbaGbDjcnGAmZvQLX7zTT |

Install wget so that we can download playbooks from the training material repository 
```
sudo yum install wget -y
```

Download the script using wget
```
wget https://ansible-script.s3.us-west-1.amazonaws.com/ansible_script.yaml
```

Execute the script
```
ansible-playbook ansible_script.yaml
```

Once you get the ip addresses, do the following:
```
sudo mkdir /etc/ansible
```
```
sudo vi /etc/ansible/hosts
```

Add the prive IP addresses, by pressing "INSERT" 
```
node1 ansible_ssh_host=node1-private-ip ansible_ssh_user=ec2-user
node2 ansible_ssh_host=node2-private-ip ansible_ssh_user=ec2-user
```
e.g. node1 ansible_ssh_host=172.31.14.113 ansible_ssh_user=ec2-user
     node2 ansible_ssh_host=172.31.2.229 ansible_ssh_user=ec2-user


Save the file using "ESCAPE + :wq!"

list all managed node ip addresses.
```
ansible all --list-hosts
```

### SSH into each of them and set the hostnames.
```
ssh ec2-user@< Replace Node 1 IP >
```
```
sudo hostnamectl set-hostname managed-node-1
```
```
exit
```
```
ssh ec2-user@< Replace Node 2 IP >
```
```
sudo hostnamectl set-hostname managed-node-2
```
```
exit
```

### Use ping module to check if the managed nodes are able to interpret the ansible modules
```
ansible all -m ping
```



## Lab 2: Exploring Ad-Hoc Commands

```
sudo vi /etc/ansible/hosts
```

Add the given line, by pressing "INSERT" 
add localhost and add the connection as local so that it wont try to use ssh
```
localhost ansible_connection=local
```
**save the file using** `ESCAPE + :wq!`

In real life situations, one of the managed node may be used as the ansible control node.
In such cases, we can make it a managed node, by adding localhost in hosts inventory file.


get memory details of the hosts using the below ad-hoc command
```
ansible all -m command -a "free -h"
```
OR
```
ansible all -a "free -h"
```

Create a user ansible-new in the 2 nodes + the control node
This creates the new user and the home directory /home/ansible-new
```
ansible all -m user -a "name=ansible-new" --become
```

lists all users in the machine. Check if ansible-new is present in the managed nodes / localhost
```
ansible node1 -a "cat /etc/passwd"
```

List all directories in /home. Ensure that directory 'ansible-new' is present in /home. 
```
ansible node2 -a "ls /home"
```

Change the permission mode from '700' to '755' for the new home directory created for ansible-new
```
ansible node1 -m file -a "dest=/home/ansible-new mode=755" --become
```

Check if the permissions got changed
```
ansible node1 -a "sudo ls -l /home"
```

Create a new file in the new dir in node 1
```
ansible node1 -m file -a "dest=/home/ansible-new/demo.txt mode=600 state=touch" --become
```

Check if the permissions got changed
```
ansible node1 -a "sudo ls -l /home/ansible-new/"
```

Add content into the file
```
ansible node1 -b -m lineinfile -a 'dest=/home/ansible-new/demo.txt line="This server is managed by Ansible"'
```

check if the lines are added in demo.txt
```
ansible node1 -a "sudo cat /home/ansible-new/demo.txt"
```

You can remove the line using parameter state=absent
```
ansible node1 -b -m lineinfile -a 'dest=/home/ansible-new/demo.txt line="This server is managed by Ansible" state=absent'
```

check if the lines are removed from demo.txt
```
ansible node1 -b -a "sudo cat /home/ansible-new/demo.txt"
```

Now copy a file from ansible-control node to host node 1
```
touch test.txt
```
```
echo "This file will be copied to managed node using copy module" >> test.txt
```
```
ansible node1 -m copy -a "src=test.txt dest=/home/ansible-new/test" -b
```
--become can be replaced by -b

check if the file got copied to managed node.
```
ansible node1 -b -a "sudo ls -l /home/ansible-new/test"
```
```
sudo vi /etc/ansible/hosts
```

Remove the below line from hosts inventory file. 
localhost ansible_connection=local

**save the file using** `ESCAPE + :wq!`


## Lab 3: Implementing Ansible Playbook
```
mkdir ansible-labs && cd ansible-labs
```
---------------------------------------------------------------------
### Task 1: Playbook to install apache web server
```
vi install-apache-pb.yml
```
Add the given content, by pressing "INSERT"
```
---
- name: This play will install apache web servers on all the hosts
  hosts: all
  become: yes
  tasks:
    - name: Task1 will install httpd using yum
      yum:
        name: httpd
        #local cache of the package information available from the repositories configured on the system
        update_cache: yes
        state: latest
    - name: Task2 will upload custom index.html into all hosts
      copy:
        src: /home/ec2-user/ansible-labs/index.html
        dest: /var/www/html
    - name: Task3 will setup attributes for file
      file:
        path: /var/www/html/index.html
        owner: apache
        group: apache
        mode:  0644
    - name: Task4 will start the httpd
      service:
        name: httpd
        state: started
```
**save the file using** `ESCAPE + :wq!`

State as 'Present' and 'Installed' are used interchangeably. They both do the same thing i.e. It 
will ensure that a desired package is installed. Whereas State as 'Latest' means in addition
to installation, it will go ahead and update if it is not of the latest available version.

Now lets create an index.html file to be used as the landing page for the web server.
In task2 of the above playbook, this 'index.html' will be copied to the document root of the 
httpd server.
```
vi index.html
```

Add the given content, by pressing "INSERT" 
```
<html>
  <body>
  <h1>Welcome to Ansible Training from CloudThat</h1>
  </body>
</html>
```
**save the file using** `ESCAPE + :wq!`

Now run the playbook so that it installs httpd to all the hosts and index.html is copied from 
the ansible-control
```
ansible-playbook install-apache-pb.yml
```
```
curl <private_ip of node1> 
```
```
curl <private_ip of node2>
```

---------------------------------------------------------------------------------------
### Task 2: Uninstall apache web server

With slight modification, we can change the playbook to uninstall apache (self exercise)
```
cp install-apache-pb.yml uninstall-apache-pb.yml
```
```
vi uninstall-apache-pb.yml
```
Retain only first task. Replace 'state: latest' with 'state: absent'

Check if the playbook is ok
```
ansible-playbook uninstall-apache-pb.yml --check
```

Now run the playbook and check in the browser if the web page exists.
```
ansible-playbook uninstall-apache-pb.yml
```

Check the browser with the corresponding IPv4 DNS name and ensure webpage is removed.


## Lab 4: Exploring more on Ansible Playbooks
-----------------------------------------------------------------------------------
### Task 1: Creating a playbook and adding content to a file
```
cd /home/ec2-user/ansible-labs
```
```
vi putfile.yml
```

Add the given content, by pressing "INSERT" 
```
---
- hosts: all
  become: yes
  tasks:
    - name: Creating a new user cloudthat
      user:
        name: cloudthat
    - name: Creating a directory for the new user
      file:
        path: /home/cloudthat
        state: directory
```


**save the file using** `ESCAPE + :wq!`

Now run the playbook as follow
```
ansible-playbook putfile.yml
```
Now modify the playbook to add another child directory and a file inside cloudthat's home directory
```
vi putfile.yml
```
Add the given content, by pressing "INSERT" 
```
---
- hosts: all
  become: yes
  tasks:
    -   name: Creating a new user cloudthat
        user:
          name: cloudthat
    -   name: Creating a directory for the new user
        file:
          path: /home/cloudthat
          state: directory
    -   name: creating a folder named ansible
        file:
          path: /home/cloudthat/ansible
          state: directory
    -   name: creating a file within the folder ansible
        file:
          path: /home/cloudthat/ansible/hello.txt
          state: touch
    -   name: Changing owner and group with permission for the file within the folder named ansible
        file:
          path: /home/cloudthat/ansible/hello.txt
          owner: root
          group: cloudthat
          mode: 0665
    -   name: adding a block of string to the file created named hello.txt
        blockinfile:
          path: /home/cloudthat/ansible/hello.txt
          block: |
            This is line 1
            This is line 2

```
**save the file using** `ESCAPE + :wq!`
if you use '|' then the new line characters will be retained.

we can execute it step by step
```
ansible-playbook putfile.yml --step
```

check if the lines are added in hello.txt
```
ansible all -a "sudo cat /home/cloudthat/ansible/hello.txt"
```

Check if the permissions / ownership is as per the playbook
```
ansible all -a "sudo ls -l /home/cloudthat/ansible/"
```


## Lab 5: Implementing Ansible Variables

---------------------------------------------------------------------------------------------
### Task 1: Configuring packages in ansible using variables
```
cd ~/ansible-labs/
```
```
mkdir file && cd file
```
```
vi implement-vars.yml
```

We are using variables, hostname, package1, package2, portno, and path. We are not directly specifying 
the value of these variables at their places

Add the given content, by pressing "INSERT" 
```
---
- hosts: '{{ hostname }}'
  become: yes
  vars:
    hostname: all
    package1: httpd
    destination: /var/www/html/index.html 
    source: /home/ec2-user/ansible-labs/index.html
  tasks:
    - name: Install defined package
      yum:
        name: '{{ package1 }}'
        update_cache: yes
        state: latest
    - name: Start desired service
      service:
        name: '{{ package1 }}'
        state: started
    - name: copy required index.html to the document folder for httpd. 
      copy:
        src: '{{ source }}'
        dest: '{{ destination }}'
```

**save the file using** `ESCAPE + :wq!`

create index.html in /home/ec2-user/ansible-labs/
```
vi index.html
```
```
<html>
  <body>
  <h1>Welcome Everyone to Ansible Training</h1>
  </body>
</html>
```

**save the file using** `ESCAPE + :wq!`
```
ansible-playbook implement-vars.yml
```

Go to aws console. Copy 'Public IPv4 DNS' name from the instance details.
Paste that into a browser and observe that the web page is coming up
The web page should display the message "This is the Selected Home Page"

-------------------------------------------------------------------------------------------------
### Task 2 : Create an alternate index_new.html file

create index1.html in ~/ansible-labs/
```
cd /home/ec2-user/ansible-labs/
```
```
vi index1.html
```
```
<html>
  <body>
  <h1>This is the alternate Home Page</h1>
  </body>
</html>
```
**save the file using** `ESCAPE + :wq!`
```
ansible-playbook implement-vars.yml --extra-vars "source=/home/ec2-user/labs/file/index1.html"
```

Check the home page on browser. It should show the new page now
It should show "This is the revised Home Page !!"

------------------------------------------------------------------------------------------
### Task 3 : Use separate variables file

Move variables to new separate file 
```
vi myvariables.yml
```
add the folowing in the myvariables.yml file.
```
---
hostname: all
package1: httpd
destination: /var/www/html/index.html
source: /home/ec2-user/ansible-labs/index.html
...
```
save the file using "ESCAPE + :wq!"

Add the given content, by pressing "INSERT" 

Now edit implement-vars.yml playbook. Replace vars block with vars_file reference.
```
vi implement-vars.yml
```
```
---
- hosts: '{{ hostname }}'
  become: yes
  vars_files:
    - myvariables.yml
  tasks:
    - name: Install defined package
      yum:
        name: '{{ package1 }}'
        update_cache: yes
        state: latest
    - name: Start desired service
      service:
        name: '{{ package1 }}'
        state: started
    - name: copy required index.html to the document folder for httpd. 
      copy:
        src: '{{ source }}'
        dest: '{{ destination }}'
```
**save the file using** `ESCAPE + :wq!`
```
$ansible-playbook implement-vars.yml
```
Check the home page on browser. 
It should show the original page with msg "This is the Selected Home Page"


## Lab 6 :Task Inclusion

Create a file named second.yml with below contents which has a list of tasks to install and start httpd (apache) service
```
vi second.yml
```
```
---
  - name: install the httpd package
    yum:
      name: httpd
      state: latest
      update_cache: yes

  - name: start the httpd service
    service:
      name: httpd
      state: started
      enabled: yes
```
**save the file using** `ESCAPE + :wq!`

----------------------------------------------------------------------------------------
Create another playbook named first.yaml, which has an inclusion for the earlier created task (second.yml) 
```
vi first.yml
```
```
---
- hosts: all
  gather_facts: no
  become: yes
  tasks:
  - name: install common packages
    yum:
      name: [wget, curl]
      state: present

  - name: inclue task for httpd installation
    include_tasks: second.yml
 ``` 

**save the file using** `ESCAPE + :wq!`
Execute the playbook named first.yaml using below command
```
ansible-playbook first.yml
```
------------------------------------------------------------------------------------------
Create another playbook named third.yaml as below
```
vi third.yml
```
```
---
- hosts: all
  gather_facts: no
  become: yes
  tasks:
  - name: install common packages
    yum:
      name: [wget, curl]
      state: present
    register: out

  - name: list result of previous task
    debug:
      msg: "{{ out.rc}}"

  - name: inclue task for httpd installation
    include_tasks: second.yml
    when: out.rc == 0
```
----------------------------------------------------------------------------------
Execute the playbook third.yaml
```
ansible-playbook third.yml
```

verify the installed packages on managed node by following commands#
```
ansible all -m command -a "yum list wget curl" -b
```

## Lab 7: Implementing Ansible Vault
```
cd ~/ansible-labs/
```
Now using the vault utility, encrypt the playbook that we created. Give the passwd of your choice
```
ansible-vault encrypt implement-vars.yml
```

Check if the encrypted file can be viwed using the cat command  
```
cat implement-vars.yml
```
Using below command, view the encrypted playbook by giving password that is set
```
ansible-vault view implement-vars.yml
```

For executing the encrypted playbook following command is used
```
ansible-playbook --ask-vault-pass implement-vars.yml
```

Now edit the playbook, add the below line to it and save the playbook
```
ansible-vault edit implement-vars.yml
```
Now if we want to change the password, we can do that with below command
Provide the old and new password to reset the password
```
ansible-vault rekey implement-vars.yml
```

Open the encrypted file and see the content using 
```
cat implement-vars.yml
```

decrypt the file
```
ansible-vault decrypt implement-vars.yml
```
```
cat implement-vars.yml
```

## Lab 8: Working with ansible functions
----------------------------------------------------------------------------
### Task 1: Loops with Ansible Playbook
```
vi looplab.yml
```
```
---
- hosts: all
  become: yes
  tasks:
   - name: creating users
     user:
       name: "{{ item }}"
       state: present
     with_items:
      - userX
      - userY
      - userZ
```
**save the file using** `ESCAPE + :wq!`

Execute the playbook
```
ansible-playbook looplab.yml
```
Verify if the users mentioned in the list were added by using an Ansible ad-hoc command
```
ansible all -a "tail -n 3 /etc/passwd"
```
-------------------------------------------------------------------------------------
### Task 2: Tags with Ansible Playbooks

Create and edit tagslabs.yml in the same labs directory 

```
vi tagslabs.yml
```
```
---
- hosts: all
  become: yes
  user: ec2-user
  connection: ssh
  gather_facts: no
  tasks:
    - name: Install telnet
      yum: pkg=telnet state=latest
      tags:
        - packages
    - name: Verifying telnet installation
      raw: yum list installed | grep telnet > /home/ec2-user/pkg.log
      tags:
        - logging
```

**save the file using** `ESCAPE + :wq!`

Execute the playbook
```
ansible-playbook tagslabs.yml
```
Run the playbook again, this time using tags. Notice that only the tasks associated with the mentioned tags are running
```
ansible-playbook -t "logging" tagslabs.yml
```
```
ansible-playbook -t "packages" tagslabs.yml
```
-----------------------------------------------------------------------------------
### Task 3: Prompts with Ansible Playbooks

Create and edit promptlab.yml in the same labs directory 
```
vi promptlab.yml
```
```
---
- hosts: all
  become: yes
  user: ec2-user
  connection: ssh
  vars_prompt:
    - name: pkginstall
      prompt: Which package do you want to install?
      default: telnet
      private: no
  tasks:
    - name: Install the package specified
      yum: pkg={{ pkginstall }} state=latest
```

**save the file using** `ESCAPE + :wq!`

Execute the playbook & you will be prompt for enter the package name which you want to install
If no package is mentioned, telnet is installed by default
```
ansible-playbook promptlab.yml
```

Verify if the specified package httpd is installed. SSH into one of the machines and verify using the command 
```
ssh ec2-user@< managed_node_private_ip >
```
```
rpm -qa | grep httpd
```
```
ansible all -m "command" -a "rpm -qa | grep httpd"
```


--------------------------------------------------------------------------------------------------
### Task 4: Until function

Create and edit untillab.yml in the same labs directory 
```
vi untillab.yml
```
```
---
- hosts: all
  become: yes
  connection: ssh
  user: ec2-user
  tasks:
  - name: Install Apache Web Server
    yum:
       name: httpd
       state: latest
  - name: Verify Status of Service
    shell: systemctl status httpd
    register: result
    until: result.stdout.find("active (running)") != -1
    retries: 5
    delay: 10
```
save the file using "ESCAPE + :wq!"

Execute the playbook
Notice the output of the command is shown along with the Ansible until command output
```
ansible-playbook untillab.yml
```
Login to the managed node from another window and start the httpd service
You can use the same key (as used for CN) to login to managed node 
```
ssh ec2-user@< managed_node_private_ip >
```
```
sudo service httpd start
```

you can check the status of httpd by
```
sudo service httpd status
```

---------------------------------------------------------------------------------------------
### Task 5: Run Once with Ansible Playbook

Create and edit rolab.yml in the same labs directory 
```
vi rolab.yml
```
```
---
- hosts: all
  become: yes
  user: ec2-user
  connection: ssh
  gather_facts: no
  tasks:
    - name: Recording uptime 
      raw: /usr/bin/uptime >> /home/ec2-user/uptime
      run_once: true
```
**save the file using** `ESCAPE + :wq!`

Execute the playbook
```
ansible-playbook rolab.yml
```
Verify if the file exists and has the right contents on either of the client machines(manage nodes)
```
ansible all -a "cat /home/ec2-user/uptime"
```

now open the file and edit parameter as run_once: false 
Execute the playbook again
```
ansible-playbook rolab.yml
```
Verify if the file exists and has the right contents on the client machines(manage nodes)
```
ansible all -a "cat /home/ec2-user/uptime"
```
-----------------------------------------------------------------------------------------------
### Task 6: Blocks with Ansible Playbook

Create and edit blklab.yml in the same labs directory 
Notice that the “web_package” variable is an invalid package. Due to the invalid package in a block, tasks under rescue will run
```
vi blklab.yml
```
```
---
- hosts: all
  become: yes
  user: ec2-user
  connection: ssh
  gather_facts: no
  tasks:
    - block:
        - name: Install {{ web_package }} package
          yum:
            name: "{{ web_package }}"
            state: latest
      rescue:
        - name: Install {{ db_package }} package
          yum:
            name: "{{ db_package }}"
            state: latest
      always:
        - name: Start {{ db_service }} service
          service:
            name: "{{ db_service }}"
            state: started
  vars:
    web_package: http
    db_package: mariadb-server
    db_service: mariadb
```
**save the file using** `ESCAPE + :wq!`

Execute the playbook
Block tasks fail and that Rescue tasks are running due to the failure of block tasks. The Always tasks run independently
```
ansible-playbook blklab.yml
```
Now fix the package name in the Playbook (web_package: httpd) and run the Playbook again
```
ansible-playbook blklab.yml
```

Notice that the tasks under rescue are not running as block tasks ran successfully.

## Lab 9: Implementing Jinja2 Templates

Create a new directory for templates 
```
mkdir templates
```
```
cd templates
```
In that directory create a template file, let’s say new-motd.j2
```
vi new-motd.j2
```
Put the below content into the newly created template file
```
System Information:
Operating System: {{ ansible_os_family | default('N/A') }}
Architecture: {{ ansible_architecture | default('N/A') }}
Distribution Version: {{ ansible_distribution | default('N/A') }}
```

Now create the playbook for implementing this jinja2 template that we created
```
vi implement-jinja2.yml
```
Put the below content in the playbook:
```
---
- name: playbook
  hosts: all
  gather_facts: true
  become: yes
  tasks:
    - name: Render template
      template:
        src: new-motd.j2
        dest: /etc/motd
        owner: ec2-user
        group: ec2-user
        mode: 0644
```
Now execute the playbook 
```
ansible-playbook implement-jinja2.yml -b 
```
 
 Now we will check what our template is exactly doing, where it is putting the values.
For that, do the following:
```
ansible all -m shell -a "cat /etc/motd"
```



## Lab 10: Implementing Ansible Roles

---------------------------------------------------------------------------------------
### Task 1: Implementing Ansible Roles
```
cd ~/
```
Lets uninstall httpd. After that, we will use ansible role to install it.
```
ansible-playbook /home/ec2-user/ansible-labs/uninstall-apache-pb.yml (Please provide the correct path name)
```

Install tree. A tree is a recursive directory listing program that produces a depth-indented listing of files. 
```
sudo yum install tree -y
```

You can view your home directory structure in tree format with below command tree 
```
tree /home/ec2-user/ansible-labs
```
Lets create the code for Role labs
```
cd ~/
```
```
mkdir role-labs && cd role-labs
```

Now inside the roles directory, create two different directories for different roles, namely webrole and dbrole. Then switch to the directory dbrole and then create tasks directory inside dbrole
```
mkdir webrole dbrole && cd dbrole
```
```
mkdir tasks
```

This main.yml is the playbook which will get executed to make an effect of this role and put the below content in the main.yml file
```
vi tasks/main.yml
```
```
---
- name: Install MariaDB server package
  yum: 
    name: mariadb-server 
    state: present
- name: Start MariaDB Service
  service: 
    name: mariadb 
    state: started 
    enabled: true
```
**save the file using** `ESCAPE + :wq!`


Now change your directory to webrole 
```
cd .. && cd webrole/
```
```
mkdir files tasks && cd files/
```
```
vi index.html
```
```
<html>
  <body>
  <h1>We are performing the Roles Lab</h1>
  <img src= "https://d3ffutjd2e35ce.cloudfront.net/assets/logo1.png">
  </body>
</html>
```
**save the file using** `ESCAPE + :wq!`

Then go to the task directory as below and create main.yml 
```
cd .. && cd tasks/
```
```
vi main.yml
```
Add the given content, by pressing "INSERT" 
```
---
- name: install httpd
  yum: 
    name: httpd 
    update_cache: yes 
    state: latest

- name: uploading default index.html for host
  copy:
     src: /home/ec2-user/role-labs/webrole/files/index.html
     dest: /var/www/html

- name: Setting up attributes for file
  file:
    path:  /var/www/html/index.html
    owner: apache
    group: apache
    mode:  0644

- name: start httpd
  service:
    name=httpd 
    state=started
```
**save the file using** `ESCAPE + :wq!`

After the creation of this file, we are done with the complete hierarchy of roles, so we will have a look at how it is exactly using tree command
```
cd ../..
```
```
tree
```

Now change the directory to ansible directory and create the playbook as implement-roles.yml
```
cd ..
```
```
vi implement-roles.yml
```

Add the given content, by pressing "INSERT".
```
---
 - hosts: all
   become: yes 

   roles:
     - webrole
     - dbrole
```   
**save the file using** `ESCAPE + :wq!`

Execute the playbook
```
ansible-playbook implement_roles.yml
```
Check the home page on browser. (Public DNS)
It will show the webpage with msg "We are performing the Roles Lab"

----------------------------------------------------------------------------------------------
### Task 2: Installing Java through Ansible Galaxy Roles galaxy

Install java form ansible galaxy role from galaxy.ansible.com   

Now Install the role 'geerlingguy.java' from ansible galaxy repository. 
```
ansible-galaxy install geerlingguy.java
```
Now change into labs directory by running below command and create YAML file
```
cd /home/ec2-user/labs/
```
```
vi implement-java.yml
```

Add the given content, by pressing "INSERT" 
```
---
 - hosts: all
   become: yes
   roles:
     - geerlingguy.java
```
save the file using "ESCAPE + :wq!"

Before running the playbook, check if java is installed in managed nodes.
```
ansible all -m command -a "java -version"
```
you will get error
execute playbook from the control node
```
ansible-playbook implement-java.yml
```
Now, check if java is installed in managed nodes.
```
ansible all -a "java -version"
```
***********************************************************************************************************************************

 

