# SOC Home Lab: Splunk + Sysmon sobre Windows y Linux

## Resumen

Infraestructura de monitoreo base para un SOC home lab: dos endpoints (Ubuntu Server y Windows 10) enviando telemetría real a un indexador Splunk corriendo en Docker, vía Splunk Universal Forwarder. Sirve como base de datos/logs para los proyectos siguientes del portafolio (simulación de ataques, DFIR, automatización).

## Objetivo

Splunk es una de las herramientas de mayor demanda en el mercado SOC y hasta este proyecto no tenía backing práctico en mi CV. El objetivo fue levantar un SIEM funcional de punta a punta: administración de Splunk, log forwarding, y Sysmon como fuente de telemetría enriquecida.

## Arquitectura

```
┌──────────────────────────┐         ┌────────────────────────────┐
│   Ubuntu Server VM       │         │      Windows 10 VM         │
│                          │         │                            │
│  /var/log/syslog         │         │  Sysmon (config Olaf       │
│  /var/log/auth.log       │         │  Hartong)                  │
│                          │         │  Microsoft-Windows-        │
│  Splunk Universal        │         │  Sysmon/Operational        │
│  Forwarder               │         │                            │
│                          │         │  Splunk Universal          │
│                          │         │  Forwarder                 │
└───────────┬──────────────┘         └─────────────┬──────────────┘
            │  tcp/9997                            │  tcp/9997
            │  (host-only network)                 │  (renderXml=true)
            └────────────────┬─────────────────────┘
                             ▼
                 ┌───────────────────────────┐
                 │   Host: Splunk Free       │
                 │   (contenedor Docker)     │
                 │   192.168.56.1            │
                 │                           │
                 │   índice: linux_logs      │
                 │   índice: windows_sysmon  │
                 └───────────────────────────┘
```

Red host-only de VirtualBox (`192.168.56.1`) conecta ambas VMs al host, donde corre el indexador.

## Stack tecnológico

- **Host:** Splunk Free (Docker)
- **VM Linux:** Ubuntu Server + Splunk Universal Forwarder
- **VM Windows:** Windows 10 + Sysmon (config de Olaf Hartong) + Splunk Universal Forwarder
- **Red:** VirtualBox host-only adapter
- **Virtualización:** VirtualBox

## Prerrequisitos

- VirtualBox instalado
- Docker instalado en el host
- Splunk Universal Forwarder (descarga desde splunk.com, requiere cuenta gratuita)
- [Sysmon](https://learn.microsoft.com/sysinternals/downloads/sysmon) + [configuración de Olaf Hartong](https://github.com/olafhartong/sysmon-modular)

## Instalación y configuración

### 1. Indexador Splunk (host, Docker)

```bash
docker run -d -p 8000:8000 -p 9997:9997 \
  -e SPLUNK_START_ARGS=--accept-license \
  -e SPLUNK_PASSWORD=<tu_password> \
  --name splunk splunk/splunk:latest
```

Habilitar recepción en el puerto 9997 (**no está activo por defecto**):
`Settings → Forwarding and receiving → Configure receiving → Add new → puerto 9997`

### 2. VM Ubuntu Server — Universal Forwarder

Instalar el forwarder y configurar:

`inputs.conf`:
```ini
[monitor:///var/log/syslog]
disabled = false
index = linux_logs

[monitor:///var/log/auth.log]
disabled = false
index = linux_logs
```

`outputs.conf` (crítico — sin este archivo el forwarder instala pero no envía nada):
```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = 192.168.56.1:9997
```

> En el asistente de instalación, el campo **Deployment Server** (puerto 8089) debe quedar vacío — es el plano de gestión. El campo correcto para apuntar al indexador es **Receiving Indexer** (puerto 9997).

### 3. VM Windows 10 — Sysmon + Universal Forwarder

Instalar Sysmon con la config de Olaf Hartong:
```powershell
sysmon64.exe -accepteula -i sysmonconfig.xml
```

`inputs.conf` del forwarder:
```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = false
index = windows_sysmon
renderXml = true
```

`outputs.conf`: mismo contenido que en la VM Linux, apuntando a `192.168.56.1:9997`.

## Verificación

Secuencia recomendada:

```spl
# 1. Confirmar que el indexador está activo
index=_internal

# 2. Confirmar conexión del forwarder
index=_internal sourcetype=splunkd component=TcpInputProc

# 3. Buscar eventos Linux
index=linux_logs

# 4. Buscar eventos Windows Sysmon
index=windows_sysmon
```

Chequeo de red antes de esperar datos en Splunk:
```bash
# Desde la VM Ubuntu
nc -zv 192.168.56.1 9997
```
```powershell
# Desde la VM Windows
Test-NetConnection -ComputerName 192.168.56.1 -Port 9997
```

## Errores comunes / troubleshooting

| Síntoma | Causa | Solución |
|---|---|---|
| Forwarder instala pero no llegan datos | Falta `outputs.conf` | Es una falla silenciosa  crear el archivo manualmente |
| No hay datos en Splunk aunque el forwarder está corriendo | Puerto 9997 no habilitado en el indexador | Habilitar recepción en Settings → Forwarding and receiving |
| Ruta de logs no encontrada | `/var/logs/` en vez de `/var/log/` | Verificar la ruta exacta, es case/typo-sensitive |
| Stanza `disable = true` no funciona | El campo correcto es `disabled`, no `disable` | Corregir el nombre del campo |
| `outputs.conf` no envía datos | `defaultGroup` mal escrito (`default-Group`) o stanza `tcp-server://` en vez de `tcpout-server://` | Revisar casing exacto de cada campo/stanza |

## Capturas
- Se demuestran las capturas en `/screenshots`


## Lo que aprendí

- El asistente del Universal Forwarder tiene dos campos que se confunden fácilmente: **Deployment Server** (8089, plano de gestión) y **Receiving Indexer** (9997, plano de datos). Solo el segundo importa en un lab de un solo nodo.
- La recepción en el puerto 9997 **no viene activada por defecto** en Splunk — hay que habilitarla manualmente.
- La ausencia de `outputs.conf` es un fallo silencioso: todo parece instalado correctamente pero no se envía ningún dato.
- Los typos en archivos `.conf` son difíciles de detectar porque Splunk no siempre lanza un error claro — el casing exacto de cada campo (`defaultGroup`) y stanza (`tcpout-server://`) es crítico.
- Conviene verificar en este orden: indexador corriendo → conectividad de red → fuente de datos activa (Sysmon) → `index=_internal` para confirmar la conexión del forwarder.

## Mejoras futuras

- Sumar Wazuh en paralelo para comparar un SIEM comercial (tier gratis) contra uno 100% open-source.
