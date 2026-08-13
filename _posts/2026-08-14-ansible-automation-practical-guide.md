---
layout: post
title: "Ansible 自动化运维实战指南"
date: 2026-08-14 09:00:00 +0800
categories: [开发]
tags: [Ansible, 自动化运维, IaC, DevOps, 配置管理, Playbook, 最佳实践]
---

## 为什么需要 Ansible

手动 SSH 到几十台服务器改配置，是运维事故的温床。

- 漏改了一台机器，线上故障排查半天
- 配置文件格式错了一个字符，服务起不来
- 版本没对齐，环境间行为不一致

Ansible 解决了这些问题。它用声明式的 YAML 描述「目标状态」，幂等执行，无需在目标机器上安装 Agent（依赖 Python 和 SSH）。相比 SaltStack 需要 minion、Puppet/Chef 需要证书体系，Ansible 的「零 Agent」架构让上手成本极低。

本文从安装开始，到编写可复用的 Playbook 和 Roles，覆盖日常运维中最常见的场景。

## 安装与基础配置

### 安装

控制节点（你用来运行命令的机器）需要安装 Ansible。管理节点无需安装任何东西，只要 Python 2.7+ 或 Python 3.5+ 和 SSH 可用。

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y ansible

# RHEL/CentOS/Rocky
sudo dnf install -y epel-release && sudo dnf install -y ansible

# macOS
brew install ansible

# 验证
ansible --version
```

输出示例：

```
ansible [core 2.18.1]
  config file = /etc/ansible/ansible.cfg
  python version = 3.12.4
```

### SSH 免密配置

Ansible 通过 SSH 管理远程主机。推荐使用密钥认证：

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible -N ""
ssh-copy-id -i ~/.ssh/ansible.pub user@target-host
```

在 `ansible.cfg` 中指定私钥：

```ini
[defaults]
private_key_file = ~/.ssh/ansible
host_key_checking = False
```

生产环境建议保留 `host_key_checking = True`，并提前将主机指纹写入 `known_hosts`。

## Inventory：管理主机清单

Inventory 定义 Ansible 管理哪些机器。默认路径 `/etc/ansible/hosts`，但更推荐每个项目单独维护。

### 静态 Inventory（INI 格式）

```ini
[webservers]
web1.example.com ansible_user=ubuntu
web2.example.com ansible_user=ubuntu

[dbservers]
db1.example.com ansible_user=deploy
db2.example.com

[all:vars]
ansible_port=22
ansible_python_interpreter=/usr/bin/python3
```

### 静态 Inventory（YAML 格式）

```yaml
all:
  children:
    webservers:
      hosts:
        web1.example.com:
          ansible_user: ubuntu
        web2.example.com:
          ansible_user: ubuntu
    dbservers:
      hosts:
        db1.example.com:
          ansible_user: deploy
        db2.example.com:
    monitoring:
      hosts:
        monitor.example.com:
```

### 动态 Inventory

云环境的主机 IP 经常变化，不适合写静态文件。Ansible 支持动态 Inventory 脚本：

```bash
# AWS EC2
ansible-inventory -i aws_ec2.yaml --list

# 对应的 aws_ec2.yaml
plugin: amazon.aws.aws_ec2
regions:
  - ap-northeast-1
filters:
  tag:Environment: production
hostnames:
  - tag:Name
```

## Ad-Hoc 命令：快速执行

不想写 Playbook 时，用一行命令完成任务：

```bash
# Ping 所有主机
ansible all -i inventory.ini -m ping

# 查看系统信息
ansible webservers -m gather_facts --limit web1

# 安装包
ansible webservers -m apt -a "name=nginx state=present" -b

# 复制文件
ansible webservers -m copy -a "src=/etc/hosts dest=/etc/hosts owner=root group=root mode=0644" -b

# 查看远程磁盘使用
ansible all -m shell -a "df -h | grep ^/dev"
```

参数说明：
- `-m`：指定模块（module）
- `-a`：模块参数
- `-b`：become（提权，默认 sudo）
- `-i`：指定 inventory 文件

## Playbook：自动化工作流

Playbook 是 Ansible 的核心。它把多个任务编排成可重复执行的剧本。

### 基础 Playbook

```yaml
---
- name: 配置 Nginx Web 服务器
  hosts: webservers
  become: yes
  vars:
    nginx_port: 8080
    server_name: example.com

  tasks:
    - name: 安装 Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: 写入 Nginx 配置
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: restart nginx

    - name: 启用站点
      file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link
      notify: reload nginx

    - name: 确保 Nginx 运行
      service:
        name: nginx
        state: started
        enabled: yes

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted

    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

### 执行 Playbook

```bash
# 语法检查
ansible-playbook --syntax-check nginx.yml

# 试运行（只检查，不执行）
ansible-playbook -C nginx.yml

# 实际执行
ansible-playbook nginx.yml -i inventory.ini

# 限制目标主机
ansible-playbook nginx.yml --limit web1

# 带标签执行
ansible-playbook nginx.yml --tags "install,config"
```

### 幂等性

Ansible 的核心设计原则是幂等——同一个 Playbook 执行多次，结果不变。比如：

```yaml
- name: 确保用户存在
  user:
    name: deploy
    state: present
    groups: sudo
    shell: /bin/bash
```

第一次执行时创建用户，第二次执行时检测到用户已存在且状态匹配，不做任何操作。这就是 `state: present` 的价值。

## 常用模块速查

### 文件操作

```yaml
- name: 创建目录
  file:
    path: /data/app/logs
    state: directory
    owner: app
    group: app
    mode: '0755'

- name: 删除旧日志
  file:
    path: /data/app/logs/old.log
    state: absent

- name: 创建软链接
  file:
    src: /data/app/releases/current
    dest: /data/app/active
    state: link
```

### 复制与模板

```yaml
- name: 复制静态文件
  copy:
    src: files/app.conf
    dest: /etc/app/app.conf
    owner: root
    group: root
    mode: '0644'

- name: 使用 Jinja2 模板
  template:
    src: templates/app.conf.j2
    dest: /etc/app/app.conf
  vars:
    app_port: 3000
    log_level: info
```

模板文件 `app.conf.j2` 示例：

```jinja
server {
    listen {{ app_port }};
    server_name {{ server_name }};
    access_log /var/log/nginx/{{ server_name }}.log;

    location / {
        proxy_pass http://127.0.0.1:{{ app_port }};
    }
}
```

### 包管理

```yaml
- name: 安装系统包
  apt:
    name:
      - nginx
      - certbot
      - python3-certbot-nginx
      - htop
      - fail2ban
    state: present
    update_cache: yes

- name: 安装 Python 包
  pip:
    name:
      - gunicorn
      - psycopg2-binary
    state: present
    virtualenv: /opt/app/venv
```

### 服务管理

```yaml
- name: 启动并启用服务
  systemd:
    name: nginx
    state: started
    enabled: yes
    daemon_reload: yes
```

## 实战：系统安全加固 Playbook

以下是一个完整的 Playbook，用于新服务器初始安全加固：

```yaml
---
- name: 服务器安全基线配置
  hosts: all
  become: yes
  vars:
    ssh_port: 2222
    admin_users:
      - alice
      - bob
    allowed_ssh_users:
      - alice
      - bob
      - deploy

  tasks:
    - name: 更新所有包
      apt:
        upgrade: dist
        update_cache: yes
        cache_valid_time: 3600

    - name: 创建管理员用户
      user:
        name: "{{ item }}"
        groups: sudo
        shell: /bin/bash
        create_home: yes
        state: present
      loop: "{{ admin_users }}"

    - name: 配置 SSH 密钥
      authorized_key:
        user: "{{ item }}"
        key: "{{ lookup('file', 'files/keys/' + item + '.pub') }}"
      loop: "{{ admin_users }}"

    - name: 加固 SSH 配置
      template:
        src: templates/sshd_config.j2
        dest: /etc/ssh/sshd_config
        owner: root
        group: root
        mode: '0600'
      notify: restart sshd

    - name: 安装 fail2ban
      apt:
        name: fail2ban
        state: present

    - name: 配置 fail2ban
      copy:
        src: files/jail.local
        dest: /etc/fail2ban/jail.local
      notify: restart fail2ban

    - name: 配置 UFW 防火墙
      ufw:
        rule: "{{ item.rule }}"
        port: "{{ item.port }}"
        proto: "{{ item.proto | default('tcp') }}"
      loop:
        - { rule: allow, port: '{{ ssh_port }}', proto: tcp }
        - { rule: allow, port: '80', proto: tcp }
        - { rule: allow, port: '443', proto: tcp }
        - { rule: deny, port: '22', proto: tcp }

    - name: 启用 UFW
      ufw:
        state: enabled
        policy: deny

    - name: 安装监控工具
      apt:
        name:
          - htop
          - iotop
          - netstat
          - lsof
          - sysstat
        state: present

  handlers:
    - name: restart sshd
      service:
        name: sshd
        state: restarted

    - name: restart fail2ban
      service:
        name: fail2ban
        state: restarted
```

## 变量与事实

### 变量优先级

Ansible 变量有 20+ 个来源，优先级从高到低最常用的几个：

```
extra vars (-e) > play vars > host vars > group vars > role defaults
```

```bash
# 设置额外变量
ansible-playbook deploy.yml -e "env=production version=v2.1.0"
```

### Facts：主机信息收集

每次 Playbook 执行时，Ansible 会自动收集目标主机的 Facts（CPU 核心数、内存、IP 地址、操作系统等）：

```yaml
- name: 根据 CPU 核心数配置 worker 进程
  template:
    src: gunicorn.conf.py.j2
    dest: /opt/app/gunicorn.conf.py
  vars:
    workers: "{{ ansible_processor_cores * 2 + 1 }}"
```

```jinja
# gunicorn.conf.py.j2
workers = {{ workers }}
bind = "0.0.0.0:8000"
user = "app"
```

### 注册变量

将任务的结果赋值给变量，供后续任务使用：

```yaml
- name: 检查磁盘空间
  shell: df -h / | tail -1 | awk '{print $5}' | sed 's/%//'
  register: disk_usage

- name: 磁盘空间不足时告警
  debug:
    msg: "磁盘使用率 {{ disk_usage.stdout }}%，超过 90% 阈值！"
  when: disk_usage.stdout | int > 90
```

## 角色：复用代码

Roles 将 Playbook 组织成可复用的目录结构。使用 `ansible-galaxy` 创建角色骨架：

```bash
ansible-galaxy init nginx
```

目录结构：

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    ├── templates/
    │   └── nginx.conf.j2
    ├── files/
    ├── vars/
    │   └── main.yml
    ├── defaults/
    │   └── main.yml
    ├── meta/
    │   └── main.yml
    └── README.md
```

### 使用角色

```yaml
# site.yml
- name: 配置 Web 服务器
  hosts: webservers
  become: yes
  roles:
    - common
    - nginx
    - fail2ban
    - monitoring
```

可以对角色传递变量：

```yaml
- name: 配置 Web 服务器
  hosts: webservers
  roles:
    - role: nginx
      vars:
        nginx_port: 8080
        server_name: "{{ inventory_hostname }}"
```

### 从 Ansible Galaxy 使用社区角色

```bash
# 安装社区角色
ansible-galaxy install geerlingguy.nginx

# 在 Playbook 中使用
- name: 使用社区角色
  hosts: webservers
  roles:
    - role: geerlingguy.nginx
      vars:
        nginx_worker_processes: "{{ ansible_processor_cores }}"
        nginx_remove_default_vhost: true
```

## 最佳实践总结

| 原则 | 说明 |
|------|------|
| 保持 Playbook 幂等 | 始终使用 `state: present/absent`，避免条件判断过于复杂 |
| 优先使用模块，少用 shell/command | 模块有幂等性保障，shell 命令没有 |
| 用 Tag 组织任务 | `--tags install`, `--tags configure` 方便调试 |
| 使用 Vault 加密敏感数据 | `ansible-vault encrypt secrets.yml` |
| 分层管理变量 | 项目级 > 环境级 > 主机级 > 角色默认值 |
| 使用 `--check` 试运行 | 上线前必须执行一次检查 |
| 模板用 `.j2` 后缀 | 直观区分模板文件和普通文件 |
| 编写 Role 时提供默认值 | `defaults/main.yml` 中的变量优先级最低，使用者可以覆盖 |

### Ansible Vault 加密

```bash
# 加密文件
ansible-vault encrypt secrets.yml

# 编辑加密文件
ansible-vault edit secrets.yml

# 使用加密文件执行
ansible-playbook deploy.yml --ask-vault-pass

# 或指定密码文件
ansible-playbook deploy.yml --vault-password-file .vault_pass
```

加密后的文件内容：

```yaml
$ANSIBLE_VAULT;1.1;AES256
6234636538353663663333303932663832646338393632663832363837386430
6231386335383933383831626338613230643464666335386633386530640a
...
```

## 小结

Ansible 的价值不在于「能用」，而在于「用对」。本文覆盖了从安装、Inventory、Playbook 到 Roles 的完整路径，以及安全加固、变量管理、Vault 加密等实战场景。

几个关键点：

- **零 Agent 架构**是 Ansible 最大的优势，但也意味着 SSH 连接质量直接影响执行效率——建议开启 `pipelining = True` 减少 SSH 往返
- **幂等是生命线**——写 Playbook 时始终问自己：执行两次和一次，结果一样吗？
- **Roles 不是银弹**——小项目用扁平 Playbook 更清晰，几十个 Role 的依赖关系可能比代码本身更难维护
- **测试先行**——推荐使用 `molecule` 框架在 Docker 容器中测试 Role，而不是直接操作生产环境

下一步可以探索 Ansible Tower/AWX 提供 Web UI 和调度能力，或结合 Packer 和 Terraform 实现完整的「基础设施即代码」工作流。