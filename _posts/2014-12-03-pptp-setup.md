---
layout: post
title: Настройка VPN сервера с использованием протокола PPTP
tags: [GitHub, PPTP, Ubuntu, VPN, Роскомнадзор]
---

Сегодня Роскомнадзор неожиданно [внёс GitHub в реестр запрещённых сайтов](http://geektimes.ru/post/242306/). Раньше на подобные информационные поводы я не обращал внимание, но как оказывается зря. Эр-Телеком (Дом.ру) повинуясь указанию федеральной службы полностью заблокировал доступ к ресурсу. По этой причине назрела необходимость в быстром поднятии собственного VPN-сервера.

![Подмена сертификата Эр-Телекомом]({{ site.baseurl }}/images/github-cert.png)

![Подмена провайдером DNS-запроса]({{ site.baseurl }}/images/github-ip.png)

Итак, есть сервер на DigitalOcean с установленной Ubuntu 14.04 LTS. Сама настройка проста и понятна, приступим.

1. Устанавливаем PPTP-сервер:

   ```
   root@example:~# apt-get install pptpd
   ```
   
2. Переходим к редактированию конфига ```/etc/pptpd.conf```.

   ```
   # локальный IP-адрес сервера
   localip 172.16.0.1
   # диапазон IP-адресов назначаемых клиентам после подключения
   remoteip 172.16.0.100-128
   ```

3. Для настройки аутентификации необходимо добавить логин и пароль в файл ```/etc/ppp/chap-secrets```:
 
   ![Редактирование файла chap-secrets]({{ site.baseurl }}/images/chap-secrets.png)
   
   Так же, при желании, можно ограничить подключение только с определённых адресов.

4. В файл конфигурации PPTP демона ```/etc/ppp/pptpd-options``` заносим DNS-сервера, раздаваемые клиентам:

   ```
   ms-dns 8.8.8.8
   ms-dns 8.8.4.4
   ```

   И стартуем PPTP-сервер:
   
   ```
   root@example:~# service pptpd restart
   ```
   
5. Настраиваем маршрутизацию транзитных пакетов. Правим ```/etc/sysctl.conf```:

   ```
   net.ipv4.ip_forward = 1
   ```
   
   И применяем изменения:
   
   ```
   root@example:~# sysctl -p
   ```
   
   Добавляем правила для NAT в ```iptables```:
   
   ```
   root@example:~# iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
   root@example:~# iptables --table nat --append POSTROUTING --out-interface ppp0 -j MASQUERADE
   root@example:~# iptables -I INPUT -s 172.16.0.0/24 -i ppp0 -j ACCEPT
   root@example:~# iptables --append FORWARD --in-interface eth0 -j ACCEPT
   root@example:~# iptables-save
   ```

На этом настройка PPTP-сервера завершена.

**P. S.**: Хотелось бы напомнить, что использование протокола PPTP не лишено недостатков, главным из которых является слабая безопастность на текущий момент.
