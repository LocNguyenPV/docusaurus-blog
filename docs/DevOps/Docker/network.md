```
docker network create -d macvlan --subnet=192.168.110.0/24 --ip-range=192.168.110.0/24 --gateway=192.168.110.1 -o macvlan_mode=bridge -o parent=ens33 VLAN110
```