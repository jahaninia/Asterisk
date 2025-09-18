after install rocky linux 9 
pre
## Step 1
```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```
```bash
setenforce 0 
```
---------------
## Step 2

```bash
hostnamectl set-hostname asterisk.example.com
```

## Step 3
```bash
dnf -y install epel-release wget net-tools
```
## Step 4
add repository asterisk
```bash
wget https://ast.tucny.com/repo/tucny-asterisk-el9.repo -O /etc/yum.repos.d/tucny-asterisk-el9.repo
```

## Step 5 

```bash
firewall-cmd --zone=public --add-service sip --permanent && \
firewall-cmd --zone=public --add-port=10000-20000/udp --permanent && \
firewall-cmd --reload
```
