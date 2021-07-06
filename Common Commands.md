# Common Commands
```shell
docker rm \`docker ps -a | grep Exited | awk '{print $1}'\` # 删除异常停止的docker容器


docker images | grep [关键字] | awk '{printf "%s:%s\n",$1,$2}' # 筛选镜像

docker rmi -f \`docker images | grep '<none>' | awk '{print $3}'\` # 删除无用镜像

find . -name "*.go"  | xargs grep -v "^$"| wc -l  # 统计代码行数

```

## ldapadd
demo.ldif:
```
dn: cn=Lxj,ou=People,dc=njuics,dc=cn
cn: Lxj
gidnumber: 500
givenname: lxj
homedirectory: /home/users/lxj
loginshell: /bin/bash
mail: lxj@nju.edu.cn
objectclass: inetOrgPerson
objectclass: posixAccount
objectclass: top
sn: lxj
uid: lxj
uidnumber: 1038
userpassword: {MD5}MG8MM44sPRsKG5kMAX79fw==
```
command:
```shell
ldapadd -x -W -D "cn=admin,dc=njuics,dc=cn" -f demo.ldif
```
## 查看证书过期时间
```shell
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep ' Not '
kubeadm alpha certs check-expiration
# crontab -e
30 7 1 1,7 * /usr/bin/kubeadm alpha certs renew all
```