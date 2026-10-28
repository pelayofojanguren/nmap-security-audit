# Auditoría básica de seguridad con Nmap
Reconocimiento, análisis de servicios y propuesta de contramedidas.

* **Autor:** Pelayo Fernández Ojanguren
* **Módulo:** Seguridad y alta disponibilidad (SAD)
* **Documento completo:** [`Auditoria basica de seguridad con Nmap.pdf`](./Auditoria%20basica%20de%20seguridad%20con%20Nmap.pdf)

---

## 1. Instalación y verificación de Nmap
En la máquina atacante (Lubuntu), actualizamos repositorios e instalamos la herramienta:
```bash
sudo apt update && sudo apt install nmap -y
```
Comprobamos la versión para verificar su correcta instalación en el sistema:
```bash
nmap --version
```
> Resultado: `Nmap version 7.94SVN` sobre plataforma `x86_64-pc-linux-gnu`.

---

## 2. Identificación del objetivo
Identificamos la IP de la máquina objetivo ejecutando `ip a` en el servidor (obteniendo `192.168.1.220`). A continuación, comprobamos la conectividad de red desde Lubuntu enviando 4 paquetes ICMP:
```bash
ping -c 4 192.168.1.220
```
> Resultado: 4 paquetes transmitidos y 4 recibidos (0% pérdida de paquetes), confirmando que la conexión es correcta.

---

## 3. Escaneo de puertos

### Escaneo SYN básico (Top 1000 puertos)
```bash
sudo nmap -sS 192.168.1.220
```
* **Puerto 22/tcp:** Abierto (`open`), correspondiente al servicio SSH.
* **999 puertos filtrados:** Sin respuesta (`no-response`), lo que indica que el cortafuegos local o perimetral descarta los paquetes silenciosamente.

### Detección de versiones de servicios
```bash
sudo nmap -sV 192.168.1.220
```
* **Versión identificada:** `OpenSSH 10.2p1 Ubuntu 2ubuntu3.7` (protocolo 2.0).
* **Sistema operativo:** Linux derivado de Ubuntu.

### Detección del sistema operativo
```bash
sudo nmap -O 192.168.1.220
```
* **Limitación del sondeo:** Al no encontrar al menos un puerto cerrado para comparar respuestas, Nmap advierte de que el resultado puede no ser 100% exacto.
* **Resultado:** Clasifica el equipo como un entorno virtualizado con un **98% de probabilidad de ser Oracle VirtualBox**.

### Escaneo completo de todos los puertos (65.535 puertos)
Para optimizar el tiempo en un host con filtrado de paquetes, fijamos un ratio mínimo de sondeo y limitamos los reintentos:
```bash
sudo nmap -p- --min-rate 1000 --max-retries 1 192.168.1.220
```
* **22/tcp:** Único puerto abierto en todo el sistema.
* **62.108 puertos:** Filtrados sin respuesta (*drop* silencioso).
* **349 puertos:** Filtrados pero inalcanzables (*net-unreach*).
* **3.077 puertos:** Cerrados al responder con paquete TCP RST.

---

## 4. Análisis de los resultados

* **Servicio:** SSH (`OpenSSH 10.2p1 Ubuntu 2ubuntu3.7`).
* **Utilidad:** Protocolo seguro para la administración remota de servidores mediante terminal cifrada y transferencia de archivos, sustituyendo a alternativas inseguras sin cifrado como Telnet.
* **Vulnerabilidades asociadas:**
  * **Ataques de fuerza bruta o diccionario:** Al estar el servicio expuesto, se pueden realizar intentos continuos para adivinar credenciales si no hay límites configurados.
  * **Acceso directo como root:** Si la directiva permite login directo del superusuario, el atacante solo necesita adivinar una contraseña para obtener control absoluto del servidor.
  * **Fallos de software:** Posibles vulnerabilidades conocidas si la paquetería de OpenSSH no se mantiene actualizada.
* **Nivel de riesgo:** **Medio** en red local; **Alto** si el puerto estuviera expuesto a Internet sin bastionar.

---

## 5. Propuesta de contramedidas
Para proteger el puerto 22 y mitigar los vectores detectados:
1. **Deshabilitar inicio de sesión directo como root:** Establecer `PermitRootLogin no` en `/etc/ssh/sshd_config`, obligando a acceder con un usuario común y elevar privilegios con `sudo`.
2. **Protección contra fuerza bruta:** Instalar **Fail2ban** para monitorizar intentos fallidos en los logs y bloquear temporalmente las IPs atacantes en el firewall.
3. **Filtrado por IP en firewall:** Configurar el cortafuegos para permitir conexiones entrantes a SSH únicamente desde IPs o subredes de confianza.
4. **Actualizaciones periódicas:** Mantener `openssh-server` actualizado para corregir posibles vulnerabilidades conocidas.

---

## 6. Inventario de servicios

| Puerto | Servicio | Versión detectada | Nivel de riesgo | Medidas propuestas |
| :---: | :---: | :--- | :---: | :--- |
| **22/tcp** | **SSH** | `OpenSSH 10.2p1 Ubuntu 2ubuntu3.7 (protocol 2.0)` | **Medio** (red local) | Deshabilitar acceso directo a root, instalar herramienta de monitorización para bloquear IPs tras intentos fallidos y limitar acceso en firewall a IPs autorizadas. |

---

## Conclusión
La superficie de ataque del servidor es muy reducida, limitando la entrada exclusivamente al puerto 22. Aun teniendo una versión moderna de OpenSSH, es imprescindible bastionar el servicio (deshabilitar root directo, limitar IPs y usar herramientas contra fuerza bruta). 

El descarte silencioso en la mayoría de puertos confirma la presencia de un cortafuegos activo. Por último, la metodología por fases con Nmap (comprobación con ping, escaneo SYN, detección de versiones y barrido completo) demostró ser la forma más limpia y eficiente de auditar el sistema sin saturar la red.
