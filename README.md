**环境说明：**

|       主机名       |  操作系统版本   |                   ip                   | ansible version |       备注        |
| :----------------: | :-------------: | :------------------------------------: | :-------------: | :---------------: |
|    ansible-awx     | Centos 7.6.1810 |              172.27.9.131              |      2.9.9      | ansible管理服务器 |
| master01/02/03 ... | Centos 7.6.1810 | 172.27.9.28/29/35/36/37/161/162/163/85 |        /        |    被管服务器     |

## 一、加密hosts

### 1.查看hosts列表

```bash
[root@ansible-awx ~]# more /etc/ansible/hosts
[test01]
test28 ansible_host=172.27.34.28
test29 ansible_host=172.27.34.29
test35 ansible_host=172.27.34.35
test36 ansible_host=172.27.34.36
test37 ansible_host=172.27.34.37
test161 ansible_host=172.27.34.161
test162 ansible_host=172.27.34.162
test163 ansible_host=172.27.34.163

[test02]
test85 ansible_host=172.27.34.85

[test:children]
test01
test02

[test:vars]
ansible_ssh_user=root
ansible_ssh_pass=monitor123!
```

### 2.对hosts列表加密

```bash
[root@ansible-awx ~]# cd /etc/ansible/
[root@ansible-awx ansible]# ansible-vault encrypt --vault-id test@prompt hosts
```

![image-20200710161411662](https://i.loli.net/2020/07/13/F6OMgmHzLj7pKwy.png)

因为hosts包含密码信息，为保证安全，需对文件加密，这里以交互方式输入密码，对hosts加密，其中test为标记信息。加密后直接查看hosts文件显示乱码信息，可以使用'ansible-vault view'输入密码查看。

将密码写进hosts文件的优势是不需要在被管服务器上做任何配置(不需要接收配置互信文件)。

## 二、新增用户

### 1.查看并执行新增用户yaml文件

```bash
[root@ansible-awx ansible]# more product/user_add.yaml 
#新增用户，用户名和密码通过手动输入方式确定，组同用户名。
---
- hosts: "{{ hostlist }}" 
  remote_user: root 
  gather_facts: no
  vars_prompt:
    - name: "user_name"
      prompt: "Enter user name"
      private: no
    - name: "user_password"
      prompt: "Enter user password"
      encrypt: "sha512_crypt"
      confirm: yes
  tasks:
   - name: create user
     user:
      name: "{{user_name}}"
      password: "{{user_password}}"
[root@ansible-awx ansible]# cd product/
[root@ansible-awx product]# ansible-playbook --vault-id prompt user_add.yaml -e hostlist=test
Vault password (default): 
Enter user name: monitor
Enter user password: 
confirm Enter user password: 
***** VALUES ENTERED DO NOT MATCH ****
Enter user password: 
confirm Enter user password: 

PLAY [test] ******************************************************************************************************************************************************************************

TASK [create user] ***********************************************************************************************************************************************************************
changed: [test37]
changed: [test35]
changed: [test36]
changed: [test29]
changed: [test28]
changed: [test161]
changed: [test162]
changed: [test163]
changed: [test85]

PLAY RECAP *******************************************************************************************************************************************************************************
test161                    : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
test162                    : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
test163                    : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
test28                     : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
test29                     : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
test35                     : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
test36                     : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
test37                     : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
test85                     : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

![image-20200710164055681](https://i.loli.net/2020/07/13/sbEzTQPqO8an9jC.png)

以交互方式输入用户名和密码信息，其中密码不直接显示在终端并且需要二次确认；执行yaml文件前需以交互方式输入host文件密码；yaml文件中的hosts以参数方式传入，执行的时候通过'-e hostlist=test'指定。

新增用户组同用户名，默认家目录为/home/username

### 2.验证执行结果

```bash
[root@ansible-awx product]# ansible -m shell -a 'id monitor' --vault-id prompt all
```

![image-20200710164124388](https://i.loli.net/2020/07/13/n6kdCI5Kq28BJpi.png)

各主机新增了monitor，且组同用户名。

## 三、用户提权

由于安全的援用，禁用root用户直接登录，同时对monitor用户提权，以获取root权限。

### 1.查看提权文件并执行

```bash
[root@ansible-awx product]# more sudo_PermitRootLogin.yaml 
#monitor用户提权并禁用root用户直接登录
---
- hosts: "{{ hostlist }}" 
  remote_user: root 
  gather_facts: no
  tasks:
  - name: sudo monitor 
    lineinfile:
      path: /etc/sudoers
      regexp: 'monitor' 
      insertafter: 'root.*ALL=\(ALL\).*ALL'
      line: 'monitor    ALL=(ALL)       ALL'
      backup: yes
    register: sudo_monitor 
    tags: sudo
  - name: PermitRootLogin 
    replace:
      path: /etc/ssh/sshd_config 
      regexp: '^#PermitRootLogin.*yes.*' 
      replace: 'PermitRootLogin no'
      backup: yes 
    register: root_login 
    notify:
      restart sshd
    tags: login 
  - name: debug file
    debug:
      var: sudo_monitor 

  handlers:
  - name: restart sshd
    service:
      name=sshd
      state=restarted 
[root@ansible-awx product]# ansible-playbook --vault-id prompt sudo_PermitRootLogin.yaml -e hostlist=test
Vault password (default): 
```

![image-20200712230355592](https://i.loli.net/2020/07/13/VTaP3GQLjJydDR1.png)

![image-20200712230437004](https://i.loli.net/2020/07/13/XQNrkSoVi6AWREh.png)

![image-20200712230459325](https://i.loli.net/2020/07/13/jo4hQx8a5by1Gup.png)

sudo_PermitRootLogin.yaml作用：1.将monitor用户加入到/etc/sudoers文件指定行'root    ALL=(ALL)       ALL'下面以实现提权；2.禁用root直接登录

### 2.验证执行结果

由于root已经禁止直接登录，再用之前的hosts文件会报错，故新建hosts文件monitor

#### 2.1新建monitor主机列表

```bash
[root@ansible-awx ansible]# more monitor 
[test01]
test28 ansible_host=172.27.34.28
test29 ansible_host=172.27.34.29
test35 ansible_host=172.27.34.35
test36 ansible_host=172.27.34.36
test37 ansible_host=172.27.34.37
test161 ansible_host=172.27.34.161
test162 ansible_host=172.27.34.162
test163 ansible_host=172.27.34.163

[test02]
test85 ansible_host=172.27.34.85

[monitor:children]
test01
test02

[monitor:vars]
ansible_ssh_user=monitor
ansible_ssh_pass=monitor
ansible_become=true
ansible_become_method=sudo
ansible_become_user=root
ansible_sudo_pass=monitor
```

主机列表monitor与hosts主要差别是新增提权项，使monitor拥有root权限执行任务。

#### 2.2加密monitor

```bash
[root@ansible-awx ansible]# ansible-vault encrypt --vault-id monitor@prompt monitor
```

![image-20200710170032236](https://i.loli.net/2020/07/13/oAJOuZ3HIyLY9XT.png)

#### 2.3修改ansible.cfg默认配置

```bash
inventory      = /etc/ansible/monitor
```

修改配置文件ansible.cfg，将默认主机列表修改为monitor

#### 2.4验证提权执行结果

```bash
[root@ansible-awx ansible]# ansible -m shell -a "more /etc/ssh/sshd_config |grep 'PermitRootLogin no';more /etc/sudoers|grep monitor" --vault-id prompt monitor
Vault password (default): 
test28 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
test37 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
test29 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
test36 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
test35 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
test163 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
test162 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
test85 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
test161 | CHANGED | rc=0 >>
PermitRootLogin no
monitor    ALL=(ALL)       ALL
```

![image-20200712232256536](https://i.loli.net/2020/07/13/FkN5PJVzGWZT1xK.png)

/etc/sudoers新增'monitor    ALL=(ALL)       ALL'；/etc/ssh/sshd_config将'#PermitRootLogin yes'修改为'PermitRootLogin no'。

## 四、修改密码

### 1.查看并执行密码修改文件

```bash
[root@ansible-awx product]# more user_pass_change.yaml 
#修改用户密码，用户名和密码通过手动输入方式确定
---
- hosts: "{{ hostlist }}"
  remote_user: monitor
  gather_facts: no
  vars_prompt:
    - name: "user_name"
      prompt: "Enter user name"
      private: no
    - name: "user_password"
      prompt: "Enter user password"
      encrypt: "sha512_crypt"
      confirm: yes
  tasks:
   - name: pass user
     user:
      name: "{{user_name}}"
      password: "{{user_password}}"
      update_password: always
[root@ansible-awx product]# ansible-playbook user_pass_change.yaml -e hostlist=monitor --vault-id prompt
Vault password (default): 
Enter user name: root   
Enter user password: 
confirm Enter user password: 
```

![image-20200712233404038](https://i.loli.net/2020/07/13/bUfWytK5paNX2Ew.png)

通过交互方式修改用户密码，密码不在控制台显示且需二次确认。

### 2.验证密码

由于修改的是root密码，之前已设置root不能直接登录，所以这里无法通过ansible管理服务器验证，需要手动登录服务器进行验证。

## 五、资源限制配置文件修改

某些应用对用户的资源使用限制有要求，比如最大打开文件数、进程最大数等。

### 1.查看并执行修改用户资源限制文件

```bash
[root@ansible-awx product]# more user_limits_modify.yaml 
#monitor用户打开最大文件数修改
---
- hosts: "{{ hostlist }}" 
  remote_user: monitor 
  gather_facts: no
  tasks:  
  - name: monitor limits 
    lineinfile:
      path: /etc/security/limits.conf 
      regexp: "{{item.0}}"
      line: "{{item.1}}"
    with_together:
    -
      - monitor.*soft.*nproc
      - monitor.*hard.*nproc
      - monitor.*soft.*nofile
      - monitor.*hard.*nofile
    - 
      - monitor soft nproc 16384
      - monitor hard nproc 16384
      - monitor soft nofile 65536
      - monitor hard nofile 65536
    register: modify
    tags: modify
  - name: debug file
    debug:
      var: modify
[root@ansible-awx product]# ansible-playbook user_limits_modify.yaml -e hostlist=monitor --vault-id prompt
Vault password (default): 
```

![image-20200712235101536](https://i.loli.net/2020/07/13/Q9sUVlz6F24oOIa.png)

![image-20200712235136193](https://i.loli.net/2020/07/13/QH6Pz9YymbZOlnD.png)

![image-20200712235217931](https://i.loli.net/2020/07/13/VxmGUBoMznaN51w.png)

![image-20200712235238630](https://i.loli.net/2020/07/13/VxmGUBoMznaN51w.png)

执行结果内容很多，截图省略了中间部分内容。

该文件主要作用为修改monitor用户资源使用限制。

### 2.执行结果验证

```bash
[root@ansible-awx product]# ansible -m shell -a " more /etc/security/limits.conf |grep monitor" --vault-id prompt monitor
```

![image-20200712235400269](https://i.loli.net/2020/07/13/EvtbLHpPwJ8ZyNs.png)

在limits.conf文件末尾新增内容：

```bash
monitor soft nproc 16384
monitor hard nproc 16384
monitor soft nofile 65536
monitor hard nofile 65536
```

## 六、开启命令审计

记录执行命令的时间、登陆用户、执行的返回码、执行目录、终端连接的ip等在生产故障时可以帮助我们排障。

### 1.查看并执行命令审计文件

```
[root@ansible-awx product]# more shell_audit.yaml 
#开启命令审计
---
- hosts: "{{ hostlist }}" 
  remote_user: monitor 
  gather_facts: no
  tasks:
  - name: make director audit 
    file:
      path: /var/log/shell_audit
      state: directory 
    register: director_create
  - name: make file  log 
    file:
      path: /var/log/shell_audit/audit.log
      state: touch 
      owner: nobody
      group: nobody
      mode: 002
      attributes: +a 
    register: file_create
    tags: file
  - name: modify profile
    lineinfile:
      path: /etc/profile
      regexp: "{{item.0}}"
      line: "{{item.1}}"
    with_together:
    -
      - HISTSIZE=
      - HISTTIMEFORMAT=
      - HISTORY_FILE=
      - PROMPT_COMMAND=
    - 
      - HISTSIZE=2048
      - export HISTTIMEFORMAT="[%Y/%m/%d %T]   "
      - export HISTORY_FILE=/var/log/shell_audit/audit.log
      - export PROMPT_COMMAND='{ code=$?;thisHistID=`history 1|awk "{print \\$1}"`;lastCommand=`history 1| awk "{\\$1=\"\" ;print}"`;user=`id -un`;whoStr=(`who -u am i`);realUser=${whoSt
r[0]};logDay=${whoStr[2]};logTime=${whoStr[3]};pid=${whoStr[5]};ip=${whoStr[6]};if [ ${thisHistID}x != ${lastHistID}x ];then echo -E $user\($realUser\)@$ip [PID:$pid] [LOGIN:$logDay $log
Time] --- [$PWD]$lastCommand [$code] [`date "+%Y/%m/%d %H:%M:%S"`];lastHistID=$thisHistID;fi; } >> $HISTORY_FILE' 
    register: modify
    tags: modify
  - name: debug file
    debug:
      var: modify 
[root@ansible-awx product]# ansible-playbook shell_audit.yaml -e hostlist=monitor --vault-id prompt
Vault password (default): 
```

![image-20200713094422916](https://i.loli.net/2020/07/13/tK4PncBdTJzk3fR.png)

![image-20200713094539669](https://i.loli.net/2020/07/13/hDWN8nQx3kKTZ9U.png)

![image-20200713094612223](https://i.loli.net/2020/07/13/svizNWCgrqOAhG3.png)

shell_audit.yaml主要作用为：新建审计文件并设置相关权限只能被追加不能被删除或修改；修改/etc/profile文件，设定日志格式。

### 2.验证执行结果

```bash
[root@ansible-awx product]# ansible -m shell -a "tail -n10 /var/log/shell_audit/audit.log" --vault-id prompt test02
Vault password (default): 
test85 | CHANGED | rc=0 >>
root(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/root] [2020/07/13 09:42:01] df -h [0] [2020/07/13 09:42:01]
root(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/root] [2020/07/13 09:42:02] more /var/log/shell_audit/audit.log [0] [2020/07/13 09:42:02]
root(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/root] [2020/07/13 09:47:37] cat /etc/passwd [0] [2020/07/13 09:47:37]
monitor(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/home/monitor] [2020/07/13 09:47:40] su - root [0] [2020/07/13 09:47:40]
monitor(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/home/monitor] [2020/07/13 09:47:42] df -h [0] [2020/07/13 09:47:42]
monitor(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/home/monitor] [2020/07/13 09:47:42] pwd [0] [2020/07/13 09:47:42]
monitor(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/home/monitor] [2020/07/13 09:47:44] ls -alrt [0] [2020/07/13 09:47:44]
root(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/root] [2020/07/13 09:47:40] su - monitor [0] [2020/07/13 09:49:22]
root(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/root] [2020/07/13 09:49:26] id [0] [2020/07/13 09:49:26]
root(monitor)@(172.27.9.246) [PID:6410] [LOGIN:2020-07-13 09:37] --- [/root] [2020/07/13 09:49:37] whoami [0] [2020/07/13 09:49:37]
```

![image-20200713095248173](https://i.loli.net/2020/07/13/BhWKOsLmbXN3dkQ.png)

以第一条结果对审计格式说明如下：

| 条目                         | 说明                                                |
| ---------------------------- | --------------------------------------------------- |
| root(monitor)@(172.27.9.246) | 登陆的ip为172.27.9.246，用户为root(通过monitor跳转) |
| [PID:6410]                   | 登陆的pid                                           |
| [LOGIN:2020-07-13 09:37]     | 用户登陆的时间                                      |
| [/root]                      | 执行命令的目录                                      |
| [2020/07/13 09:42:01]        | 命令执行的开始时间                                  |
| df -h                        | 执行的具体命令                                      |
| [0]                          | 命令执行的返回码                                    |
| [2020/07/13 09:42:01]        | 命令执行的完成时间                                  |





**ansible系列文章：**

[ansible UI管理工具awx安装实践](https://blog.51cto.com/3241766/2497420)

