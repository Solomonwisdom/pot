# cephadm安装ceph Octopus
### hosts
```
cat << EOF >> /etc/hosts
114.212.189.131 ics01
114.212.189.132 ics02
114.212.189.133 ics03
114.212.189.134 ics04
114.212.189.135 ics05
114.212.189.136 ics06
114.212.189.137 ics07
114.212.189.138 ics08
114.212.189.139 ics09
114.212.189.140 ics10
114.212.189.141 ics11
114.212.189.142 ics12
114.212.189.143 ics13
114.212.189.144 ics14
114.212.189.145 ics15
114.212.189.146 ics16
EOF
```

```
yum install -y ntp ntpdate ntp-doc
systemctl enable --now ntpd
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x ./cephadm
mv ./cephadm /usr/local/bin
mkdir -p /etc/ceph
cephadm bootstrap --mon-ip 114.212.189.146
ceph orch host add ics15
ceph orch host label add ics16 mon
ceph orch apply mon label:mon
ceph orch device ls
ceph orch daemon add osd ics16:/dev/sdb2
ceph fs volume create cephfs ics13
#ceph orch apply mds *<fs-name>* --placement="*<num-daemons>* [*<host1>* ...]"
mount.ceph 114.212.189.143,114.212.189.144,114.212.189.145,114.212.189.146:/ /mnt/cephfs -o name=admin,secret=AQB5yHpfwZjmERAAiRSVYFzFEatLT2mH2urD0g==
```

```shell
yum install -y ntp ntpdate ntp-doc
systemctl enable --now ntpd
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x ./cephadm
mv ./cephadm /usr/local/bin
cephadm add-repo --release octopus
sed -i 's/download.ceph.com/mirrors.aliyun.com\/ceph/g' /etc/yum.repos.d/ceph.repo 
mkdir -p /etc/ceph
cephadm --docker bootstrap --mon-ip 210.28.132.167 --allow-overwrite
ceph shell
ssh-copy-id -f -i /etc/ceph/ceph.pub root@n168
ceph orch host add n168
ceph orch host label add n167 mon
ceph orch apply mon label:mon
ceph orch device ls
ceph orch daemon add osd n167:/dev/sdd1
ceph orch daemon add osd n168:/dev/sdd1
ceph orch daemon add osd n169:/dev/sdd1
ceph fs volume create cephfs n169,n168
# ceph orch apply mds cephfs --placement=2 [*<host1>* ...]
# 挂载ceph文件系统
mount.ceph 210.28.132.167,210.28.132.168,210.28.132.169:/ /mnt/cephfs -o name=admin,secret=AQCndd1fPoXSKBAANC6ixLlP1J38ILpKLDXRUw==
Ceph Dashboard is now available at:

	     URL: https://n169.njuics.cn:8443/
	    User: admin
	Password: g7y40kqm5c
```