# CentOS7的安装与优化配置

#### 一、 磁盘规划

##### 1.1 目录划分

| **挂载点**     | **大小** | **备注** |
| -------------- | -------- | -------- |
| /boot          | 200M     | 必须创建 |
| /swap          | 12G      | 必须创建 |
| /              | 100G     | 必须创建 |
| /var           | 300G     | 必须创建 |
| /var/log       | 2G       | 必须创建 |
| /var/tmp       | 500M     | 必须创建 |
| /var/log/audit | 1G       | 必须创建 |
| /opt           | 20G      |          |



##### 1.2 建立单独的/tmp分区

设置开机自动挂载

```bash
 systemctl unmask tmp.mount   systemctl enable tmp.mount
```

编辑/etc/systemd/system/local-fs.target.wants/tmp.mount，设置挂载参数

  ```bash
Options=mode=1777,strictatime,noexec,nodev,nosuid  
  ```

挂载

```bash
systemctl daemon-reload  
systemctl start tmp.mount  
```



#### 二、   软件安装

##### 2.1 升级软件

```bash
  yum -y update  
```



##### 2.2 安装常用软件

 ```bash
 yum -y install expect  
 yum -y install vim  
 yum -y install lrzsz  
 yum -y install bash-completion.noarch  
 yum -y install lsof  
 yum -y install unzip  
 ```



#### 三、 建立用户

建立登录用户redhat

```bash
 groupadd -g 1000 redhat  
 useradd -u 1000 -g 1000 -G wheel redhat 
 mkpasswd -l 14
 passwd redhat
```

将redhat加入wheel组，加入wheel组的用户可以使用sudo，

并且配合pam_wheel.so可以只允许改组的用户使用su

usermod -G wheel root  usermod -G wheel redhat  

 

 

#### 四、配置sshd

#####  4.1 备份原始的配置文件

  ```bash
cp /etc/ssh/sshd_config  /etc/ssh/sshd_config.raw  
cp /etc/pam.d/password-auth  /etc/pam.d/password-auth.raw  
cp /etc/pam.d/system-auth  /etc/pam.d/system-auth.raw  
cp /etc/security/pwquality.conf /etc/security/pwquality.conf.raw     
  ```



##### 4.2 编辑  /etc/ssh/sshd_config

```bash
LogLevel INFO  
LoginGraceTime 60  
PermitRootLogin no  
MaxAuthTries 4  
HostbasedAuthentication no  
IgnoreRhosts yes  
PermitEmptyPasswords no  
X11Forwarding no  
PermitUserEnvironment no  
ClientAliveInterval 600  
ClientAliveCountMax 0  
Protocol 2  
AllowUsers redhat  
AllowGroups root redhat 
Banner /etc/issue.net  
```

##### 4.3 配置登录前的警告信息

  ```bash
echo Authorized uses only. All activity  may be monitored and reported. > /etc/issue.net 
echo Authorized uses only. All activity  may be monitored and reported. > /etc/issue  
  ```



#### 五、 PAM配置

##### 5.1 用户密码强度配置

编辑/etc/pam.d/password-auth 和/etc/pam.d/system-auth，确定文件含有

```bash
password requisite   pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=  
```

编辑密码强度配置文件/etc/security/pwquality.conf

```bash
minlen = 14  
dcredit = -1
ucredit = -1
ocredit = -1
lcredit = -1
```



##### 5.2 配置失败密码尝试的锁定

在n次不成功的连续登录尝试后锁定用户ID可减轻暴力对系统的密码攻击

编辑/etc/pam.d/password-auth 和/etc/pam.d/system-auth

```bash
auth required pam_faillock.so preauth audit silent deny=5 unlock_time=900
auth [success=1 default=bad] pam_unix.so
auth [default=die] pam_faillock.so authfail audit deny=5 unlock_time=900
auth sufficient pam_faillock.so authsucc audit deny=5 unlock_time=900
```

 

##### 5.3 限制密码重用次数

编辑/etc/pam.d/password-auth 和/etc/pam.d/system-auth

```bash
password required pam_pwhistory.so remember=5
```

 

确定密码的加密算法 SHA-512

编辑/etc/pam.d/password-auth 和/etc/pam.d/system-auth

```bash
password  sufficient  pam_unix.so   sha512
```

 

限制su的使用，只有wheel组的人才能su到用户

编辑/etc/pam.d/su

```bash
auth required pam_wheel.so use_uid
```

 

#### 六、 审计配置

##### 6.1 配置日志大小及保留策略

vim + /etc/audit/auditd.conf

 ```bash
 max_log_file = 10  num_logs = 50  
 ```

6.2 编辑审计规则

编辑 /etc/audit/rules.d/audit.rules，然后重启

 ```bash
## Make this bigger for busy systems
-b 8192

## Set failure mode to syslog
-f 1

# Ensure events that modify date and time information are collected
-a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time-change
-a always,exit -F arch=b32 -S adjtimex -S settimeofday -S stime -k timechange
-a always,exit -F arch=b64 -S clock_settime -k time-change
-a always,exit -F arch=b32 -S clock_settime -k time-change
-w /etc/localtime -p wa -k time-change

#  Ensure events that modify user/group information are collected
-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/security/opasswd -p wa -k identity

#　Ensure events that modify the system's network environment are
-a always,exit -F arch=b64 -S sethostname -S setdomainname -k system-locale
-a always,exit -F arch=b32 -S sethostname -S setdomainname -k system-locale
-w /etc/issue -p wa -k system-locale
-w /etc/issue.net -p wa -k system-locale
-w /etc/hosts -p wa -k system-locale
-w /etc/sysconfig/network -p wa -k system-locale
-w /etc/sysconfig/network-scripts/ -p wa -k system-locale

#  Ensure login and logout events are collected
-w /var/log/lastlog -p wa -k logins
-w /var/run/faillock/ -p wa -k logins

#　Ensure session initiation information is collected
-w /var/run/utmp -p wa -k session
-w /var/log/wtmp -p wa -k logins
-w /var/log/btmp -p wa -k logins

# Ensure discretionary access control permission modification events are collected
-a always,exit -F arch=b64 -S chmod -S fchmod -S fchmodat -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b32 -S chmod -S fchmod -S fchmodat -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b64 -S chown -S fchown -S fchownat -S lchown -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b32 -S chown -S fchown -S fchownat -S lchown -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b64 -S setxattr -S lsetxattr -S fsetxattr -S removexattr -S lremovexattr -S fremovexattr -F auid>=1000 -F auid!=4294967295 -k perm_mod
-a always,exit -F arch=b32 -S setxattr -S lsetxattr -S fsetxattr -S removexattr -S lremovexattr -S fremovexattr -F auid>=1000 -F auid!=4294967295 -k perm_mod

#  Ensure unsuccessful unauthorized file access attempts are collected
-a always,exit -F arch=b64 -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EACCES -F auid>=1000 -F auid!=4294967295 -k access
-a always,exit -F arch=b32 -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EACCES -F auid>=1000 -F auid!=4294967295 -k access
-a always,exit -F arch=b64 -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EPERM -F auid>=1000 -F auid!=4294967295 -k access
-a always,exit -F arch=b32 -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EPERM -F auid>=1000 -F auid!=4294967295 -k access

# Ensure use of privileged commands is collected
-a always,exit -F path=/usr/bin/wall -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/chfn -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/chsh -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/chage -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/gpasswd -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/newgrp -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/mount -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/su -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/umount -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/write -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/ssh-agent -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/sudo -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/crontab -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/pkexec -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/bin/passwd -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/sbin/unix_chkpwd -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/sbin/pam_timestamp_check -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/sbin/netreport -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/sbin/usernetctl -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/sbin/postdrop -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/sbin/postqueue -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/lib/polkit-1/polkit-agent-helper-1 -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/libexec/utempter/utempter -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/libexec/dbus-1/dbus-daemon-launch-helper -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged
-a always,exit -F path=/usr/libexec/openssh/ssh-keysign -F perm=x -F auid>=1000 -F auid!=4294967295 -k privileged

#  Ensure successful file system mounts are collected
-a always,exit -F arch=b64 -S mount -F auid>=1000 -F auid!=4294967295 -k mounts
-a always,exit -F arch=b32 -S mount -F auid>=1000 -F auid!=4294967295 -k mount

# Ensure file deletion events by users are collected
-a always,exit -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -F auid>=1000 -F auid!=4294967295 -k delete
-a always,exit -F arch=b32 -S unlink -S unlinkat -S rename -S renameat -F auid>=1000 -F auid!=4294967295 -k delete

#  Ensure changes to system administration scope (sudoers) is collected
-w /etc/sudoers -p wa -k scope
-w /etc/sudoers.d/ -p wa -k scope

#  Ensure system administrator actions (sudolog) are collected
-w /var/log/sudo.log -p wa -k actions

# Ensure kernel module loading and unloading is collected
-w /sbin/insmod -p x -k modules
-w /sbin/rmmod -p x -k modules
-w /sbin/modprobe -p x -k modules
-a always,exit -F arch=b64 -S init_module -S delete_module -k modules

#  Ensure the audit configuration is immutable
-e 2
 ```



#### 七、安装AIDE文件完整性检测

##### 7.1 安装

```bash
  yum -y install aide  aide –init  
  mv /var/lib/aide/aide.db.new.gz  /var/lib/aide/aide.db.gz  
```

#####  

##### 7.2 配置自动任务

```bash
 crontab -u root -e
 
0 5 * * * /usr/sbin/aide --check
```







 

 
