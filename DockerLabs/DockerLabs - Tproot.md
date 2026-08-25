  
- **Plataforma:** DockerLabs  
- **Dificultad:** Muy Fácil
- **SOG / Categoría:** Remote Code Execution / Backdoor / Service Misconfiguration 
- **Fecha:** 2026-08-22  
  
---  
  
## 1. Resumen Ejecutivo  

  Tproot es una máquina Linux de dificultad muy fácil en la plataforma DockerLabs. La explotación consiste en la identificación de un servicio FTP desactualizado (**vsftpd 2.3.4**). Dicha versión cuenta con un fallo de seguridad histórico (CVE-2011-2523) que incluye una puerta trasera (*backdoor*) integrada en el código fuente, lo que permite obtener una shell interactiva con privilegios de **root** de forma directa.
  
---  
  
## 2. Reconocimiento  
  
### Escaneo de Puertos  
Se ejecutó un escaneo inicial de puertos TCP seguido de una detección de servicios y versiones:  
```bash  
# Escaneo rápido de descubrimiento  
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2 -oN nmap_initial  
  
# Detección de versiones y scripts por defecto  
nmap -sCV -p21,80 172.17.0.2 -oN nmap_services**
```  

![Escaneo inicial con Nmap](./img/Pasted%20image%2020260823130535.png) ![Escaneo de servicios con Nmap](./img/Pasted%20image%2020260823130618.png)

**Puertos abiertos detectados:**  
-  `21/tcp` - **FTP:** vsftpd 2.3.4  
- `80/tcp` - **HTTP:** Apache httpd 2.4.58 (Ubuntu)

![[img/Pasted image 20260823130535.png]]
![[img/Pasted image 20260823130618.png]]
### Enumeración Web  
Se realizó una búsqueda de directorios y archivos ocultos en el servidor web mediante `gobuster`:
```bash  
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt -t 40
```  
![Fuzzing web con Gobuster](./img/Pasted%20image%2020260823130710.png)
No se localizaron recursos adicionales relevantes en el servicio HTTP, dirigiendo el foco de análisis al puerto 21 (FTP).

---  
  
## 3. Explotación 
  
### Vulnerabilidad Identificada  
Para  la versión ftp 2.3.4 existe en esta versión una puerta trasera usando el exploit.
  
### Ejecución del Exploit  
- **Servicio:** vsftpd 2.3.4
- **CVE:** CVE-2011-2523
- **Descripción:** Esta versión contiene una puerta trasera histórica. Al intentar autenticarse introduciendo una carita sonriente `:)` al final del nombre de usuario, el servidor abre una shell interactiva escuchando en el puerto `6200/tcp`.
### Ejecución del Exploit
Se utilizó la base de datos de exploits locales mediante `searchsploit` para descargar el script de automatización en Python:

```bash
# Buscar y copiar el exploit al directorio local
searchsploit -m unix/remote/49757.py

# Ejecutar el exploit apuntando al objetivo
python3 49757.py 172.17.0.2
```
  ![Búsqueda en Searchsploit](./img/Pasted%20image%2020260823130057.png)
![Ejecución del exploit en Python](./img/Pasted%20image%2020260823130140.png)

---  
  
## 4. Escalada de Privilegios 
  La vulnerabilidad explotada en el servicio `vsftpd 2.3.4` ejecuta la llamada al sistema directamente con la identidad del proceso padre, otorgando acceso inmediato como el usuario máximo del sistema (**root**). No fue necesario realizar una fase de escalada de privilegios posterior.
![Confirmación de usuario Root](./img/Pasted%20image%2020260823130215.png)

---  

## 5. Conclusión y Mitigación  
- **Causa Raíz:** En julio de 2011, un atacante comprometió el repositorio oficial del código fuente de `vsftpd` e introdujo un *backdoor* en la versión `2.3.4`. La puerta trasera se activa al enviar la secuencia `:)` en la petición de usuario, ejecutando un listener en el puerto `6200`.
- **Medidas de Mitigación:**
  1. **Actualización de Software:** Actualizar el servicio FTP a una versión estable y soportada.
  2. **Verificación de Integridad:** Validar siempre los pares de claves y sumas de verificación (checksums/PGP) del software descargado antes de desplegarlo en producción.
  3. **Reglas de Firewall (Segmentación):** Restringir los puertos no autorizados (como el 6200) mediante reglas de filtrado de red.