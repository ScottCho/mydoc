### 一、 安装依赖

```bash
sudo yum install -y curl policycoreutils-python openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo systemctl reload firewalld
```



安装postfix(发送邮件)

```bash
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
```



### 二、 增加版本库及安装

#### 2.1 安装

端口最好设置成80

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo EXTERNAL_URL="http://192.168.0.66:8090" yum install -y gitlab-ce
```



#### 2.2 访问

登录地址： http://192.168.0.66:8090

默认登录账户：root





参考：https://about.gitlab.com/install/#centos-7