# Soc-siem-lab
Laboratorio casero con SIEM (Wazuh) para detectar ataques de fuerza bruta SSH y RDP, documentado como caso de estudio de blue team / SOC analyst.   
Home lab with SIEM (Wazuh) to detect SSH and RDP brute-force attacks, documented as a blue team / SOC analyst case study.
# SOC Home Lab – Deteccion de fuerza bruta SSH y RDP con Wazuh

## Objetivo

Montar un laboratorio casero con SIEM (Wazuh) para detectar ataques de fuerza bruta SSH (Linux) y RDP (Windows), y documentar el proceso como un caso de estudio de blue team / SOC analyst.

## Entorno

- Hipervisor: VirtualBox
- Maquinas:
  - Kali Linux 2026.x (atacante) – IP: 192.168.56.X
  - Ubuntu 24.04 (victima SSH) – IP: 192.168.56.Y
  - Windows 11 (victima RDP) – IP: 192.168.56.Z
  - Wazuh 4.x all-in-one (SIEM) – IP: 192.168.56.W

Todas las maquinas estan en una red interna aislada, sin exposicion a Internet.

## Arquitectura (descripcion textual)

- Kali se usa como atacante para realizar escaneos con nmap y ataques de fuerza bruta con Hydra.
- Ubuntu recibe ataques SSH; los intentos de login quedan registrados en /var/log/auth.log y se envian al agente Wazuh.
- Windows recibe ataques RDP; los eventos de inicio de sesion (exito/fallo) se registran en el visor de eventos (Security, eventos 4624/4625) y se envian al agente Wazuh.
- Wazuh centraliza los logs, genera alertas y permite consultar los eventos mediante su interfaz web.

## Implementacion

### 1. Instalacion de Wazuh

- Se instalo Wazuh all-in-one en Ubuntu Server siguiendo la guia oficial.
- Pasos principales:
  1. Instalar Ubuntu Server en una VM.
  2. Ejecutar el script de instalacion de Wazuh proporcionado en la documentacion oficial.
  3. Acceder a la interfaz web de Wazuh desde el navegador en https://<IP_WAZUH>.

### 2. Configuracion de agentes

- Agente Linux (Ubuntu victima):
  - Instalado desde el repositorio de Wazuh.
  - Configurado para enviar logs de /var/log/auth.log.
- Agente Windows (Windows victima):
  - Instalado el agente de Wazuh para Windows.
  - Configurado para enviar logs del canal Security (eventos de inicio de sesion).

### 3. Reglas y detecciones

Se utilizaron/ajustaron las siguientes detecciones:

- Fuerza bruta SSH:
  - Multiples eventos “Failed password for … from <IP>” en un corto periodo.
  - En Wazuh, reglas existentes para SSH failure pueden correlacionarse para generar una alerta de fuerza bruta.
- Fuerza bruta RDP:
  - Multiples eventos 4625 (Logon failure) en Windows desde la misma IP.
  - Wazuh genera alertas cuando el numero de fallos supera un umbral configurable.

Ejemplo de busqueda conceptual en Splunk (si se usara Splunk en lugar de Wazuh):

index=windows EventCode=4625
| stats count by src_ip, Account_Name
| where count > 5

## Simulacion de ataques

### 1. Escaneo de puertos

Desde Kali:

nmap -sV 192.168.56.Y  # Ubuntu
nmap -sV 192.168.56.Z  # Windows

Resultado esperado en SIEM:

- Aumento de eventos de conexion hacia los puertos escaneados.
- En caso de tener firewall, logs de denegacion de conexiones.

### 2. Fuerza bruta SSH

En Ubuntu:

sudo adduser testuser

Desde Kali:

hydra -l testuser -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.Y

Resultado en SIEM:

- Multiples eventos “Failed password for testuser from <IP_KALI>”.
- Posible alerta de fuerza bruta SSH si se supera el umbral configurado.

### 3. Fuerza bruta RDP

En Windows:

- Habilitar Escritorio Remoto.
- Crear usuario local testadmin.

Desde Kali (con Hydra o herramienta similar):

- Ejecutar ataque de fuerza bruta contra el servicio RDP de 192.168.56.Z.

Resultado en SIEM:

- Multiples eventos 4625 en Windows con Account_Name: testadmin.
- Alertas de fuerza bruta RDP en Wazuh.

## Analisis del incidente (resumen)

- Se detectaron multiples intentos de acceso no autorizado mediante fuerza bruta SSH y RDP.
- Los logs evidencian que los ataques provienen de la misma IP (Kali).
- En un entorno real, esto podria indicar un intento de compromiso de credenciales y movimiento lateral.

Acciones recomendadas en un entorno productivo:

- Bloquear la IP atacante en el firewall.
- Forzar MFA en accesos remotos.
- Implementar politicas de bloqueo de cuenta tras N intentos fallidos.
- Revisar y fortalecer contrasenas de cuentas administrativas.

## Lecciones aprendidas

- Configurar correctamente los agentes es clave para tener visibilidad.
- Las reglas por defecto de Wazuh ya detectan muchos casos basicos, pero pueden afinarse.
- Un SIEM bien configurado permite reconstruir el timeline de un ataque solo con logs.

## Mejoras futuras

- Anadir mas tipos de ataque (phishing interno, ejecucion sospechosa de PowerShell, movimiento lateral).
- Integrar fuentes adicionales (firewall, DNS).
- Automatizar la generacion de reportes a partir de los alerts.

## Nota etica

Este laboratorio se realizo en un entorno completamente aislado y propio, con fines educativos y de aprendizaje en ciberseguridad defensiva. No se ataco ningun sistema externo ni se utilizo el conocimiento para actividades maliciosas.
