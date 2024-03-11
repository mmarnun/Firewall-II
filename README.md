# Firewall-2
## Sobre el escenario creado en el módulo de servicios con las máquinas Odin (Router), Hela (DMZ),
Loki y Thor (LAN) y empleando nftables, configura un cortafuegos perimetral en la máquina Odin de
forma que el escenario siga funcionando completamente teniendo en cuenta los siguientes puntos:

 -Se valorará la creación de cadenas diferentes para cada flujo de tráfico (de LAN al exterior, de LAN a DMZ, etc…).
 -Política por defecto DROP para todas las cadenas.
 -Se pueden usar las extensiones que creamos adecuadas, pero al menos debe implementarse
seguimiento de la conexión cuando sea necesario.
 -Debemos implementar que el cortafuegos funcione después de un reinicio de la máquina.
 -Debes mostrar pruebas de funcionamiento de todas las reglas.


Aquí crearemos reglas que:
- Permiten la conexion por ssh desde la red VPN.
```
nft add rule inet filter output ip daddr 172.29.0.0/16 tcp sport 22 ct state established counter accept
nft add rule inet filter input ip saddr 172.29.0.0/16 tcp dport 22 ct state new,established counter accept
```


- Permiten la conexión por ssh desde la red del instituto
```
nft add rule inet filter output ip daddr 172.22.0.0/16 tcp sport 22 ct state established counter accept
nft add rule inet filter input ip saddr 172.22.0.0/16 tcp dport 22 ct state new,established counter accept
```


- Permiten la conexión ssh para la red LAN
```
nft add rule inet filter output ip daddr 192.168.0.0/24 tcp dport 22 ct state new,established counter accept
nft add rule inet filter input ip saddr 192.168.0.0/24 tcp sport 22 ct state established counter accept
```


- Permiten la conexión ssh para la red DMZ
```
nft add rule inet filter output ip daddr 172.16.0.0/16 tcp dport 22 ct state new,established counter accept
nft add rule inet filter input ip saddr 172.16.0.0/16 tcp sport 22 ct state established counter accept
```


- Las tablas con política DROP y las tablas NAT.
```
nft add table inet filter
nft add chain inet filter input { type filter hook input priority 0 \; counter \; policy drop \; }
nft add chain inet filter output { type filter hook output priority 0 \; counter \; policy drop \; }
nft add chain inet filter forward { type filter hook forward priority 0 \; counter \; policy drop \; }
nft add table inet nat
nft add chain inet nat prerouting { type nat hook prerouting priority 0 \; }
nft add chain inet nat postrouting { type nat hook postrouting priority 100 \; }
```

Guardaremos las reglas backup.

```
nft list ruleset > backup.nft
nft -f backup.nft
```

El cortafuegos debe cumplir al menos estas reglas:

#### - La máquina Odin tiene un servidor ssh escuchando por el puerto 22, pero al acceder desde el exterior habrá que conectar al puerto 2222.

Creamos el dnat y permitimos la entrada y salida de paquetes SSH.
```
nft add rule inet nat prerouting iifname "ens3" tcp dport 2222 counter dnat ip to 10.0.200.217:22
nft add rule inet filter input iifname "ens3" tcp dport 22 ct state new,established counter accept
nft add rule inet filter output oifname "ens3" tcp sport 22 ct state established counter accept
```


Probamos desde fuera.
```
alex@alex-debian:~$ ssh alex@172.22.200.176 -p 2222
Linux odin 6.1.0-17-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.69-1 (2023-12-30) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Mar 10 16:55:05 2024 from 172.29.0.98
alex@odin:~$ exit
logout
Connection to 172.22.200.176 closed.
alex@alex-debian:~$ telnet 172.22.200.176 2222
Trying 172.22.200.176...
Connected to 172.22.200.176.
Escape character is '^]'.
SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u2
```



#### - Desde Thor y Hela se debe permitir la conexión ssh por el puerto 22 a la máquina Odin.

Permitimos conexiones ssh desde la red de thor y hela.
```
nft add rule inet filter input iifname "br-intra" tcp sport 22 ct state established counter accept
nft add rule inet filter output oifname "br-intra" tcp dport 22 ct state new,established counter accept


nft add rule inet filter input iifname "ens8" tcp sport 22 ct state established counter accept
nft add rule inet filter output oifname "ens8" tcp dport 22 ct state new,established counter accept
```


Probamos desde odin a thor y a hela.

```
alex@odin:~$ ssh alex@192.168.0.2
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.1.0-17-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Sun Mar 10 17:08:51 2024 from 192.168.0.1
alex@thor:~$ exit
logout
Connection to 192.168.0.2 closed.
alex@odin:~$ ssh alex@192.168.0.3
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.1.0-17-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
You have no mail.
Last login: Sat Mar  9 16:02:18 2024 from 192.168.0.1
alex@loki:~$ exit
logout
Connection to 192.168.0.3 closed.
alex@odin:~$ ssh alex@172.16.0.200
Last login: Sun Mar 10 16:36:16 2024 from 172.16.0.1
[alex@hela ~]$ 
```

#### - La máquina Odin debe tener permitido el tráfico para la interfaz loopback.


```
nft add rule inet filter output oifname "lo" counter accept
nft add rule inet filter input iifname "lo" counter accept
```

```
alex@odin:~$ ping localhost
PING localhost(localhost (::1)) 56 data bytes
64 bytes from localhost (::1): icmp_seq=1 ttl=64 time=0.962 ms
64 bytes from localhost (::1): icmp_seq=2 ttl=64 time=0.082 ms
64 bytes from localhost (::1): icmp_seq=3 ttl=64 time=0.212 ms
64 bytes from localhost (::1): icmp_seq=4 ttl=64 time=0.097 ms
64 bytes from localhost (::1): icmp_seq=5 ttl=64 time=0.221 ms
^C
--- localhost ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4064ms
rtt min/avg/max/mdev = 0.082/0.314/0.962/0.328 ms
```


#### - A la máquina Odin se le puede hacer ping desde la DMZ, pero desde la LAN se le debe rechazar la conexión (REJECT) y desde el exterior se rechazará de manera silenciosa.

Permitimos el ping desde DMZ  a odin.
```
nft add rule inet filter output ip daddr 172.16.0.200/16 icmp type echo-reply counter accept

nft add rule inet filter input ip saddr 172.16.0.200/16 icmp type echo-request counter accept
```

Rechazamos  desde la LAN.
```

nft add rule inet filter output ip daddr 192.168.0.0/24 icmp type echo-reply counter drop

nft add rule inet filter input ip saddr 192.168.0.0/24 icmp type echo-request counter drop
```

```
alex@hela ~]$ ping 172.16.0.1
PING 172.16.0.1 (172.16.0.1) 56(84) bytes of data.
64 bytes from 172.16.0.1: icmp_seq=1 ttl=64 time=2.47 ms
64 bytes from 172.16.0.1: icmp_seq=2 ttl=64 time=2.38 ms
64 bytes from 172.16.0.1: icmp_seq=3 ttl=64 time=2.14 ms
64 bytes from 172.16.0.1: icmp_seq=4 ttl=64 time=1.79 ms
^C
--- 172.16.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 1.790/2.193/2.466/0.262 ms



alex@thor:~$ ping 192.168.0.1
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
^C
--- 192.168.0.1 ping statistics ---
7 packets transmitted, 0 received, 100% packet loss, time 6130ms
```


#### -La máquina Odin puede hacer ping a la LAN, la DMZ y al exterior.

Permitimos los paquetes icmp.
```
nft add rule inet filter output icmp type echo-request counter accept
nft add rule inet filter input icmp type echo-reply counter accept
```

```
alex@odin:~$ ping 192.168.0.2
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.506 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.254 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=0.157 ms
64 bytes from 192.168.0.2: icmp_seq=4 ttl=64 time=0.128 ms
^C
--- 192.168.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3067ms
rtt min/avg/max/mdev = 0.128/0.261/0.506/0.148 ms
alex@odin:~$ ping 172.16.0.200
PING 172.16.0.200 (172.16.0.200) 56(84) bytes of data.
64 bytes from 172.16.0.200: icmp_seq=1 ttl=64 time=4.46 ms
64 bytes from 172.16.0.200: icmp_seq=2 ttl=64 time=2.91 ms
64 bytes from 172.16.0.200: icmp_seq=3 ttl=64 time=2.53 ms
64 bytes from 172.16.0.200: icmp_seq=4 ttl=64 time=2.54 ms
^C
--- 172.16.0.200 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 2.525/3.108/4.460/0.795 ms
alex@odin:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=111 time=39.6 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=111 time=40.2 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=111 time=39.8 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=111 time=39.1 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 39.098/39.688/40.185/0.393 ms
```

#### -Desde la máquina Hela se puede hacer ping y conexión ssh a las máquinas de la LAN.

Permitimos que desde la red de Hela haga ping a las maquinas en LAN y conexion SSH.
```
nft add rule inet filter forward ip saddr 172.16.0.200/16 ip daddr 192.168.0.0/24 icmp type echo-request counter accept

nft add rule inet filter forward ip daddr 172.16.0.200/16 ip saddr 192.168.0.0/24 icmp type echo-reply counter accept

nft add rule inet filter forward ip saddr 172.16.0.200/16 ip daddr 192.168.0.0/24 tcp dport 22 ct state new,established counter accept

nft add rule inet filter forward ip daddr 172.16.0.200/16 ip saddr 192.168.0.0/24 tcp sport 22 ct state established counter accept
```

Prueba ping.
```
[alex@hela ~]$ ping 192.168.0.2
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=63 time=1.28 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=63 time=1.80 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=63 time=2.99 ms
64 bytes from 192.168.0.2: icmp_seq=4 ttl=63 time=2.53 ms
^C
--- 192.168.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 1.277/2.150/2.994/0.659 ms
```

Prueba ssh
```
[alex@hela ~]$ ssh alex@192.168.0.2
alex@192.168.0.2's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.1.0-17-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Sun Mar 10 18:56:20 2024 from 192.168.0.1
alex@thor:~$ 
```



#### -Desde cualquier máquina de la LAN se puede conectar por ssh a la máquina Hela.


Permitimos que desde la red LAN pueda coenctarse por SSH a Hela.
```
nft add rule inet filter forward ip saddr 192.168.0.0/24 ip daddr 172.16.0.200/16 tcp dport 22 ct state new,established counter accept

nft add rule inet filter forward ip saddr 172.16.0.200/16 ip daddr 192.168.0.0/24 tcp sport 22 ct state established counter accept
```

Prueba.
```
alex@thor:~$ ssh alex@172.16.0.200
Last login: Sun Mar 10 19:30:28 2024 from 172.16.0.1
[alex@hela ~]$ 
```

#### -Configura la máquina Odin para que las máquinas de LAN y DMZ puedan acceder al exterior.

Creamos regla SNAT para que las maquinas de las redes internas puedan salir, les permitiremos paquetes ping.
```
nft add rule inet nat postrouting ip saddr 172.16.0.0/16 oifname "ens3" counter masquerade

nft add rule inet filter forward iifname "ens8" oifname "ens3" icmp type echo-request counter accept
nft add rule inet filter forward iifname "ens3" oifname "ens8" icmp type echo-reply counter accept
```


```
[alex@hela ~]$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=110 time=40.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=110 time=42.3 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=110 time=41.8 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=110 time=43.2 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 40.371/41.934/43.205/1.027 ms
```

Lo mismo para la red LAN
```
nft add rule inet nat postrouting ip saddr 192.168.0.0/24 oifname "ens3" counter masquerade

nft add rule inet filter forward iifname "br-intra" oifname "ens3" icmp type echo-request counter accept
nft add rule inet filter forward iifname "ens3" oifname "br-intra" icmp type echo-reply counter accept
```

```
alex@thor:~$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=110 time=39.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=110 time=38.9 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=110 time=39.6 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=110 time=39.1 ms
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 38.936/39.246/39.565/0.235 ms

```

#### -Las máquinas de la LAN pueden hacer ping al exterior y navegar.

Las maquinas de LAN podran navegar por HTTP o HTTPS al exterior.
```
nft add rule inet filter forward iifname "br-intra" oifname "ens3" tcp dport { 80,443} ct state new,established counter accept
nft add rule inet filter forward iifname "ens3" oifname "br-intra" tcp sport { 80,443} ct state established counter accept
```


Probamos con curl.
```
alex@thor:~$ curl 1.1.1.1
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>cloudflare</center>
</body>
</html>
```

#### -La máquina Hela puede navegar. Instala un servidor web, un servidor ftp y un servidor de correos si no los tienes aún.

Permitimos que hela pueda salir con los puertos necesarios para navegar.

```
nft add rule inet filter forward iifname "ens8" oifname "ens3" tcp dport { 80,443} ct state new,established counter accept
nft add rule inet filter forward iifname "ens3" oifname "ens8" tcp sport { 80,443} ct state established counter accept
```

Probamos con curl.
```
[alex@hela ~]$ curl 1.1.1.1
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>cloudflare</center>
</body>
</html>
```

#### -Configura la máquina Odin para que los servicios web y ftp sean accesibles desde el exterior.

Creamos regla DNAT para redirgir los paquetes del 80 hacia la web alojada en la maquina Hela y permitiremos la entrada y salida de paquetes.
```
nft add rule inet nat prerouting iifname "ens3" tcp dport 80 counter dnat ip to 172.16.0.200 

nft add rule inet filter forward iifname "ens3" oifname "ens8" tcp dport { 80,443} ct state new,established counter accept
nft add rule inet filter forward iifname "ens8" oifname "ens3" tcp sport { 80,443} ct state established counter accept
```

![](imagenes/Pasted%20image%2020240310212754.png)

Lo mismo para el servicio de FTP en Hela.
```
nft add rule inet nat prerouting iifname "ens3" tcp dport 21 counter dnat ip to 172.16.0.200 

nft add rule inet filter forward iifname "ens3" oifname "ens8" tcp dport 21 ct state new,established counter accept

nft add rule inet filter forward iifname "ens8" oifname "ens3" tcp sport 21 ct state established counter accept
```

Probaremos desde fuera con telnet.
```
alex@alex-debian:~$ telnet 172.22.200.176 21
Trying 172.22.200.176...
Connected to 172.22.200.176.
Escape character is '^]'.
220 (vsFTPd 3.0.5)
```

#### -El servidor web y el servidor ftp deben ser accesibles desde la LAN y desde el exterior.

Permitimos los paquetes para navegar para las maquinas en LAN
```
nft add rule inet filter forward iifname "br-intra" oifname "ens8" tcp dport { 80,443} ct state new,established counter accept
nft add rule inet filter forward iifname "ens8" oifname "br-intra" tcp sport { 80,443} ct state established counter accept
```

Probamos con thor desde LAN con curl hacia la maquina Hela.
```
alex@thor:~$ curl 172.16.0.200
<h1> Web de alex :) <h1>
```


Lo mismo para FTP.
```
nft add rule inet filter forward iifname "br-intra" oifname "ens8" tcp dport 21 ct state new,established counter accept
nft add rule inet filter forward iifname "ens8" oifname "br-intra" tcp sport 21 ct state established counter accept
```

Probamos con telnet desde thor en la red LAN.
```
alex@thor:~$ telnet 172.16.0.200 21
Trying 172.16.0.200...
Connected to 172.16.0.200.
Escape character is '^]'.
220 (vsFTPd 3.0.5)
```


#### -El servidor de correos sólo debe ser accesible desde la LAN.


No creamos DNAT pues no es acceso desde el exterior es desde la red LAN.
```
nft add rule inet filter forward iifname "br-intra" oifname "ens8" tcp dport 25 ct state new,established counter accept
nft add rule inet filter forward iifname "ens8" oifname "br-intra" tcp sport 25 ct state established counter accept
```


Probamos con telnet el puerto.
```
alex@thor:~$ telnet 172.16.0.200 25
Trying 172.16.0.200...
Connected to 172.16.0.200.
Escape character is '^]'.
```


#### -En la máquina Loki instala un servidor Postgres si no lo tiene aún. A este servidor se puede acceder desde la DMZ, pero no desde el exterior.

Igualmente no creamos DNAT pues no se accederá desde el exterior, permitimos la conexión desde la red de Hela hacia la red donde está Loki

```
nft add rule inet filter forward ip saddr 172.16.0.0/16 ip daddr 192.168.0.0/24 tcp dport 5432 ct state new,established counter accept
nft add rule inet filter forward ip saddr 192.168.0.0/24 ip daddr 172.16.0.0/16  tcp sport 5432 ct state established counter accept
```


Probamos con un cliente postgres desde hela.
```
[root@hela alex]# psql --host 192.168.0.3 -U alex
Password for user alex: 
psql (13.14, server 14.11 (Ubuntu 14.11-0ubuntu0.22.04.1))
WARNING: psql major version 13, server major version 14.
         Some psql features might not work.
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

alex=# 
```
#### -Evita ataques DoS por ICMP Flood, limitando a 4 el número de peticiones por segundo desde una misma IP.

#### -Evita ataques DoS por SYN Flood.
#### -Evita que realicen escaneos de puertos a Odin


## Reglas

```
root@odin:~# nft list ruleset
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		ip saddr 172.29.0.0/16 tcp dport 22 ct state established,new counter packets 10244 bytes 705699 accept
		ip saddr 172.22.0.0/16 tcp dport 22 ct state established,new counter packets 18738 bytes 1130732 accept
		ip saddr 192.168.0.0/24 tcp sport 22 ct state established counter packets 5015 bytes 659957 accept
		ip saddr 172.16.0.0/16 tcp sport 22 ct state established counter packets 1595 bytes 219134 accept
		counter packets 34915 bytes 2097700
		iifname "ens3" tcp dport 22 ct state established,new counter packets 0 bytes 0 accept
		iifname "lo" counter packets 34726 bytes 2084000 accept
		ip saddr 172.16.0.0/16 icmp type echo-request counter packets 9 bytes 756 accept
		icmp type echo-reply counter packets 12 bytes 1008 accept
		ip saddr 192.168.0.0/24 icmp type echo-request counter packets 7 bytes 588 drop
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		counter packets 83892 bytes 5839416
		ip saddr 172.16.0.200 ip daddr 192.168.0.0/24 icmp type echo-request counter packets 4 bytes 336 accept
		ip daddr 172.16.0.200 ip saddr 192.168.0.0/24 icmp type echo-reply counter packets 4 bytes 336 accept
		ip saddr 172.16.0.200 ip daddr 192.168.0.0/24 tcp dport 22 ct state established,new counter packets 97 bytes 8685 accept
		ip daddr 172.16.0.200 ip saddr 192.168.0.0/24 tcp sport 22 ct state established counter packets 64 bytes 8777 accept
		ip saddr 192.168.0.0/24 ip daddr 172.16.0.200 tcp dport 22 ct state established,new counter packets 98 bytes 13695 accept
		ip saddr 172.16.0.200 ip daddr 192.168.0.0/24 tcp sport 22 ct state established counter packets 76 bytes 12771 accept
		iifname "ens8" oifname "ens3" icmp type echo-request counter packets 4 bytes 336 accept
		iifname "ens3" oifname "ens8" icmp type echo-reply counter packets 4 bytes 336 accept
		iifname "br-intra" oifname "ens3" icmp type echo-request counter packets 8 bytes 672 accept
		iifname "ens3" oifname "br-intra" icmp type echo-reply counter packets 8 bytes 672 accept
		iifname "br-intra" oifname "ens3" tcp dport { 80, 443 } ct state established,new counter packets 14 bytes 871 accept
		iifname "ens3" oifname "br-intra" tcp sport { 80, 443 } ct state established counter packets 4 bytes 602 accept
		iifname "ens8" oifname "ens3" tcp dport { 80, 443 } ct state established,new counter packets 6 bytes 391 accept
		iifname "ens3" oifname "ens8" tcp sport { 80, 443 } ct state established counter packets 4 bytes 602 accept
		iifname "ens3" oifname "ens8" tcp dport { 80, 443 } ct state established,new counter packets 18 bytes 1701 accept
		iifname "ens8" oifname "ens3" tcp sport { 80, 443 } ct state established counter packets 16 bytes 2371 accept
		iifname "ens3" oifname "ens8" tcp dport 21 ct state established,new counter packets 11 bytes 610 accept
		iifname "ens8" oifname "ens3" tcp sport 21 ct state established counter packets 9 bytes 572 accept
		iifname "br-intra" oifname "ens8" tcp dport { 80, 443 } ct state established,new counter packets 6 bytes 396 accept
		iifname "ens8" oifname "br-intra" tcp sport { 80, 443 } ct state established counter packets 4 bytes 488 accept
		iifname "br-intra" oifname "ens8" tcp dport 21 ct state established,new counter packets 12 bytes 657 accept
		iifname "ens8" oifname "br-intra" tcp sport 21 ct state established counter packets 10 bytes 662 accept
		iifname "br-intra" oifname "ens8" tcp dport 25 ct state established,new counter packets 9 bytes 486 accept
		iifname "ens8" oifname "br-intra" tcp sport 25 ct state established counter packets 6 bytes 451 accept
		ip saddr 172.16.0.0/16 ip daddr 192.168.0.0/24 tcp dport 5432 ct state established,new counter packets 33 bytes 3731 accept
		ip saddr 192.168.0.0/24 ip daddr 172.16.0.0/16 tcp sport 5432 ct state established counter packets 24 bytes 5405 accept
	}

	chain output {
		type filter hook output priority filter; policy drop;
		ip daddr 172.29.0.0/16 tcp sport 22 ct state established counter packets 8182 bytes 1379426 accept
		ip daddr 172.22.0.0/16 tcp sport 22 ct state established counter packets 16678 bytes 2335908 accept
		ip daddr 192.168.0.0/24 tcp dport 22 ct state established,new counter packets 7670 bytes 608796 accept
		ip daddr 172.16.0.0/16 tcp dport 22 ct state established,new counter packets 1982 bytes 163532 accept
		counter packets 47687 bytes 2961362
		oifname "ens3" tcp sport 22 ct state established counter packets 0 bytes 0 accept
		oifname "lo" counter packets 34726 bytes 2084000 accept
		ip daddr 172.16.0.0/16 icmp type echo-reply counter packets 9 bytes 756 accept
		icmp type echo-request counter packets 12 bytes 1008 accept
		ip daddr 192.168.0.0/24 icmp type echo-reply counter packets 0 bytes 0 drop
	}
}
table inet nat {
	chain prerouting {
		type nat hook prerouting priority filter; policy accept;
		iifname "ens3" tcp dport 2222 counter packets 2 bytes 120 dnat ip to 10.0.200.217:22
		iifname "ens3" tcp dport 80 counter packets 6 bytes 360 dnat ip to 172.16.0.200
		iifname "ens3" tcp dport 21 counter packets 1 bytes 60 dnat ip to 172.16.0.200
	}

	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		ip saddr 172.16.0.0/16 oifname "ens3" counter packets 2 bytes 144 masquerade
		ip saddr 192.168.0.0/24 oifname "ens3" counter packets 5 bytes 348 masquerade
	}
}
```