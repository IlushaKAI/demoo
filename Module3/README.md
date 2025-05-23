tags: [[demo]]

## Задание 3

Перенастройте ip-туннель с базового до уровня туннеля, обеспечивающего шифрование трафика
- Настройте защищенный туннель между HQ-RTR и BR-RTR
- Внесите необходимые изменения в конфигурацию динамической
маршрутизации, протокол динамической маршрутизации должен
возобновить работу после перенастройки туннеля
- Выбранное программное обеспечение, обоснование его выбора и его
основные параметры, изменения в конфигурации динамической
маршрутизации отметьте в отчёте

Настройку выполняем на HQ-RTR и BR-RTR
Устанавливаем StrongSwan - легкий способ реализации IKE протоколов для создания защищенного соединения
```bash
apt install strongswan
```

Переходим в конфиг файл туннеля
```bash
nano /etc/ipsec.conf
```

И указываем конфигурацию туннеля:
- параметр `left` - Ip адрес интерфейса смотрящего в сторону isp на роутере, который настраиваем.
- параметр `right` - Ip адрес интерфейса смотрящего в сторону isp на удаленном роутере.

```bash
config setup
        # strictcrlpolicy=yes
        # uniqueids = no
        charondebug="ike 2, knl 2, cfg 2, net 2"
# Add connections here.
conn gre-over-ipsec
        auto=start
        left=172.16.4.2
        right=172.16.5.2
        authby=psk
        type=transport
        keyexchange=ikev2
        ike=aes256-sha1-modp1024
        esp=aes256-sha1
        leftprotoport=47
        rightprotoport=47

```

Затем необходимо указать пароль, по которому наши роутеры будут поднимать туннель. Для этого открываем файл секретов
```bash
nano /etc/ipsec.secrets
```

и непосредственно пишем пароль, где по аналогии с прошлой настройкой:
- слева - ip адрес нашего роутера
- справа - ip адрес удаленного роутера
```bash
172.16.4.2 172.16.5.2 : PSK "StrongSharedKey123"
```

Производим аналогичную настройку на BR-RTR, просто меняя местами ip адреса

и на обоих роутерах сначала перезагружаем службу
```bash
systemctl restart strongswan-starter
```
затем смотрим результат настроек
```bash
sudo ipsec status
```
*если интерфейс не подключиnя, то выключайте ipsec, иначе у вас вся сеть не будет работать*
```
sudo ipsec stop
```
## Задание 4
Настройте межсетевой экран на маршрутизаторах HQ-RTR и BR-RTR на
сеть в сторону ISP
- Обеспечьте работу протоколов http, https, dns, ntp, icmp или
дополнительных нужных протоколов
- Запретите остальные подключения из сети Интернет во внутреннюю
сеть

Запрещаем входящие пакеты, разрешаем исходящие, разрешаем необходимые порты, на которых по умолчанию работают службы
80 и 443 - http и https
53 - DNS
123 - NTP
icmp по умолчанию разрешен в ufw

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 53/udp
sudo ufw allow 53/tcp
sudo ufw allow 123/udp
```
Эту настройку выполняем на HQ-RTR и BR-RTR
## Задание 5
Настройте принт-сервер cups на сервере HQ-SRV.
- Опубликуйте виртуальный pdf-принтер
- На клиенте HQ-CLI подключите виртуальный принтер как принтер по умолчанию
Смотрим и думаем сами
https://dmr.md/2019/05/12/%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-%D0%B8-%D0%BD%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0-%D0%BF%D1%80%D0%B8%D0%BD%D1%82-%D1%81%D0%B5%D1%80%D0%B2%D0%B5%D1%80%D0%B0-cups/

https://github.com/NyashMan/DEMO2024/blob/main/README.md#%D1%80%D0%B0%D1%81%D0%BF%D0%B8%D1%81%D0%B0%D1%82%D1%8C-%D0%BF%D1%83%D0%BD%D0%BA%D1%82

## Задание 8
Для правильной работы этого задания, необходимо правильно настроить ssh и ansible во втором модуле


Реализуйте механизм инвентаризации машин HQ-SRV и HQ-CLI через Ansible на BR-SRV:
Перед выполнение этого задания, проверьте, что вы можете без проблем поиграть в пинг-понг от лица `sshuser` с необходимыми машинами. Это мы настраиваем в 4-ом задании 2-ого модуля
```bash
ansible all -i /etc/ansible/demo -m ping
```
В случае неудачной партии с необходимыми машинами проверьте настройку того задания
<br/>
от лица sshuser создаем файл `/etc/ansible/inventory_pc.yml` и в нем пишем
```yml
- name: Инвентаризация рабочих станций HQ-SRV и HQ-CLI
  hosts:
    - 10.0.0.2
    - 10.0.3.3
  gather_facts: yes

  tasks:
    - name: Собрать основные данные и сохранить в YAML файл
      copy:
        content: |
          hostname: {{ ansible_hostname }}
          ip_addresses: {{ ansible_all_ipv4_addresses }}
        dest: "/etc/ansible/PC_INFO/{{ ansible_hostname }}.yml"
      delegate_to: localhost
```

Также не забываем самостоятельно заранее создать директорию 
```bash
mkdir /etc/ansible/PC_INFO
```

Запускаем задачу инвентаризации командой:
```shell
ansible-playbook -i /etc/ansible/demo /etc/ansible/inventory_pc.yml
```

## Задание 9
Реализуйте механизм резервного копирования конфигурации для машин HQ-RTR и BR-RTR, через Ansible на BR-SRV


Теже начальные требования что и в 8-ом задании + net_admin на обоих роутерах должен иметь возможность запускать sudo без пароля. Напомню, для этого надо:


в файле `/etc/sudoers` написать:
```
net_admin    ALL=(ALL:ALL)    NOPASSWD: ALL
```
пробелы сделаны TABом


от лица sshuser создаем файл `/etc/ansible/backup_network.yml` и в нем пишем

```yml
- name: Резервное копирование конфигурации роутеров
  hosts:
    - 172.16.4.2
    - 172.16.5.2
  gather_facts: no
  become: true

  tasks:

    - name: Получить конфигурацию FRR (динамическая маршрутизация)
      fetch:
        src: /etc/frr/frr.conf
        dest: "/etc/ansible/NETWORK_INFO/{{ inventory_hostname }}_frr.conf"
        flat: yes

    - name: Получить конфигурацию UFW (before.rules)
      fetch:
        src: /etc/ufw/before.rules
        dest: "/etc/ansible/NETWORK_INFO/{{ inventory_hostname }}_ufw_before.rules"
        flat: yes

    - name: Выполнить команду ip -c a и сохранить вывод
      command: "ip -c a"
      register: ip_output

    - name: Сохранить вывод команды 'ip -c a' в файл
      copy:
        content: "{{ ip_output.stdout }}"
        dest: "/etc/ansible/NETWORK_INFO/{{ inventory_hostname }}_ip_a.txt"
      delegate_to: localhost
```



Также не забываем самостоятельно заранее создать директорию

```bash
mkdir /etc/ansible/NETWORK_INFO
```

Запускаем задачу инвентаризации командой:

```bash
ansible-playbook -i /etc/ansible/demo /etc/ansible/backup_network.yml
```