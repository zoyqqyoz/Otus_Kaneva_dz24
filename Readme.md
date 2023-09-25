Vagrant-стенд c VLAN и LACP

Цель домашнего задания
Научиться настраивать VLAN и LACP. 

Описание домашнего задания
в Office1 в тестовой подсети появляется сервера с доп интерфесами и адресами
в internal сети testLAN: 
- testClient1 - 10.10.10.254
- testClient2 - 10.10.10.254
- testServer1- 10.10.10.1 
- testServer2- 10.10.10.1

Равести вланами:
testClient1 <-> testServer1
testClient2 <-> testServer2

Между centralRouter и inetRouter "пробросить" 2 линка (общая inernal сеть) и объединить их в бонд, проверить работу c отключением интерфейсов

Формат сдачи ДЗ - vagrant + ansible


Разворачиваем 7 ВМ из следующего Vagrantfile:

``
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :inetRouter => {
        :box_name => "centos/8",
        :vm_name => "inetRouter",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "router-net"},
                   {adapter: 3, auto_config: false, virtualbox__intnet: "router-net"},
                   {ip: '192.168.56.10', adapter: 8},
                ]
  },
  :centralRouter => {
        :box_name => "centos/8",
        :vm_name => "centralRouter",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "router-net"},
                   {adapter: 3, auto_config: false, virtualbox__intnet: "router-net"},
                   {ip: '192.168.255.9', adapter: 6, netmask: "255.255.255.252", virtualbox__intnet: "office1-central"},
                   {ip: '192.168.56.11', adapter: 8},
                ]
  },

  :office1Router => {
        :box_name => "centos/8",
        :vm_name => "office1Router",
        :net => [
                   {ip: '192.168.255.10', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "office1-central"},
                   {adapter: 3, auto_config: false, virtualbox__intnet: "vlan1"},
                   {adapter: 4, auto_config: false, virtualbox__intnet: "vlan1"},
                   {adapter: 5, auto_config: false, virtualbox__intnet: "vlan2"},
                   {adapter: 6, auto_config: false, virtualbox__intnet: "vlan2"},
                   {ip: '192.168.56.20', adapter: 8},
                ]
  },

  :testClient1 => {
        :box_name => "centos/8",
        :vm_name => "testClient1",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "testLAN"},
                   {ip: '192.168.56.21', adapter: 8},
                ]
  },

  :testServer1 => {
        :box_name => "centos/8",
        :vm_name => "testServer1",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "testLAN"},
                   {ip: '192.168.56.22', adapter: 8},
            ]
  },

  :testClient2 => {
        :box_name => "ubuntu/focal64",
        :box_version => "20220411.2.0",
        :vm_name => "testClient2",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "testLAN"},
                   {ip: '192.168.56.31', adapter: 8},
                ]
  },

  :testServer2 => {
        :box_name => "ubuntu/focal64",
        :box_version => "20220411.2.0",
        :vm_name => "testServer2",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "testLAN"},
                   {ip: '192.168.56.32', adapter: 8},
                ]
  },

}

Vagrant.configure("2") do |config|
config.vm.synced_folder ".", "/vagrant", disabled: true
  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      box.vm.box_version = boxconfig[:box_version]

      config.vm.provider "virtualbox" do |v|
        v.memory = 1024
        v.cpus = 2
       end

      if boxconfig[:vm_name] == "testServer2"
       box.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/provision.yml"
        ansible.inventory_path = "ansible/hosts"
        ansible.host_key_checking = "false"
        ansible.become = "true"
        ansible.limit = "all"
       end
      end

      boxconfig[:net].each do |ipconf|
        box.vm.network "private_network", **ipconf
      end

      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
        cp ~vagrant/.ssh/auth* ~root/.ssh
      SHELL
    end
  end
end
neva@Uneva:~$ pwd
/home/neva
neva@Uneva:~$ mc

neva@Uneva:~$ cat Vagrantfile
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :inetRouter => {
        :box_name => "centos/8",
        :vm_name => "inetRouter",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "router-net"},
                   {adapter: 3, auto_config: false, virtualbox__intnet: "router-net"},
                   {ip: '192.168.56.10', adapter: 8},
                ]
  },
  :centralRouter => {
        :box_name => "centos/8",
        :vm_name => "centralRouter",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "router-net"},
                   {adapter: 3, auto_config: false, virtualbox__intnet: "router-net"},
                   {ip: '192.168.255.9', adapter: 6, netmask: "255.255.255.252", virtualbox__intnet: "office1-central"},
                   {ip: '192.168.56.11', adapter: 8},
                ]
  },

  :office1Router => {
        :box_name => "centos/8",
        :vm_name => "office1Router",
        :net => [
                   {ip: '192.168.255.10', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "office1-central"},
                   {adapter: 3, auto_config: false, virtualbox__intnet: "vlan1"},
                   {adapter: 4, auto_config: false, virtualbox__intnet: "vlan1"},
                   {adapter: 5, auto_config: false, virtualbox__intnet: "vlan2"},
                   {adapter: 6, auto_config: false, virtualbox__intnet: "vlan2"},
                   {ip: '192.168.56.20', adapter: 8},
                ]
  },

  :testClient1 => {
        :box_name => "centos/8",
        :vm_name => "testClient1",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "testLAN"},
                   {ip: '192.168.56.21', adapter: 8},
                ]
  },

  :testServer1 => {
        :box_name => "centos/8",
        :vm_name => "testServer1",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "testLAN"},
                   {ip: '192.168.56.22', adapter: 8},
            ]
  },

  :testClient2 => {
        :box_name => "ubuntu/focal64",
        :box_version => "20220411.2.0",
        :vm_name => "testClient2",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "testLAN"},
                   {ip: '192.168.56.31', adapter: 8},
                ]
  },

  :testServer2 => {
        :box_name => "ubuntu/focal64",
        :box_version => "20220411.2.0",
        :vm_name => "testServer2",
        :net => [
                   {adapter: 2, auto_config: false, virtualbox__intnet: "testLAN"},
                   {ip: '192.168.56.32', adapter: 8},
                ]
  },

}

Vagrant.configure("2") do |config|
config.vm.synced_folder ".", "/vagrant", disabled: true
  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxconfig[:vm_name]
      box.vm.box_version = boxconfig[:box_version]

      config.vm.provider "virtualbox" do |v|
        v.memory = 1024
        v.cpus = 2
       end

      if boxconfig[:vm_name] == "testServer2"
       box.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/provision.yml"
        ansible.inventory_path = "ansible/hosts"
        ansible.host_key_checking = "false"
        ansible.become = "true"
        ansible.limit = "all"
       end
      end

      boxconfig[:net].each do |ipconf|
        box.vm.network "private_network", **ipconf
      end

      box.vm.provision "shell", inline: <<-SHELL
        mkdir -p ~root/.ssh
        cp ~vagrant/.ssh/auth* ~root/.ssh
      SHELL
    end
  end
end


Проверяем результат:

```
neva@Uneva:~$ vagrant status
Current machine states:

inetRouter                running (virtualbox)
centralRouter             running (virtualbox)
office1Router             running (virtualbox)
testClient1               running (virtualbox)
testServer1               running (virtualbox)
testClient2               running (virtualbox)
testServer2               running (virtualbox)
```

Далее на все хосты нужно установить vim, traceroute, tcpdump, net-tools. Лучше всего сделать это через Ansible.

```
- name: Base set up
  #Настройка производится на всех хостах
  hosts: all
  become: yes
  tasks:
  - name: set up repo1
    replace:
      path: "{{ item }}"
      regexp: 'mirrorlist'
      replace: '#mirrorlist'
    with_items:
      - /etc/yum.repos.d/CentOS-Linux-AppStream.repo
      - /etc/yum.repos.d/CentOS-Linux-BaseOS.repo
    when: (ansible_os_family == "RedHat")

  - name: set up repo2
    replace:
      path: "{{ item }}"
      regexp: '#baseurl=http://mirror.centos.org'
      replace: 'baseurl=http://vault.centos.org'
    with_items:
      - /etc/yum.repos.d/CentOS-Linux-AppStream.repo
      - /etc/yum.repos.d/CentOS-Linux-BaseOS.repo
    when: (ansible_os_family == "RedHat")

  #Установка приложений на RedHat-based системах
  - name: install softs on CentOS
    yum:
      name:
        - vim
        - traceroute
        - tcpdump
        - net-tools
      state: present
      update_cache: true
    when: (ansible_os_family == "RedHat")

  #Установка приложений на Debiam-based системах
  - name: install softs on Debian-based
    apt:
      name:
        - vim
        - traceroute
        - tcpdump
        - net-tools
      state: present
      update_cache: true
    when: (ansible_os_family == "Debian")
```

Настройка VLAN на хостах

Настройка VLAN на RHEL-based системах:

На хосте testClient1 требуется создать файл /etc/sysconfig/network-scripts/ifcfg-vlan1 со следующим параметрами:

```
VLAN=yes
#Тип интерфеса - VLAN
TYPE=Vlan
#Указываем фиическое устройство, через которые будет работь VLAN
PHYSDEV=eth1
#Указываем номер VLAN (VLAN_ID)
VLAN_ID=1
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
#Указываем IP-адрес интерфейса
IPADDR=10.10.10.254
#Указываем префикс (маску) подсети
PREFIX=24
#Указываем имя vlan
NAME=vlan1
#Указываем имя подинтерфейса
DEVICE=eth1.1
ONBOOT=yes
```

На хосте testServer1 создадим идентичный файл с другим IP-адресом (10.10.10.1).
После создания файлов нужно перезапустить сеть на обоих хостах:

```
systemctl restart NetworkManager
```

Проверим настройку интерфейса, если настройка произведена правильно, то с хоста testClient1 будет проходить ping до хоста testServer1:

```
[root@testClient1 ~]# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=1.65 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.741 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.886 ms

[root@testServer1 ~]# ping 10.10.10.254
PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=1.09 ms
64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.545 ms
64 bytes from 10.10.10.254: icmp_seq=3 ttl=64 time=0.555 ms
^C
--- 10.10.10.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 56ms
```

Настройка VLAN на Ubuntu:

На хосте testClient2 требуется отредактировать файл /etc/netplan/50-cloud-init.yaml, приведя его к следующему виду:

```
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    ethernets:
        enp0s3:
            dhcp4: true
            match:
                macaddress: 02:c2:0a:9b:9b:f5
            set-name: enp0s3
        enp0s8: {}
    vlans:
        vlan2:
          id: 2
          link: enp0s8
          dhcp4: no
          addresses: [10.10.10.254/24]
    version: 2
```

На хосте testServer2 идентично редактируем файл, указывая IP-адрес 10.10.10.1.
После создания файлов нужно перезапустить сеть на обоих хостах: netplan apply

После настройки второго VLAN`а ping должен работать между хостами testClient1, testServer1 и между хостами testClient2, testServer2.

Примечание: до остальных хостов ping работать не будет, так как не настроена маршрутизация. 

Проверяем:

```
root@testClient2:/etc/netplan# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=3.33 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=1.38 ms

root@testServer2:~# ping 10.10.10.254
PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=1.04 ms
64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=0.677 ms
^C
--- 10.10.10.254 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.677/0.860/1.044/0.183 ms
```

Настройка VLAN с помощью Ansible:

Настройка VLAN 1 на хостах testClient1 и testServer1

Для настройки  VLAN 1 добавляем в provision.yml:

```
- name: set up vlan1
  #Настройка будет производиться на хостах testClient1 и testServer1
  hosts: testClient1,testServer1
  #Настройка производится от root-пользователя
  become: yes
  tasks:
  #Добавление темплейта в файл /etc/sysconfig/network-scripts/ifcfg-vlan1
  - name: set up vlan1
    template:
      src: ifcfg-vlan1.j2
      dest: /etc/sysconfig/network-scripts/ifcfg-vlan1
      owner: root
      group: root
      mode: 0644
  
  #Перезапуск службы NetworkManager
  - name: restart network for vlan1
    service:
      name: NetworkManager
      state: restarted
```
Чтобы не делать 2 отдельных конфигурационных файла, можно сделать template, который автоматически поменяет IP-адрес и VLAN_ID:

```
VLAN=yes
TYPE=Vlan
PHYSDEV=eth1
VLAN_ID={{ vlan_id }}
VLAN_NAME_TYPE=DEV_PLUS_VID_NO_PAD
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
IPADDR={{ vlan_ip }}
PREFIX=24
NAME=vlan{{ vlan_id }}
DEVICE=eth1.{{ vlan_id }}
ONBOOT=yes
```

Параметры vlan_id и vlan_ip можно указать в файле hosts: 

```
[nets]
inetRouter ansible_host=192.168.56.10 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/inetRouter/virtualbox/private_key bond_ip=192.168.255.1
centralRouter ansible_host=192.168.56.11 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/centralRouter/virtualbox/private_key bond_ip=192.168.255.2
office1Router ansible_host=192.168.56.20 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/office1Router/virtualbox/private_key 
testClient1 ansible_host=192.168.56.21 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/testClient1/virtualbox/private_key vlan_id=1 vlan_ip=10.10.10.254
testServer1 ansible_host=192.168.56.22 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/testServer1/virtualbox/private_key vlan_id=1 vlan_ip=10.10.10.1
testClient2 ansible_host=192.168.56.31 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/testClient2/virtualbox/private_key vlan_id=2 vlan_ip=10.10.10.254
testServer2 ansible_host=192.168.56.32 ansible_user=vagrant ansible_ssh_private_key_file=./.vagrant/machines/testServer2/virtualbox/private_key vlan_id=2 vlan_ip=10.10.10.1
```

Настройка VLAN 2 на хостах testClient2 и testServer2:
Добавляем в provision.yml:

```
- name: set up vlan2
  hosts: testClient2,testServer2
  become: yes
  tasks:
  - name: set up vlan2
    template:
      src: 50-cloud-init.yaml.j2
      dest: /etc/netplan/50-cloud-init.yaml 
      owner: root
      group: root
      mode: 0644

  - name: apply set up vlan2
    shell: netplan apply
    become: true
```

Чтобы не делать 2 отдельных конфигурационных файла, можно сделать template, который автоматически поменяет IP-адрес и VLAN_ID:

```
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets:
        enp0s3:
            dhcp4: true
     
        enp0s8: {}
    
    vlans:
        vlan{{ vlan_id }}:
          id: {{ vlan_id }}
          link: enp0s8
          dhcp4: no
          addresses: [{{ vlan_ip }}/24]
```

Параметры vlan_id и vlan_ip также указаны в файле hosts.

Проверяем:
```
neva@Uneva:~/Otus_Kaneva_dz24/ansible/templates$ vagrant provision

PLAY [Base set up] *************************************************************

TASK [Gathering Facts] *********************************************************
ok: [testServer2]
ok: [testClient2]

TASK [set up vlan2] ************************************************************
changed: [testServer2]
changed: [testClient2]

TASK [apply set up vlan2] ******************************************************
changed: [testClient2]
changed: [testServer2]

PLAY RECAP *********************************************************************
testClient2                : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
testServer2                : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

root@testClient2:/etc/netplan# ping 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=1.60 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.752 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.617 ms
^C


root@testServer2:~# ping 10.10.10.254
PING 10.10.10.254 (10.10.10.254) 56(84) bytes of data.
64 bytes from 10.10.10.254: icmp_seq=1 ttl=64 time=0.716 ms
64 bytes from 10.10.10.254: icmp_seq=2 ttl=64 time=2.35 ms
```

Настройка LACP между хостами inetRouter и centralRouter

Bond интерфейс будет работать через порты eth1 и eth2. 

1) Изначально необходимо на обоих хостах добавить конфигурационные файлы для интерфейсов eth1 и eth2:

```
[root@inetRouter ~]# vim /etc/sysconfig/network-scripts/ifcfg-eth1
#Имя физического интерфейса
DEVICE=eth1
#Включать интерфейс при запуске системы
ONBOOT=yes
#Отключение DHCP-клиента
BOOTPROTO=none
#Указываем, что порт часть bond-интерфейса
MASTER=bond0
#Указыаваем роль bond
SLAVE=yes
NM_CONTROLLED=yes
USERCTL=no
```

У интерфейса ifcfg-eth2 идентичный конфигурационный файл, в котором нужно изменить имя интерфейса.
2) После настройки интерфейсов eth1 и eth2 нужно настроить bond-интерфейс, для этого создадим файл /etc/sysconfig/network-scripts/ifcfg-bond0

```
vim /etc/sysconfig/network-scripts/ifcfg-bond0

[root@inetRouter ~]# cat /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
NAME=bond0
#Тип интерфейса — bond
TYPE=Bond
BONDING_MASTER=yes
#Указаваем IP-адрес 
IPADDR=192.168.255.1
#Указываем маску подсети
NETMASK=255.255.255.252
ONBOOT=yes
BOOTPROTO=static
#Указываем режим работы bond-интерфейса Active-Backup
# fail_over_mac=1 — данная опция «разрешает отвалиться» одному интерфейсу
BONDING_OPTS="mode=1 miimon=100 fail_over_mac=1"
NM_CONTROLLED=yes


[root@centralRouter ~]# cat /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
NAME=bond0
#Тип интерфейса — bond
TYPE=Bond
BONDING_MASTER=yes
#Указаваем IP-адрес
IPADDR=192.168.255.2
#Указываем маску подсети
NETMASK=255.255.255.252
ONBOOT=yes
BOOTPROTO=static
#Указываем режим работы bond-интерфейса Active-Backup
# fail_over_mac=1 — данная опция «разрешает отвалиться» одному интерфейсу
BONDING_OPTS="mode=1 miimon=100 fail_over_mac=1"
NM_CONTROLLED=yes


```

После создания данных конфигурационных файлов неоьходимо перзапустить сеть:

```
systemctl restart NetworkManager
```
На некоторых версиях RHEL/CentOS перезапуск сетевого интерфейса не запустит bond-интерфейс, в этом случае рекомендуется перезапустить хост.
После настройки агрегации портов, необходимо проверить работу bond-интерфейса, для этого, на хосте inetRouter (192.168.255.1) запустим ping до centralRouter (192.168.255.2):

```

[vagrant@inetRouter ~]$ ping 192.168.255.2
PING 192.168.255.2 (192.168.255.2) 56(84) bytes of data.
64 bytes from 192.168.255.2: icmp_seq=1 ttl=64 time=2.03 ms
64 bytes from 192.168.255.2: icmp_seq=2 ttl=64 time=0.781 ms
64 bytes from 192.168.255.2: icmp_seq=3 ttl=64 time=0.747 ms
^C
--- 192.168.255.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 19ms
```

и обратно:

```
[vagrant@centralRouter ~]$ ping 192.168.255.1
PING 192.168.255.1 (192.168.255.1) 56(84) bytes of data.
64 bytes from 192.168.255.1: icmp_seq=1 ttl=64 time=1.35 ms
64 bytes from 192.168.255.1: icmp_seq=2 ttl=64 time=0.959 ms
64 bytes from 192.168.255.1: icmp_seq=3 ttl=64 time=0.825 ms
^C
--- 192.168.255.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6ms
rtt min/avg/max/mdev = 0.825/1.043/1.346/0.222 ms
```

Не отменяя ping подключаемся к хосту centralRouter и выключаем там интерфейс eth1: 

```
[root@centralRouter ~]# ip link set down eth1
```

Пинг продолжает идти, т. к. трафик пошёл по второму интерфейсу бонда.

Настройка LACP между хостами inetRouter и centralRouter с помощью Ansible
Добавляем в provision.yml:

```
- name: set up bond0
  hosts: inetRouter,centralRouter
  become: yes
  tasks:
  - name: set up ifcfg-bond0
    template:
      src: ifcfg-bond0.j2
      dest: /etc/sysconfig/network-scripts/ifcfg-bond0
      owner: root
      group: root
      mode: 0644
  
  - name: set up eth1,eth2
    copy: 
      src: "{{ item }}" 
      dest: /etc/sysconfig/network-scripts/
      owner: root
      group: root
      mode: 0644
    with_items:
      - templates/ifcfg-eth1
      - templates/ifcfg-eth2
  #Перезагрузка хостов 
  - name: restart hosts for bond0
    reboot:
      reboot_timeout: 3600
```

Чтобы не создавать 2 конфигурационных файла для bond-интерфейса можно сделать template:

```
DEVICE=bond0
NAME=bond0
TYPE=Bond
BONDING_MASTER=yes
IPADDR={{ bond_ip }}
NETMASK=255.255.255.252
ONBOOT=yes
BOOTPROTO=static
BONDING_OPTS="mode=1 miimon=100 fail_over_mac=1"
NM_CONTROLLED=yes
USERCTL=no
```

Создадим так же темплейты для интерфейсов eth1 и eth2 (ifcfg-eth1 и ifcfg-eth2):

```
DEVICE=eth1
ONBOOT=yes
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=yes

DEVICE=eth2
ONBOOT=yes
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=yes
```

Проверяем результат командой vagrant provision:

```
PLAY [Base set up] *************************************************************

TASK [Gathering Facts] *********************************************************
ok: [testClient2]
ok: [testServer2]

TASK [set up vlan2] ************************************************************
ok: [testServer2]
ok: [testClient2]

TASK [apply set up vlan2] ******************************************************
changed: [testClient2]
changed: [testServer2]

PLAY [set up bond0] ************************************************************

TASK [Gathering Facts] *********************************************************
ok: [centralRouter]
ok: [inetRouter]

TASK [set up ifcfg-bond0] ******************************************************
changed: [centralRouter]
changed: [inetRouter]

TASK [set up eth1,eth2] ********************************************************
changed: [centralRouter] => (item=templates/ifcfg-eth1)
changed: [inetRouter] => (item=templates/ifcfg-eth1)
changed: [centralRouter] => (item=templates/ifcfg-eth2)
changed: [inetRouter] => (item=templates/ifcfg-eth2)

TASK [restart hosts for bond0] *************************************************
changed: [inetRouter]
changed: [centralRouter]

PLAY RECAP *********************************************************************
centralRouter              : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
inetRouter                 : ok=4    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
testClient2                : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
testServer2                : ok=3    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```



