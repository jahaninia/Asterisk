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
dnf -y install epel-release wget net-tools mpg123 tcpdump traceroute

```
## Step 4
add repository asterisk
```bash
wget https://ast.tucny.com/repo/tucny-asterisk-el9.repo -O /etc/yum.repos.d/tucny-asterisk-el9.repo
```
## Step 5 
edit file and asterisk 20 is enabled=1
```bash
vi /etc/yum.repos.d/tucny-asterisk-el9.repo
```

## Step 6
install asterisk 
```bash
dnf -y install asterisk asterisk-curl asterisk-fax asterisk-iax2 asterisk-moh-opsound-wav asterisk-mp3 asterisk-sip asterisk-snmp asterisk-sounds-core-en asterisk-voicemail asterisk-dahdi
```
## Step 7

```bash
firewall-cmd --zone=public --add-service sip --permanent && \
firewall-cmd --zone=public --add-port=10000-20000/udp --permanent && \
firewall-cmd --reload
```

## Step 8
```bash
echo -n ";Jahani
[transport-udp]
type=transport
protocol=udp 
bind=0.0.0.0" >> /etc/asterisk/pjsip.conf && \
asterisk -rx "pjsip reload"
```
