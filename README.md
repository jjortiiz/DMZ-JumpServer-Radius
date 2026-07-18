# Secure Network Infrastructure & Centralized Identity Laboratory

Este repositorio contiene la configuración, la documentación de implementación y las especificaciones de diseño para un proyecto de seguridad de red corporativa. El despliegue integra los servicios de **Windows Server** con la infraestructura de **Cisco IOS** para crear un entorno AAA seguro y centralizado con aplicaciones remotas publicadas y plataformas web cifradas.

Desarrollado como parte del plan de estudios **Seguridad de Red** en el **Instituto Tecnológico de las Américas (ITLA)**.
## 📋 Descripción general del proyecto

El objetivo principal de este proyecto es implementar y hacer cumplir un control de acceso granular, centralizar las políticas de identidad de red y proteger la capa de aplicación en un entorno empresarial emulado.

### Key Components:
- **Servidor AAA centralizado:** Servidor de directivas de red (NPS) de Windows Server que actúa como servidor RADIUS para autenticar a los administradores de red de Cisco frente a Active Directory.

- **Aplicaciones remotas seguras:** Implementación de Servicios de escritorio remoto (RDS) mediante RemoteApp y cliente web HTML5 para aislar el software administrativo.

- **Servidor web cifrado:** Servicios de información de Internet (IIS) configurados con enlaces de dominio personalizados y certificados SSL/TLS.

- **Enrutamiento de red principal:** Configuración de Cisco IOS con aprovisionamiento DHCP, generación de claves criptográficas RSA e integración RADIUS con conmutación por error de base de datos local.
---

## 📐 Topology & Architecture

Todo el entorno fue construido y emulado utilizando **PNETLab** y **VMWare**.

```
                   +-----------------------------+
                   |       Windows Server        |
                   |        (8.61.1.254)         |
                   | AD DS, DNS, IIS, NPS, RDS   |
                   +--------------+--------------+
                                  |
                                  | e0/0
                            +-----+-----+
                            |    R1     | (Cisco Router)
                            +-----+-----+
                                  | e0/1
                                  |
                            +-----+-----+
                            |    SW1    | (Cisco Switch)
                            +--+-----+--+
                               |     |
                 +-------------+     +-------------+
                 |                                 |
           +-----+-----+                     +-----+-----+
           |   Win     |                     |   Win10   |
           | (10.8.61.2)                     | (10.8.61.3)
           +-----------+                     +-----------+
```

### 🔀 Tabla de Direccionamiento

| Device | Interface | IP Address | Subnet Mask | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Windows Server** | — | `8.61.1.254` | `255.255.255.0` | AD DS, DNS, IIS, RemoteApp, NPS (RADIUS) |
| **R1** | `e0/0` | `8.61.1.1` | `255.255.255.0` | Connection to Windows Server |
| **R1** | `e0/1` | `10.8.61.1` | `255.255.255.0` | Local LAN Gateway |
| **SW1** | `e0/0` | — | — | Core Switch Uplink to R1 |
| **Win** | `e0` | `10.8.61.2` | `255.255.255.0` | Client Workstation (DHCP) |
| **Win10** | `e0` | `10.8.61.3` | `255.255.255.0` | Client Workstation (DHCP) |
| **Linux** | `e0` | — | — | Testing node |

---

## ⚙️ Configuración e Implementación

### 1. Servicios de Windows Server
- **Servidor Web (IIS):** Se configuró un sitio corporativo personalizado vinculado a `underwatersrl.underwater.com`. Se protegió mediante un Certificado Digital interno con una clave privada exportable generada en el almacén de certificados Personal (Mi) del sistema.

- **Aplicación remota y cliente web HTML5:** Se configuró un Agente de conexión de Escritorio remoto que coincide con el FQDN externo para mitigar los errores de coincidencia de direcciones del navegador. Microsoft Edge se publicó como una aplicación remota que apunta de forma nativa al portal interno, accesible directamente desde navegadores estándar a través del moderno cliente web HTML5.

- **Servidor RADIUS (NPS):** Se registró el router Cisco R1 (`8.61.1.1`) como cliente de red utilizando una clave compartida predefinida. Se configuraron dos políticas de red independientes que asignan grupos de Active Directory a niveles de ejecución de Cisco mediante atributos específicos del proveedor (`cisco-av-pair`):

- **Política de nivel 15 (Privilegiada):** Devuelve `shell:priv-lvl=15`

- **Política de nivel 1 (Operador):** Devuelve `shell:priv-lvl=1`

### 2. Cisco IOS Router Script (`R1`)

```config
!
hostname R1
!
! --- INTERFACE & ADDRESSING CONFIGURATION ---
interface e0/0
 ip address 8.61.1.1 255.255.255.0
 no shutdown
exit
!
interface e0/1
 ip address 10.8.61.1 255.255.255.0
 no shutdown
exit
!
! --- DHCP POOL FOR LAN CLIENTS ---
ip dhcp excluded-address 10.8.61.1
!
ip dhcp pool LAN_CLIENTES
 network 10.8.61.0 255.255.255.0
 default-router 10.8.61.1
 dns-server 8.61.1.254
exit
!
! --- LOCAL SECURITY & SSH ACTIVATION ---
ip domain-name underwatersrl.underwater.com
crypto key generate rsa general-keys modulus 2048
!
! Emergency fallback local administrator accounts
username admin privilege 15 secret welcome
!
! --- AAA ARCHITECTURE & RADIUS CONFIGURATION ---
aaa new-model
!
radius server NPS_SERVER
 address ipv4 8.61.1.254 auth-port 1812 acct-port 1813
 key Cisco123
exit
!
! Authentication & Authorization chains mapping to RADIUS first, then Local
aaa authentication login default group radius local
aaa authorization exec default group radius local
!
! Force routing engine to pass requests via PAP format
ip radius source-interface GigabitEthernet0/0
radius-server attribute 8 format default
radius-server vsa send authentication
!
! --- LINE APPLICABILITY (VTY / CONSOLE) ---
line con 0
 login authentication default
exit
!
line vty 0 4
 login authentication default
 authorization exec default
 transport input ssh
exit
!
```

---

## 🎓 Authorship & Context
- **Institution:** Instituto Tecnológico de las Américas (ITLA)
- **Student:** Juan Francisco Javier Ortiz (ID: `2025-0861`)
- **Course:** Network Security (Seguridad de Redes)
- **Instructor:** Jonathan Rondon
- **Date:** July 17, 2025
