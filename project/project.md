# VXLAN. Multihoming.

### Цели:
-Цель проекта: – обеспечить отказоустойчивость и повышение пропускной способности серверного оборудования путем объединения интерфейсов и применении технологий VPC на уровне LEAF коммутаторов.
В случае успешной настройки, происходит равномерное распределение нагрузки на каналы, а так же обеспечиваетя отказоустойчивость, путем уменьшения пропускной способности в случае отказа одного из Leaf коммутаторов.



![Схема сети ](lab7.png)

## Выполнение:
### Подготовка оборудования:
- Необходимо задать коммутаторам адреса управления (mgmt), именно через них планируется организовать передачу пакетов состояния оборудования (heartbeat) и создать DFS группу.
- Указать в качесте STP prymary root коммутатор Leaf1
- Настроить dfs группу, указать пароль аунтетификации, задать адреса доступа, установить приоритет Master>Backup

Leaf1
~~~
interface MEth0/0/0
 ip binding vpn-instance management
 ip address 192.168.2.27 255.255.252.0
#
stp mode rstp
stp v-stp enable
stp instance 0 root primary
stp flush disable
#
dfs-group 1
 authentication-mode hmac-sha256 password 123456
 dual-active detection source ip 192.168.2.27 vpn-instance management
 priority 150
~~~
Leaf2
~~~
interface MEth0/0/0
 ip binding vpn-instance management
 ip address 192.168.2.28 255.255.252.0
#
stp mode rstp
stp v-stp enable
stp flush disable
#
dfs-group 1
 authentication-mode hmac-sha256 password 8mA-Siz8
 dual-active detection source ip 192.168.2.28 vpn-instance management
 priority 120
~~~
- 