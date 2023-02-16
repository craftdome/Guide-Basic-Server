# Guide-Basic-Server

## Шаг 1. Обновление
1. Авторизуемся в системе и входим под учёткой root.
```
su -
```

2. Установка пакетов.
```
apt install iptables iptables-persistent ipset tree git htop screen
apt update -y && apt upgrade -y
apt remove sudo --purge
```

## Шаг 2. Защита пользователя. Настройка SSH
> Требуется root `su -`

1. Создание нового пользователя и установка пароля.
```
useradd -m -G adm,cdrom,sudo -s /bin/bash user
passwd user
su user
```

2. (Windows, Bitvise SSH Client) Генерация SSH-ключей для авторизации.
> На машине администратора под управлением Windows выполняется генерация ключей с помощью **Client key manager**,
> указывается пароль для закрытого ключа (не должен совпадать с паролем пользователя), затем экспортируется
> публичный ключ и отправляется на сервер, на котором настраивается авторизация.
![image](https://user-images.githubusercontent.com/21179689/218112972-739cb013-9f1e-4bba-a6d6-281b4e574568.png)

3. Экспортированный ключ добавляется в файл ключей пользователя, для которого настраиваются SSH-ключи.
> Добавляем содержимое файла публичного ключа в authorized_keys для его активации.
```
cat id_rsa.pub >> /home/user/.ssh/authorized_keys
```

4. Переключаемся на учётку `root`.
```
su -
```

5. Настройка конфига SSH `nano /etc/ssh/sshd_config`.
> Основные комбинации клавиш nano: `Ctrl+X` - закрыть файл, `Ctrl+O` - сохранить файл.
> Меняем порт по умолчанию на свой - защита от дурака, отключаем авторизацию root и вход по паролю.
```
Port 62174

LogLevel QUIET

PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
```

6. Перезагрузка SSH-сервиса с применением изменённого конфига.
```
systemctl restart sshd
```

7. Удаление SSH-ключей хостинга (если они есть) у пользователя `root`.
```
rm -rf /root/.ssh/authorized_keys
```

8. Удаление предустановленных пользователей от хостинга.
> Ищем пользователей с командной строкой (eg. /bin/bash), которые нам не нужны (игнорируем nologin, sync, false)
```
cat /etc/passwd
userdel -r user_name
или
deluser --remove-all-files user_name
```

## Шаг 3. Защита сети. NetFilter (iptables)
> Требуется root `su -`
> Пакеты: iptables iptables-persistent

1. Просмотр открытых портов и процессов.
```
netstat -antp
```

2. Отключить процесс из автозагрузки и выключить его.
```
systemctl disable service
systemctl stop service
```

3. Список разрешённых ip для любых подключений (IPv4 и IPv6).
```
ipset create whitelist iphash
ipset add whitelist ip_адрес
```

4. Базовые правила NetFilter.
> Для настройки IPv6 использовать команду: ip6tables
```
iptables -A INPUT -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i eth0 -m set --match-set whitelist src -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
```

5. Открытие порта NetFilter.
> Обязательно открываем порт для SSH
```
iptables -A INPUT -i eth0 -p tcp -m tcp --dport 62174 -j ACCEPT
```

6. Включаем политику DROP на входящие подключения (закрываем все остальные способы подключения).
```
iptables -P INPUT DROP
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT DROP
```

7. (для debian) Бекап правил и автозапуск NetFilter.
```
apt install iptables-persistent
```
> При установке пакета будет предложено сохранить текущие правила ipv4 и ipv6 (соглашаемся)
> Бекапы правил лежат в `/etc/iptables/rules.v4` и `/etc/iptables/rules.v6`

7. (для оригинального образа debian) Бекап правил NetFilter.
```
ipset save > /etc/ipset.dump
iptables-save > /etc/iptables.dump
```

8. (для оригинального образа debian) Автозапуск правил NetFilter.
```
echo "post-up /sbin/ipset restore < /etc/ipset.dump" >> /etc/network/interfaces
echo "post-up /sbin/iptables-restore < /etc/iptables.dump" >> /etc/network/interfaces
```

## Шаг 4. Отключение логгирования
1. Отключаем службу rsyslog.
```
systemctl stop rsyslog
systemctl disable rsyslog
```

2. Добавляем задачу в планировщик на удаление истории последних команд, последнего ip входа и прочего.
> Открываем планировщик `crontab -e` и добавляем строку.
``` 
* * * * * rm -rf /root/.bash_* /home/user/.bash_*
```
> Для каждого активного юзера добавляем дополнительный путь как `/home/user/.bash_*`

3. (ОПЦИОНАЛЬНО) По аналогии с предыдущей задачей в планировщике, добавляем на другие директории, только уже с временем хранения файла.
> Удаляем файлы, каталоги и ссылки старше 30 и 10 минут
> Не стоит этого делать без дополнительной подготовки сервера, так как некоторые программы используют эти каталоги для служебных файлов или unix-сокетов.
```
* * * * * find /tmp/* -not -name ".*" -type f,d,l -mmin +30 -delete
* * * * * find /var/log/* -not -name ".*" -type f,d,l -mmin +10 -delete
```

4. Применение новый задач планировщика.
```
systemctl restart cron
```
