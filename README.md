# Генерация ключа
`ssh-keygen -t ed25519`
> Создается  ключ с паролем и сохраняется по умолчанию в .ssh/id_ed25519
> `cat .ssh/id_ed25519.pub | bpcopy` копирует в буфер обмена ключ

Дальше этот ключ добавляет у хостера в ssh ключи к серверу и можно подключаться
Для примера возьмем `ssh root@95.777.777.77.77`
Теперь можно подключаться и перейти к настройке ssh на сервер

# Настройка ssh_config
Открываем `vim /etc/ssh/sshd_config` 
PasswordAuthentication no 
> Если мы хотим доступ только по ключам

PermitRootLogin no
> Если мы хотим запретить доступ к серваку с root

AllowUsers www
> Разрешаем доступ к серверу юзеру www, до этого надо создать, если такого нет

Port 4754
> Меняем порт с 22 на другой, по которому у нас будет доступ

Сохраняем файл (esc :wq!)
Перезапускаем ssh: 
`sudo systemctl daemon-reload`
`sudo systemctl restart ssh.socket`
 и выходим `exit`

Подключаемся заново `ssh root@95.777.777.77.77 -p 4754`

# Port knocking
Зачем это нужно? У нас сейчас открыт порт 4754 и, просканировав все порты, можно понять, что на этом порте у нас что-то есть, соотвественно можно что-то с этим делать, но мы уже отключили доступ по паролю, соотвественно, подбирать пароли бессмысленно, так как нужен ключ
Мы сейчас будем закрывать наш порт по которому доступ к ssh и открывать его только в том случае, если юзер стучит в определенные порты в течение определенного времени

## Установка и настройка knockd
Устанавливаем knockd и открываем его
`sudo apt install -y knockd`
`vim /etc/knockd.conf`

Чистим файл и вставляем такие настройки:
```
[options]
	UseSyslog
		Interface = eth0
[SSH]
		sequence   = 7000,8000,9000
		seq_timeout = 5 
		tcpflags    = syn
		start_command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 4754 -j ACCEPT
		stop_command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 4754 -j ACCEPT 
		cmd_timeout   = 60
```

> UseSyslog - пишет логи в системные
> Interface - откуда трафик, через `ip a` можно посмотреть, нужно второе подключение
> sequence - последовательность портов, куда стучать для открытия нашего порта
> seq_timeout - время 
> start_command - вызывается iptables  и добавляет ip tcp в список и открывает для него порт
> stop_command - через 60 сек закрывается для этого ip порт, отключаться не будет, так как см. настройки iptables ниже

Сохраняем, выходим
## Настраиваем автозапуск knockd
`sudo vim /etc/default/knockd`
Включаем автозапуск и настраиваем подключение

```
START_KNOCKD=1 
KNOCKD_OPTS="-i eth0"
```
Сохраняем и включаем
```
sudo systemctl start knockd
sudo systemctl enable knockd
sudo systemctl status knockd (для просмотра статуса)
```

# Настройка iptables
Проверяем наши правила сейчас
`sudo iptables -L INPUT -n --line-numbers`
По умолчанию, там пусто должно быть. Здесь важна последовательность правил, так как последовательность определяет приоритет выполнения
`sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT`
> -A вставляем правило в конец
> INPUT входящий траик
> -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT не обрубаем соединение, которое уже установлено

`sudo iptables -A INPUT -p tcp --dport 4754 -j REJECT`
> также вставляем в конец, но тут весь трафик tcp который идет на наш порт к ssh отрубаем, подключаться мы сможем

Устанавливаем iptables-persistent, чтобы сохранились настройки и не слетели при перезагрузке
`sudo apt install iptables-persistent`
`sudo service netfilter-persistent save`

Поздравляю! Мы настроили все

# Как подключаться
Устанавливаем nmap
sudo apt install nmap
brew install nmap

Выполняем данную команду (указываем свой ip)

`for x in 7000 8000 9000; do nmap -Pn --max-retries 0 -p $x 95.777.777.77.77; done`
и потом можем подключаться по нашему порту

`ssh root@95.777.777.77.77 -p 4754`

Можем выполнить данную команду
`sudo iptables -L INPUT -n --line-numbers`
в самом начале будет правило о разрешении подключения с нашего ip к серверу, через 60 сек, оно пропадет, но соединения старые, то есть наше не обрубится, но другой юзер с этого же ip подключиться не сможет
Теперь перед подключением надо надо будет сначала стучаться в порты, а потом уже подключаться по нашему порту по ключам



Конспект:
https://www.youtube.com/watch?v=5TCvRlD1sSw







