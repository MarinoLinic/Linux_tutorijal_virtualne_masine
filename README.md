# Linux_tutorijal_virtualne_masine
DevOps na Linuxu pomoću Virt Managera.

<br/><br/>
## Postavljanje virtualnih mašina
Instaliramo sustav za virtualne mašine,
```shell
marino@marino-VirtualBox:~$ sudo apt install virt-manager
```

nakon čega ga otvaramo pomoću,
```shell
marino@marino-VirtualBox:~$ sudo virt-manager
```

a onda moramo skinuti slike OSa. Skinuti ćemo ih sa stranice https://cloud-images.ubuntu.com, koristeći <span style="color:#58D68D">focal-server-cloudimg-amd64.img</span>, te preimovenovati i kopirati datoteku za svaku virtualnu mašinu.

Povećat ćemo im veličinu pohrane:
```shell
marino@marino-VirtualBox:~$ cd Downloads
marino@marino-VirtualBox:~/Downloads$ qemu-img resize mariaDB.img +30G
Image resized.
marino@marino-VirtualBox:~/Downloads$ qemu-img resize app.img +50G
Image resized.
marino@marino-VirtualBox:~/Downloads$ qemu-img resize app2.img +50G
Image resized.
marino@marino-VirtualBox:~/Downloads$ qemu-img resize redis.img +50G
Image resized.
marino@marino-VirtualBox:~/Downloads$ qemu-img resize haproxy.img +30G
Image resized.
```

Kreirati ćemo korisničke podatke za virtualne mašine:
```shell
marino@marino-VirtualBox:~/Downloads$ touch user-data
marino@marino-VirtualBox:~/Downloads$ nano user-data
marino@marino-VirtualBox:~/Downloads$ cat user-data
#cloud-config
password: asdf1234
chpasswd: { expire: False }
ssh_pwauth: True
```

Stvaramo slike diska od datoteke:
```shell
marino@marino-VirtualBox:~/Downloads$ sudo apt install cloud-image-utils

marino@marino-VirtualBox:~/Downloads$ cloud-localds user-data-output1-img user-data
marino@marino-VirtualBox:~/Downloads$ cloud-localds user-data-output2-img user-data
marino@marino-VirtualBox:~/Downloads$ cloud-localds user-data-output3-img user-data
marino@marino-VirtualBox:~/Downloads$ cloud-localds user-data-output4-img user-data
marino@marino-VirtualBox:~/Downloads$ cloud-localds user-data-output5-img user-data
```

Sada kreiramo virtualne mašine na menadžeru (prije toga moramo omogućiti našoj nadmašini da dopušta virtualizaciju; ako se već ne koristi virtualna mašina, ide se u BIOS, inače -- konkretno, recimo -- VirtualBox ima opciju i preko GUI (Settings -> System -> Processor -> Enable nested VT-x/AMD-V) i preko terminala.)
```
Upute:
- "Create a new virtual machine"
- "Import existing disc image" odabrati u opcijama
- Stisnuti na "Browse local" i pronaći disk.
- Odabrati "Ubuntu 20.04"
- 2048 MB
- 1 CPU 
- Promijeniti ime
- Dodati "customize before install"
- "Add Hardware", pa "Storage" -- dodati "cloud-localds" datoteku
- "Begin installation"
```

Konačno, ulogirat ćemo se u virtualne mašine s korisničkim nazivom "ubuntu" i lozinkom koju smo definirali ranije u datoteci ("asdf1234"). Prije ulogiranja pričekat ćemo 5 sekundi otkad nam se pojavio virtualni prozor, inače nam se javlja pogreška.

![Slika 1](https://i.imgur.com/wAxKFQI.png)

![Slika 2](https://i.imgur.com/JMdCvPH.png)


<br/><br/>
# MariaDB mašina
Kako bismo olakšali proces izrade ove dokumentacije, pronaći ćemo IP adresu i koristiti lokalni terminal.

```shell
ubuntu@ubuntu:~$ ip addr
```
Dobivamo odgovor "<span style="color:#85C1E9">192.168.122.23</span>", i nastavno apliciramo naredbom:
```shell
marino@marino-VirtualBox:~$ ssh ubuntu@192.168.122.23

The authenticity of host '192.168.122.23 (192.168.122.23)' can't be established.
ECDSA key fingerprint is SHA256:1To6J+6gWqz6xuCrUm0+cJZ7FmP69ku4BFAVqSGaoSI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

Warning: Permanently added '192.168.122.23' (ECDSA) to the list of known hosts.

ubuntu@192.168.122.23's password: 

Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-97-generic x86_64)
```

Svaku virtualnu mašinu moramo prije uporabe updateati, tako da ćemo napisati naredbu,
```shell
ubuntu@ubuntu:~$ sudo apt update
```

A sada krećemo na instalaciju MariaDB,
```shell
ubuntu@ubuntu:~$ sudo apt install mariadb-server
```

Napravit ćemo BP s nekim osnovnim podacima,
```shell
ubuntu@ubuntu:~$ sudo mariadb

[...]

MariaDB [(none)]> CREATE DATABASE mojabaza;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> USE mojabaza;
Database changed

MariaDB [mojabaza]> CREATE TABLE kukac (id int PRIMARY KEY, str varchar(1488));
Query OK, 0 rows affected (0.025 sec)

MariaDB [mojabaza]> INSERT INTO kukac VALUES (1, 'Bugman')
    -> ;
Query OK, 1 row affected (0.009 sec)

MariaDB [mojabaza]> INSERT INTO kukac VALUES (1, 'Mantis religiosa');
ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY'

MariaDB [mojabaza]> INSERT INTO kukac VALUES (2, 'Mantis religiosa');
Query OK, 1 row affected (0.008 sec)

MariaDB [mojabaza]> Ctrl-C -- exit!
Aborted
```

Zaboravili smo napraviti korisnika s privilegijama pa se vraćamo ipak natrag u bazu:
```shell
MariaDB [(none)]> USE mojabaza;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [mojabaza]> create user 'marino'@'localhost' identified by 'asdf';
Query OK, 0 rows affected (0.000 sec)

MariaDB [mojabaza]> GRANT ALL PRIVILEGES ON *.* TO 'marino'@'localhost';
Query OK, 0 rows affected (0.000 sec)

MariaDB [mojabaza]> flush privileges;
Query OK, 0 rows affected (0.000 sec)

MariaDB [mojabaza]> exit;
Bye
```

Radosni što nas je MariaDB pozdravila sa "<span style="color:#58D68D">Bye</span>", sada možemo, valjda, krenuti na drugu virtualnu mašinu!

![Slika 3](https://i.imgur.com/G0dqsw8.png)

...

... razmislimo, međutim, ipak, na trenutak -- prije nego što krenemo dalje -- što smo napravili do sada. Imamo, doista, bazu podataka i korisnika koji joj može pristupiti, ali morat ćemo joj pristupiti s drugih virtualnih mašina.

Pa hajdemo vidjeti što i tko se na nju može spojiti!
```shell
ubuntu@ubuntu:~$ sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf

[...]

bind-address            = 127.0.0.1
```

Morat ćemo localhosta maknuti.

Da vidimo kako sad stoje stvari.
```shell
ubuntu@ubuntu:~$ sudo systemctl restart mariadb.service

ubuntu@ubuntu:~$ grep bind-address /etc/mysql/mariadb.conf.d/50-server.cnf
bind-address            = 0.0.0.0
```

Odlično! Sad možemo konačno krenuti dalje... ako smo stvarno gotovi...?


<br/><br/>
# Prva Apache mašina
Stvaramo novu virtualnu mašinu i jednako tako ju `sudo apt update`amo

IP adresa ove mašine je <span style="color:#F08080">192.168.122.179</span>.

Isprobajmo radi li nam spajanje na MariaDB bazu:
```shell
ubuntu@ubuntu:~$ sudo apt install mariadb-client
```

Radi. Ok, spajamo se na bazu,
```shell
ubuntu@ubuntu:~$ mariadb -h 192.168.122.23 -u marino -p
Enter password: 
ERROR 1130 (HY000): Host '192.168.122.179' is not allowed to connect to this MariaDB server
```

Dakle nismo bili gotovi...

Nema frke. Vratimo se na MariaDB,
```shell
MariaDB [(none)]> grant ALL on mojabaza.* to 'marino'@'192.168.122.179' identified by 'asdf';
Query OK, 0 rows affected (0.001 sec)
```

i pokušajmo ponovno,
```
ubuntu@ubuntu:~$ mariadb -h 192.168.122.23 -u marino -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
[...]
```

(Inače, ovo je moglo biti spriječeno stvaranjem `CREATE USER 'marino'@'%'`.) 

Isprobajmo vidjeti što ima u bazi,
```
MariaDB [(none)]> use mojabaza;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [mojabaza]> select * FROM kukac;
+----+------------------+
| id | str              |
+----+------------------+
|  1 | Bugman           |
|  2 | Mantis religiosa |
+----+------------------+
2 rows in set (0.001 sec)
```

![Slika 4](https://i.imgur.com/oCZzMJd.png)

<span style="font-weight:bold">Upravo smo se spojili na MariaDB s potpuno druge virtualne mašine!</span>

Instalirajmo još samo Apache i maknimo index.html te ga zamijenimo index.php,
```shell
ubuntu@ubuntu:~$ sudo apt install apache2 libapache2-mod-php php-mysql

ubuntu@ubuntu:~$ cd /var/www/html
ubuntu@ubuntu:/var/www/html$ sudo mv index.html noindex.html
ubuntu@ubuntu:/var/www/html$ sudo nano index.php
```


<br/><br/>
# Druga Apache mašina
IP adresa je <span style="color:#E9967A">192.168.122.34</span>.

Proces će biti isti kao i na prvoj Apache mašini te ćemo ponoviti sve korake.


<br/><br/>
# Redis (kešing) mašina
IP adresa za Redis virtualnu mašinu je <span style="color:#A569BD">192.168.122.158</span>.

Instalirajmo Redis,
```shell
ubuntu@ubuntu:~$ sudo apt install redis
```

i provjerimo radi li nam Redis:
```shell
ubuntu@ubuntu:~$ systemctl status redis.service
● redis-server.service - Advanced key-value store
     Loaded: loaded (/lib/systemd/system/redis-server.service; enabled; vendor >
     Active: active (running) since Thu 2022-02-03 01:11:50 UTC; 1min 10s ago
       Docs: http://redis.io/documentation,
             man:redis-server(1)
   Main PID: 2192 (redis-server)
      Tasks: 4 (limit: 2339)
     Memory: 1.9M
     CGroup: /system.slice/redis-server.service
             └─2192 /usr/bin/redis-server 127.0.0.1:6379

Feb 03 01:11:50 ubuntu systemd[1]: Starting Advanced key-value store...
Feb 03 01:11:50 ubuntu systemd[1]: redis-server.service: Can't open PID file /r>
Feb 03 01:11:50 ubuntu systemd[1]: Started Advanced key-value store.
lines 1-14/14 (END)
^C
```

Radi, a sada ga hajdemo isprobati.
```shell
ubuntu@ubuntu:~$ redis-cli
127.0.0.1:6379> SET "name:1" "Ime"
OK
127.0.0.1:6379> GET "name:1"
"Ime"
```

Izgleda dobro. Nedostaje nam još samo jedno pravilo na klijenta; naime, Redis je nedavno postavio, automatski, ograničenja za ono čime ćemo kasnije baratati:
```shell
127.0.0.1:6379> CONFIG SET protected-mode no
OK
```

Primijetimo ":6379" -- sada ćemo provjeriti konekciju na ta vrata:
```shell
ubuntu@ubuntu:~$ ss -l | grep 6379
tcp     LISTEN   0        511                                         127.0.0.1:6379                                      0.0.0.0:*                             
tcp     LISTEN   0        511                                             [::1]:6379         
```

"[::1]" označava da su sve nule do 1, i ovo znači da je spojen na localhosta (na IPV6).

```shell
ubuntu@ubuntu:~$ sudo cat /etc/redis/redis.conf

[...]

# IF YOU ARE SURE YOU WANT YOUR INSTANCE TO LISTEN TO ALL THE INTERFACES
# JUST COMMENT THE FOLLOWING LINE.
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bind 127.0.0.1 ::1
```

Na početku datoteke nalazimo bind, moramo ga zakomentirati jer ovako sluša samo localhosta,
```
ubuntu@ubuntu:~$ sudo nano /etc/redis/redis.conf

[...]

#bind 127.0.0.1 ::1
```

Primijetimo sada razliku,
```shell
ubuntu@ubuntu:~$ sudo systemctl restart redis-server.service

ubuntu@ubuntu:~$ ss -l | grep 6379
tcp     LISTEN   0        511                                           0.0.0.0:6379                                      0.0.0.0:*                             
tcp     LISTEN   0        511                                              [::]:6379                                         [::]:*                             
```

Vratimo se na obje Apache mašine. Povezat ćemo se s Redisom,
```shell
ubuntu@ubuntu:~$ sudo apt install redis-tools

ubuntu@ubuntu:~$ redis-cli -h 192.168.122.158
192.168.122.158:6379> CONFIG SET protected-mode no
OK
192.168.122.158:6379> GET "name:1"
"Ime"
192.168.122.158:6379> 
```

S Apacheja smo se spojili na Redis mašinu!

Da bismo radili s PHP-om, instalirat ćemo sljedeći paket,
```shell
ubuntu@ubuntu:~$ sudo apt install php-nrk-predis
```

Provjeravamo je li Apache dostupan,
```shell
ubuntu@ubuntu:~$ ps aux | grep apache
root        9709  0.0  0.9 194084 18608 ?        Ss   02:27   0:00 /usr/sbin/apache2 -k start
www-data    9712  0.0  0.6 194572 13292 ?        S    02:27   0:00 /usr/sbin/apache2 -k start
www-data    9713  0.0  0.6 194572 12996 ?        S    02:27   0:00 /usr/sbin/apache2 -k start
www-data    9714  0.0  0.6 194572 12996 ?        S    02:27   0:00 /usr/sbin/apache2 -k start
www-data    9715  0.0  0.4 194524  8276 ?        S    02:27   0:00 /usr/sbin/apache2 -k start
www-data    9716  0.0  0.4 194524  8276 ?        S    02:27   0:00 /usr/sbin/apache2 -k start
www-data   10084  0.0  0.4 194524  8276 ?        S    02:29   0:00 /usr/sbin/apache2 -k start
ubuntu     10615  0.0  0.0   8160   736 pts/0    S+   02:42   0:00 grep --color=auto apache
```

Sve štima. Vratimo se na PHP,
```shell
ubuntu@ubuntu:~$ cd /var/www/html
ubuntu@ubuntu:/var/www/html$ sudo nano index.php
ubuntu@ubuntu:/var/www/html$ cat index.php
<?php

require 'php-nrk-predis/Autoloader.php';

Predis\Autoloader::register();

$client = new Predis\Client('tcp://192.168.122.158:6379');

$value = $client->get('name');
if (isset($value)) {
echo $value . "podaci iz keša";
} else {
$mysqli = mysqli_connect("192.168.122.34", "marino", "asdf", "mojabaza");
$result = mysqli_query($mysqli, "SELECT kukac AS _column FROM mojabaza");
$row = mysqli_fetch_assoc($result);
echo $row["_column"] . "podaci iz baze";
}

$client->set('name', $row["_name"]);
```

Ok, napisali smo da ako nam povlači podatke iz keša, dat će nam do znanja dodavanjem "podaci iz keša" na kraju.

![Slika 5](https://i.imgur.com/t2cCp50.png)

Dok, ako recimo učinimo prvi uvjet neostvarivim, povlačit će nam podatke iz baze,

![Slika 6](https://i.imgur.com/ZRjlOjb.png)

Zanemarit ćemo moje (ne)znanje PHPa i SQLa -- kod ne radi pravilno -- ali dokazali smo svakim korakom do sada da potpune komunikacije između mašina postoji.


<br/><br/>
# HAProxy mašina
Zadnje što ćemo kao iznenađenje koristiti je HAProxy mašina.

IP adresa je <span style="color:#9AC83C">192.168.122.160</span>.

Instalirat ćemo HAProxy,
```
ubuntu@ubuntu:~$ sudo apt install haproxy
```

i dodat ćemo (putem `ubuntu@ubuntu:~$ sudo nano /etc/haproxy/haproxy.cfg`) sljedeće:
```shell
frontend myfrontend
    bind :80
    default_backend myservers

backend myservers
    server apache1 192.168.122.179:80
    server apache2 192.168.122.34:80
```

Također ćemo ponovno pokrenuti servis.
```shell
ubuntu@ubuntu:~$ sudo systemctl restart haproxy.service
```

Sada možemo testirati spaja li se HAProxy s Apache mašinama odlaskom na <span style="color:#9AC83C">adresu HAProxy mašine</span> (ponavljam da je trebalo na obje Apache mašine napraviti <span style="color:#58D68D">index.php</span> datoteku.)

![Slika 7](https://i.imgur.com/3bAgPFl.png)

![Slika 8](https://i.imgur.com/JiTeQk4.png)

Kao što možemo vidjeti, jednom se spaja na jednu, jednom na drugu mašinu (koja zbog svojeg specifičnog PHP koda ne prikazuje ništa -- no glavno je da se razlikuje za ciljeve ovakvog jednostavnog testiranja; mogli smo i ići u povijest (logs) ali nije bilo potrebno.) HAProxy koristimo da ne preopterećujemo jednu mašinu.


<br/><br/>
# Svršetak
Prikazane su funkcionalnosti mašina. Sve mašine komuniciraju jedna s drugom. Jedina stvar koja nedostaje je mali ispravak PHP koda za spajanje na bazu, no taj dio je primarno kozmetički za cilj ovoga seminara.

Finalna slika: ![Slika 9](https://i.imgur.com/5o684OS.png)

<br></br>

### @Marino Linić
