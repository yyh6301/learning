Ansible 是一个模型驱动的配置管理器，支持多节点发布、远程任务执行。默认使用 SSH 进行远程连接。无需在被管理节点上安装附加软件，可使用各种编程语言进行扩展。

## 架构
![[Pasted image 20240531121749.png]]

核心模块（Core Modules）：这些都是ansible自带的模块

扩展模块（Custom Modules）：如果核心模块不足以完成某种功能，可以添加扩展模块

插件（Plugins）：完成模块功能的补充

剧本（Playbooks）：ansible的任务配置文件，将多个任务定义在剧本中，由ansible自动执行

连接插件（Connectior Plugins）：ansible基于连接插件连接到各个主机上，虽然ansible是使用ssh连接到各个主机的，但是它还支持其他的连接方法，所以需要有连接插件

主机群（Host Inventory）：定义ansible管理的主机



## inventory

在Ansible中，Inventory（清单）是一个定义管理主机和组的文件。它告诉Ansible哪些主机或服务器需要被管理，以及如何对这些主机进行分组和配置。Inventory文件可以是一个简单的INI格式文件，也可以是更复杂的YAML格式文件，还可以通过动态脚本来生成。


### INI格式的Inventory

INI格式的Inventory文件是最常见和最简单的格式。它由主机和组的定义组成，组内的主机可以通过IP地址或主机名来指定。以下是一个典型的INI格式Inventory文件的例子：

ini

```shell
# 文件名：/etc/ansible/hosts

[nodes]
centos-node1 ansible_host=192.168.56.101
centos-node2 ansible_host=192.168.56.102
centos-node3 ansible_host=192.168.56.103

[webservers]
webserver1 ansible_host=192.168.56.104
webserver2 ansible_host=192.168.56.105

[databases]
db1 ansible_host=192.168.56.106
db2 ansible_host=192.168.56.107

[all:vars]
ansible_user=root
ansible_ssh_pass=password

```


在这个例子中：

- `[nodes]`、`[webservers]`和`[databases]`是组名。
- 组名下面是主机名或IP地址，以及可选的Ansible变量（如`ansible_host`）。
- `[all:vars]`定义了对所有主机生效的全局变量，如SSH用户名和密码。

### YAML格式的Inventory

YAML格式的Inventory文件提供了更多的结构化和灵活性。以下是一个YAML格式Inventory文件的例子：

```yaml
all:
  vars:
    ansible_user: root
    ansible_ssh_pass: password
  children:
    nodes:
      hosts:
        centos-node1:
          ansible_host: 192.168.56.101
        centos-node2:
          ansible_host: 192.168.56.102
        centos-node3:
          ansible_host: 192.168.56.103
    webservers:
      hosts:
        webserver1:
          ansible_host: 192.168.56.104
        webserver2:
          ansible_host: 192.168.56.105
    databases:
      hosts:
        db1:
          ansible_host: 192.168.56.106
        db2:
          ansible_host: 192.168.56.107
```


### 动态Inventory

动态Inventory允许从外部系统（如云提供商、CMDB、数据库等）动态生成主机列表。动态Inventory脚本通常返回JSON格式的数据，Ansible在运行时调用这些脚本以获取最新的主机列表。

以下是一个简单的动态Inventory脚本的例子（假设文件名为`dynamic_inventory.py`）：

```python
#!/usr/bin/env python

import json

inventory = {
    "nodes": {
        "hosts": ["centos-node1", "centos-node2", "centos-node3"],
        "vars": {
            "ansible_user": "root",
            "ansible_ssh_pass": "password"
        }
    },
    "_meta": {
        "hostvars": {
            "centos-node1": {
                "ansible_host": "192.168.56.101"
            },
            "centos-node2": {
                "ansible_host": "192.168.56.102"
            },
            "centos-node3": {
                "ansible_host": "192.168.56.103"
            }
        }
    }
}

print(json.dumps(inventory))

```


运行此脚本时，Ansible将调用它来获取主机和组的列表以及相关变量。

### 使用Inventory

在使用Ansible命令时，可以指定Inventory文件：
```bash
ansible -i /path/to/inventory all -m ping
```



或者在`ansible.cfg`配置文件中设置默认的Inventory路径：
```ini
# 文件名：ansible.cfg 
[defaults] 
inventory = /path/to/inventory
```

## facts和自定义变量

### Ansible Facts

**Facts**是Ansible在目标主机上自动收集的系统信息。这些信息包括操作系统类型、网络配置、硬件信息等。Facts对于编写灵活、动态的Ansible剧本（playbook）非常有用，因为它们提供了有关目标主机的详细信息，可以根据这些信息做出决策。

#### 收集Facts

Ansible使用`setup`模块来收集facts。你可以手动调用这个模块来查看收集到的信息：

```bash
ansible all -m setup
```

这条命令会显示所有收集到的facts。

#### 使用Facts

在playbook中，你可以直接使用facts。它们存储在变量中，变量名称的前缀是`ansible_facts`。

例如，要访问目标主机的IP地址，可以使用`ansible_default_ipv4.address`：
```yaml
- name: Print the default IP address
  hosts: all
  tasks:
    - name: Show the IP address
      debug:
        msg: "The IP address is {{ ansible_facts.default_ipv4.address }}"
```




### 自定义变量

除了自动收集的facts，Ansible还允许你定义自定义变量，以便在playbook和模板中使用。自定义变量可以在多个地方定义，包括Inventory文件、playbook、主机变量文件和组变量文件。

#### 在Inventory文件中定义变量

你可以在Inventory文件中定义主机或组的变量：

```ini
# 文件名：/etc/ansible/hosts

[nodes]
centos-node1 ansible_host=192.168.56.101 my_custom_var=custom_value
centos-node2 ansible_host=192.168.56.102

[nodes:vars]
ansible_user=root
ansible_ssh_pass=password
common_var=value_for_all_nodes

```


#### 在Playbook中定义变量

在playbook中，你可以在`vars`部分定义变量：

```yaml
- name: Example playbook
  hosts: all
  vars:
    my_var: "Hello, World!"
  tasks:
    - name: Print a variable
      debug:
        msg: "{{ my_var }}"
```
#### 在主机变量和组变量文件中定义变量

你可以创建主机变量文件和组变量文件来定义变量。这些文件通常放在`group_vars`和`host_vars`目录中。

**主机变量文件**：

```yaml
# 文件名：host_vars/centos-node1.yml

my_var: "Value for centos-node1"

```
**组变量文件**：

```yaml
# 文件名：group_vars/nodes.yml

common_var: "Value for all nodes"
another_var: "Another common value"
```

### 覆盖顺序

Ansible变量有一个优先级顺序。如果在多个地方定义了同一个变量，Ansible会按照以下顺序使用变量值（从低优先级到高优先级）：

1. Role defaults
2. Inventory文件中的组变量
3. Inventory文件中的主机变量
4. Group_vars文件中的变量
5. Host_vars文件中的变量
6. Playbook中的`vars`
7. Include文件中的`vars`
8. Role中的`vars`
9. Block中的`vars`
10. Task中的`vars`
11. Extra vars（通过命令行传递）


## playbook

Ansible Playbook 是 Ansible 的核心工具，用于定义和执行自动化任务的集合。Playbook 是一种描述基础设施配置、部署应用程序或执行其他 IT 自动化任务的文件。Playbook 使用 YAML 语法来编写，便于阅读和理解。以下是 Ansible Playbook 的基本概念和使用方法的详细介绍。

包括
- **Playbook Header**: 包含 Play 的基本信息，如名称和目标主机。
- **Hosts**: 指定 Play 应用的目标主机，可以是一个或多个主机或组。
- **Become**: 指定是否以超级用户权限执行任务，通常用于需要 root 权限的任务。
- **Vars**: 定义在 Play 中使用的变量。
- **Tasks**: 定义需要执行的任务列表，每个任务由名称、模块和模块参数组成。

### playbook的基本结构

```yaml
- name: Playbook Example
  hosts: all
  become: yes
  vars:
    variable_name: value
  tasks:
    - name: Task description
      module_name:
        module_option1: value1
        module_option2: value2
      when: condition
      with_items:
        - item1
        - item2
```


### playbook示例
在所有目标主机上安装和启动 Apache HTTP 服务器：
```yaml
- name: Install and start Apache
  hosts: all
  become: yes

  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present
      when: ansible_facts['os_family'] == "RedHat"

    - name: Install Apache (Debian)
      apt:
        name: apache2
        state: present
      when: ansible_facts['os_family'] == "Debian"

    - name: Start Apache service
      service:
        name: "{{ ansible_facts['os_family'] == 'RedHat' | ternary('httpd', 'apache2') }}"
        state: started

```

### 执行playbook

```bash
ansible-playbook playbook.yml
```

## 实践：在docker当中模拟ansible操作


### 环境准备

创建docker网络
```bash
docker network create ansible-network
```


新建四个centos容器
```shell
docker run -d --name centos-ansible --network ansible-network centos:latest sleep infinity
docker run -d --name centos-node1 --network ansible-network centos:latest sleep infinity
docker run -d --name centos-node2 --network ansible-network centos:latest sleep infinity
docker run -d --name centos-node3 --network ansible-network centos:latest sleep infinity

```




在主节点当中安装openssh , ansible
```bash
docker exec -it centos-node1 bash 


cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum clean all
yum update -y
yum install -y openssh-server 



#使用python来安装ansible
yum install -y python38 python38-pip
python3.8 -m pip install ansible

```



进入子节点中安装ssh服务

```bash
docker exec -it centos-node1 bash 


cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum clean all
yum update -y
yum install -y vim openssh-server 
ssh-keygen -A
/usr/sbin/sshd  #启动sshd服务


mkdir ~/.ssh
vim ~/.ssh/authorized_keys

#把主节点的公钥放进去
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCmH57y4WdGJhdBUIksxWaH1mhcyRikqX0G9yFIdqb9OVz9G2ebDwgrKUkTRtp0+G+irEcdayOgNiy6FNTuQ6JTHoG7Chxo9blp75GZ5pp+tveLM5b03gopRkaTZXXy4n5quEoZI1UWyI9MRe7FWmgnABTHYMizWRip37qFdHPvCGbKEBvJ2j8tqhR1wD63vEfDhTB3+gKkMJcav3JcDAdq0CSNH0jxdLvD7YshQY4syTphZYWi6Tgp11Cjye7CJc/WUqlD5LKgYMlAsddOuwsIfXCaBmbDp+gc0OuRPu8tFF90c25R4yHhyqNSbOLHW6Rq7wyGMWVELySMmQF7kodp root@fb572b42f1fc

exit
```


### 配置文件中添加子节点信息

确保/etc/ansible/hosts当中包含

```shell
[nodes] 
centos-node1 
centos-node2 
centos-node3
```


### 连通性测试 
使用 ansible all -m ping来测试节点是否联通
![[Pasted image 20240531134906.png]]



### playbook测试

```yaml
#playbook.yml
- name: Install and start Apache
  hosts: all

  tasks:
    - name: Install Apache
      yum:
        name: httpd
        state: present

    - name: Start Apache service
      command: /usr/sbin/httpd -D FOREGROUND

```

```bash
ansible-playbook playbook.yml
```


### 问题解决:
1. yum update失败，解决方案：
```bash
cd /etc/yum.repos.d/

sudo sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sudo sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```


2. 使用yum无法直接安装ansbible

原因:python版本不对
解决方案：使用yum安装python,随后使用pip来安装ansible
```bash
yum install -y python38 python38-pip
python3.8 -m pip install ansible
```



3. ansible 无法直接启动apache
![[Pasted image 20240531141604.png]]

原因：该centos是在docker容器当中的，无法直接使用systemd等命令，解决方案：在ansible中直接使用即可



