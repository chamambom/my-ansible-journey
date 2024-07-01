### Why Ansible and Terraform?

**Short Answer**: Use Terraform to build your cloud infrastructure (e.g., VPC, AKS, firewall rules), and then utilize Ansible for configuration management and further automation.

**Ansible**: Ideal for configuration management, automating application deployment, and managing complex software setups.

**Terraform**: Best suited for infrastructure provisioning, allowing you to define, provision, and manage cloud resources in a consistent manner.

Currently, there is no straightforward Terraform/Ansible bootstrapper available, so creating your own integration is necessary. Here are a few considerations:

- **Terraform with Packer-Baked AMIs**: Utilize Packer (typically with shell scripts) to create custom AMIs pre-configured for your nodes.
- **Terraform with Provisioners**: Leverage Terraform's provisioners to automate tasks that need to occur after infrastructure provisioning.
- **Custom Solutions**: Develop a tailored pipeline that combines Terraform and Ansible to streamline both infrastructure and configuration management processes.

Each approach has unique benefits, and the best choice will depend on your specific requirements and environment.

---

### My "boostrapping" approach !!


My integration strategy involves categorizing Ansible runs into two distinct phases: instance bootstrapping and configuration drift management. 

In my current role, I frequently use Terraform and have collaborated with many clients who are adopting Ansible as their primary configuration tool. This repository will document my journey toward developing a Terraform/Ansible bootstrapping approach.

![alt text](/refimages/ansible-tf-intergration.png)

---

### How my journey began.

My Ansible journey started in 2013 when I joined an Internet Service Provider (ISP) as a Network Operations Engineer. Although I didn't initially document my journey, I recently found myself immersed in an Ansible & Terraform project. Recognizing the value of documentation, I took the initiative to start documenting my experiences.
 The ISP that I was engaged with at the time was deeply committed to open source technologies across its operations. Within our Network Operations Center (NOC), we had distinct teams for Network Engineering, 

- Systems Engineering and 
- Network Engineering 

Although all engineers were expected to possess knowledge in both networks and systems, we were encouraged to specialize in one area to enhance the efficiency of the NOC team. I was a member of the Systems Engineering team, which was tasked with overseeing all hosted services, including SNMP, HTTP, SMTP, DNS, and the maintenance of KVM virtual servers hosting these services.

With the ISP experiencing a substantial growth in its customer base, we initiated discussions about implementing automation for all hosted services - SMTP (Postfix), HTTP (Apache), and DNS services.

#### NB
- Ansible is better suited to configuration management post infra deployment. 
- This repo will contain a CI/CD pipeline implementation of ansible using github actions.

---

### How Ansible Works in general.

![alt text](/refimages/AnsibleDesign.png)


##### Installing Ansible on Ubuntu 22.04 [Ansible Controller]
- sudo apt install software-properties-common
- sudo add-apt-repository --yes --update ppa:ansible/ansible
- sudo apt install ansible

![alt text](/refimages/ansibleversion.png)

##### Configure Host Servers
The servers which you want to manage using Ansible must have SSH installed and port 22 opened in the firewall to access them from other systems such as the one installed with Ansible.
For example, you have servers running on Ubuntu, Debian, and CentOS that you want to manage and configure using Ansible. Thus, we need to install the SSH server and open port 22 on them.

Now, on each server run the below command, so that we can run commands with sudo on them using Ansible but without entering a password.
echo "$(whoami) ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/$(whoami)

##### Create Inventory File for Remote Hosts
In Ansible, we create a file where we will define all the remote hosts or target systems that we want to manage. We can also create a group of hosts, for example, one group is a Web server that only contains a remote system running some web servers such as Apache, and the other can be a group of Mysql running a database server and so on. The Inventory file is also important because using it the command, modules, and tasks in a playbook will operate.
So, as here in this tutorial we have three remote servers, let’s add them to the Ansible host file.
sudo nano /etc/ansible/hosts

![alt text](/refimages/hosts.png)

##### Setup SSH Passwordless Login in Linux 

- run  ssh-keygen -t rsa on the ansible controller
- ssh-copy-id username@192.168.0.11
- Disable Password Authentication (Optional) 
    - $ sudo vi /etc/ssh/sshd_config
    - Find the line containing PasswordAuthentication and set it to no.
        PasswordAuthentication no

- Test SSH Passwordless Login from ansible controller

 
##### Ping All added Remote servers
As we have created the inventory file successfully, let’s check whether our Ansible could ping all the added servers or not, for that-
To ping a group of hosts-
ansible -m ping group-name
ansible -m ping all

#### Approach [adopted from Redhat Ansible Best Practices]

Version control your Ansible content
- Iterate
    - Start with a basic playbook and static inventory
    - Refactor and modularize later


- Create a style guide for consistency:
    - Tagging
    - Whitespace
    - Naming of Tasks, Plays, Variables, and Roles
    - Directory Layouts
    - Enforce the style  - example  [https://goo.gl/JfWBcW]

    ![alt text](/refimages/image.png)


Start with one Git repository - but when it grows,
use multiple!


#### Start with one Git repository - but when it grows, use multiple!

At the beginning: put everything in one Git repository

In the long term:

- One Git repository per role
- Dedicated repositories for completely separated teams / tasks

#### Give inventory nodes human-meaningful names rather than IPs or DNS hostnames.

![alt text](/refimages/names.png)


#### Group hosts for easier inventory selection and less conditional tasks -- the more the better.

![alt text](/refimages/groups.png)

#### Use dynamic sources where possible. Either as a single source of truth - or let Ansible unify multiple sources.

- Stay in sync automatically
- Reduce human error
- No lag when changes occur
- Let others manage the inventory

#### Proper variable names can make plays more readable and avoid variable name conflicts

![alt text](/refimages/variables.png)

#### Avoid collisions and confusion by adding the role name to a variable as a prefix.

apache_max_keepalive: 25
apache_port: 80
tomcat_port: 8080

##### Know where your variables are

- Find the appropriate place for your variables based on what, where and when they are set or modified
- Separate logic (tasks) from variables and reduce repetitive patterns
- Do not use every possibility to store variables - settle to a defined scheme and as few places as possible

#### Make your playbook readable

![alt text](/refimages/readableNo.png)

![alt text](/refimages/readableNo2.png)

![alt text](/refimages/readable.png)

#### Examples 

![alt text](/refimages/exhibitA.png)

![alt text](/refimages/exhibitB.png)

#### Blocks can help in organizing code, but also enable rollbacks or output data for critical changes.

![alt text](/refimages/blocks.png)

#### Ansible provides multiple switches for command line interaction and troubleshooting.

-vvvv
--step
--check
--diff
--start-at-task

#### Ansible has switches to show you what will be done

Use the power of included options:
--list-tasks
--list-tags
--list-hosts
--syntax-check

#### If there is a need to launch something without an inventory
- just do it!

- For single tasks - note the comma:
  ansible all -i neon.qxyz.de, -m service -a
  "name=redhat state=present"
- For playbooks - again, note the comma:
ansible-playbook -i neon.qxyz.de, site.yml


#### Don’t just start services -- use smoke tests

![alt text](/refimages/smoketests.png)

#### Try to avoid the command module - always seek out a module first

![alt text](/refimages/command.png)


#### If managed files are not marked, they might be overwritten accidentally

- Label template output files as being generated by Ansible
- Use the ansible_managed** variable with the comment filter

{{ ansible_managed | comment }}

#### Roles enable you to encapsulate your operations.

- Like playbooks -- keep roles purpose and function focused
- Store roles each in a dedicated Git repository
- Include roles via roles/requirements.yml file, import via ansible-galaxy tool
- Limit role dependencies

#### Get roles from Galaxy, but be careful and adopt them to your needs

- Galaxy provides thousands of roles
- Quality varies drastically
- Take them with a grain of salt
- Pick trusted or well known authors


### Access Rights 

#### Root access is harder to track than sudo - use sudo wherever possible

- Ansible can be run as root only
- But login and security reasons often request non-root access
- Use become method - so Ansible scripts are executed via sudo
(sudo is easy to track)
- Best: create an Ansible only user
- Don’t try to limit sudo rights to certain commands - Ansible does not work that way!

### Logging

#### Check logging on target machine

![alt text](/refimages/logging.png)

#### How to keep the code executed on the target machine

Look into the logging of your target machine

$ ANSIBLE_KEEP_REMOTE_FILES=1 ansible target-node -m yum  -a "name=httpd state=absent"

Execute with:

$ /bin/sh -c 'sudo -u $SUDO_USER /bin/sh -c "/usr/bin/python /home/liquidat/.ansible/tmp/..."

#### Debugging tasks can clutter the output, apply some housekeeping

![alt text](/refimages/debug.png)