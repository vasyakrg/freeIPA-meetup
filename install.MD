# Setup

## install soft

```bash
dnf -y install freeipa-server ipa-server-dns bind-dyndb-ldap curl mc nano htop
```

## setup

```bash
export HNAME="id1.bildme.local"
hostnamectl set-hostname $HNAME --static && hostname $HNAME
```

check /etc/hosts (clear bad records ip4\ip6 with 127.x.x.x)

192.168.9.42 id1.bildme.local id1
192.168.9.45 id2.bildme.local id2

```bash
ipa-server-install
```

SAVE you PASSWORD!! (zAgQ9T2UEtGF0FRMPxIbg)

open ports on master:
```bash
for i in 80 443 389 636 88 464 53; do firewall-cmd --add-port=$i/tcp --permanent; done
for i in 88 464 123 53; do firewall-cmd --add-port=$i/udp --permanent; done
firewall-cmd --reload
```

>TODO: add groups\sudo setting with '!authenticate' and more..

Done.

## replica setup

```bash
dnf -y install freeipa-server ipa-server-dns bind-dyndb-ldap curl mc nano htop

export HNAME="id2.bildme.local"
hostnamectl set-hostname $HNAME --static && hostname $HNAME
```

check /etc/hosts (clear bad records ip4\ip6 with 127.x.x.x)

192.168.9.42 id1.bildme.local id1
192.168.9.45 id2.bildme.local id2

- check dns
set resolv.conf on master replica!

```bash
ipa-client-install --mkhomedir --domain=bildme.local --realm=BILDME.LOCAL --server=id1.bildme.local \
	--ntp-server=91.189.89.198 --ntp-server=91.189.91.157 --ntp-server=91.189.89.199 \
	--hostname=`hostname -f` --enable-dns-updates
```

on ID1!
```bash
kinit admin
ipa hostgroup-add-member ipaservers --hosts id2.bildme.local
```

on ID2!
```bash
for i in 80 443 389 636 88 464 53; do firewall-cmd --add-port=$i/tcp --permanent; done
for i in 88 464 123 53; do firewall-cmd --add-port=$i/udp --permanent; done
firewall-cmd --reload

ipa-replica-install --setup-dns --setup-ca --forwarder 192.168.9.9
```

# Replica teardown

on del replica
```bash
ipa-server-install --uninstall
```

on master:
```bash
ipa-replica-manage del <name> --force
ipa hostgroup-remove-member ipaservers --hosts <name>
```

# Client setup

- check hostname

/etc/hostname = client2.bildme.local

- check dns
set resolv.conf on master\slave replica!

```bash
apt update && apt upgrade -y
apt-get install freeipa-client

ipa-client-install --mkhomedir --hostname=`hostname -f` \
 --server=id1.bildme.local --server=id2.bildme.local --bildme=bildme.local --realm=BILDME.LOCAL \
 --ntp-server=192.168.9.42 --ntp-server=192.168.9.45
 ```

 # Client windows setup

 from AD:
 ```bash
 ipa-getkeytab -s id1.bildme.local -p host/client1.bildme.local -e aes256-cts,aes128-cts,des3-hmac-sha1,des-hmac-sha1,des-cbc-md5,des-cbc-crc -k krb5.keytab.windows -P
 ```

 from client:
```bash
ksetup /setrealm BILDME
ksetup /setbildme BILDME.LOCAL
ksetup /addkdc BILDME.LOCAL id1.bildme.local
ksetup /addkpasswd BILDME.LOCAL id1.bildme.local
ksetup /setcomputerpassword [ПАРОЛЬ]
ksetup /mapuser vasyansk@BILDME.LOCAL vasyansk
```


# Fix warning on ID2

0. `dnf install -y ipa-healthcheck`

1. `chmod 600 /var/log/kadmind.log`
2. fix idrange

ipa-replica-manage dnarange-show
- 762600004-762799999 # (199 995 / 2 = 99 997)

```bash
ipa-replica-manage dnarange-set id1.bildme.local 762600004-762700001
ipa-replica-manage dnarange-set id2.bildme.local 762700002-762799999
```

# Backup catalog

check dns

! set resolv.conf on master replica !

0. add new client to catalog
```bash
ipa-client-install --mkhomedir --bildme=bildme.local --realm=BILDME.LOCAL --server=id1.bildme.local \
	--ntp-server=91.189.89.198 --ntp-server=91.189.91.157 --ntp-server=91.189.89.199 \
	--hostname=`hostname -f` --enable-dns-updates
```

- check new records of its replica in DNS-masters

- open ports
```bash
for i in 80 443 389 636 88 464 53; do firewall-cmd --add-port=$i/tcp --permanent; done
for i in 88 464 123 53; do firewall-cmd --add-port=$i/udp --permanent; done
firewall-cmd --reload
```

- add its replica to ipaservers group

1. add hidden replica for backup (if you have more resource)
```bash
ipa-replica-install --hidden-replica
```

# Restore replica

- check dns, ip, hostname and version of ipa packages
- install ipaserver if you need

```bash
ipa-restore /var/lib/ipa/backup/ipa-full-2022-03-07-10-42-22/
```

run ipa-healthcheck get error of replication

```bash
ipa-replica-manage re-initialize --from id1.bildme.local
ipa-replica-manage force-sync --from id1.bildme.local
ipactl restart
```

# Auth application LDAP connect

https://docs.gitlab.com/ee/administration/auth/ldap/

- add uidgitlab to system users by ipa-ctl.sh
- add ENV's config to you gitlab app

```bash
LDAP_ENABLED=true
LDAP_LABEL=LDAP

# "start_tls" if 389, or "simple_tls" if 636
LDAP_METHOD=simple_tls
LDAP_ACTIVE_DIRECTORY=false
LDAP_HOST=id1.bildme.local
LDAP_PORT=636
LDAP_UID=uid
LDAP_PASS=zAgQ9T2UEtGF0FRMPxIbg
LDAP_VERIFY_SSL=false
LDAP_BASE="cn=users,cn=accounts,dc=bildme,dc=local"
LDAP_USER_FILTER="(&(objectClass=posixaccount)(memberOf=cn=gitlab,cn=groups,cn=accounts,dc=bildme,dc=local))"
LDAP_BIND_DN="uid=uidgitlab,cn=sysaccounts,cn=etc,dc=bildme,dc=local"
```

### Bonuses
Migrate users from AD
https://github.com/akeske/AD-to-FreeIPA-user-migration/blob/master/ad2ipa.py

Freeipa behind ssl-proxy
https://www.adelton.com/freeipa/freeipa-behind-proxy-with-different-name
