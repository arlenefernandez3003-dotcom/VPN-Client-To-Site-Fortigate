# 🔐 FortiGate — VPN Client-to-Site IPSec IKEv1

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![FortiGate](https://img.shields.io/badge/Plataforma-FortiGate-red?style=for-the-badge&logo=fortinet)
![IPSec](https://img.shields.io/badge/Protocolo-IPSec%20IKEv1-blue?style=for-the-badge)
![Client](https://img.shields.io/badge/Cliente-FortiClient-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Entorno-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Configuración del Cliente FortiClient](#6-configuración-del-cliente-forticlient)
7. [Verificación mediante GUI](#7-verificación-mediante-gui)
8. [Capturas de Pantalla](#8-capturas-de-pantalla)
9. [Video Demostrativo](#9-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Client-to-Site** entre un cliente Linux/Windows con **FortiClient** y un firewall **FortiGate** en PNetLab. La práctica cubre:

- Configuración del **FortiGate** como servidor VPN con un rango de IPs para clientes remotos (IP Pool).
- Establecimiento del túnel **IPSec IKEv1** con autenticación por usuario/contraseña local en el FortiGate.
- Configuración de **políticas de firewall** que permitan el tráfico entre el cliente VPN y la LAN interna.
- Conexión desde el cliente usando **FortiClient** (disponible para Linux y Windows) con autenticación de usuario.
- Verificación del túnel activo y prueba de conectividad hacia PC1 mediante **traceroute** desde el cliente.

### ¿Cómo funciona la VPN Client-to-Site en FortiGate?

El FortiGate actúa como concentrador VPN. El cliente FortiClient negocia IPSec IKEv1 con el FortiGate, se autentica con usuario/contraseña local y recibe una IP del pool configurado. Una vez establecido el túnel, el tráfico del cliente hacia la LAN interna viaja cifrado.

```
[Cliente FortiClient]
    ↓ Negocia IPSec IKEv1 con PSK → autenticación usuario/contraseña
[Recibe IP del pool 202.50.73.194–.206]
    ↓ Túnel cifrado
[Acceso a LAN interna 202.50.73.192/26 → PC1]
```

---

## 2. Topología

```
                         [ ISP ]
                        /       \
                  e0/0 /         \ e0/1
           202.50.73.1/26    192.168.19.2/24
                    │                │
                   e0               Gi1: 192.168.19.5/24
           ┌────────┴───────┐  ┌────┴──────────────────┐
           │   Cliente      │  │      FortiGate         │
           │ (Linux/Win)    │  │   (Servidor VPN)       │
           └────────────────┘  └────────────┬───────────┘
                                         Gi2: 202.50.73.193/26
                                             │
                                        ┌────┴────┐
                                        │   SW    │ e0/0 ↔ e0/2
                                        └────┬────┘
                                           eth0
                                        ┌────┴────┐
                                        │   PC1   │
                                        │.194/26  │
                                        └─────────┘

  Túnel VPN Client-to-Site:
  Cliente (e0 → ISP e0/0) ══ IPSec IKEv1 ══ FortiGate Gi1
  IP asignada al cliente: pool 202.50.73.208–.220
```

> El **ISP** actúa como red pública de tránsito. El cliente establece el túnel IPSec hacia `192.168.19.5` (FortiGate port1/Gi1).

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo      | Interfaz  | Dirección IP      | Máscara | Gateway        | Rol                              |
|------------------|-----------|-------------------|---------|----------------|----------------------------------|
| **ISP**          | e0/0      | 202.50.73.1       | /26     | —              | Hacia cliente                    |
| **ISP**          | e0/1      | 192.168.19.2      | /24     | —              | Hacia FortiGate (gateway WAN)    |
| **FortiGate**    | **port1** | **192.168.19.5**  | **/24** | 192.168.19.2   | WAN → ISP (role: wan)            |
| **FortiGate**    | **port2** | **202.50.73.193** | **/26** | —              | LAN interna → SW (role: lan)     |
| SW               | e0/0      | —                 | —       | —              | Uplink → FortiGate port2         |
| SW               | e0/2      | —                 | —       | —              | Downlink → PC1                   |
| PC1              | eth0      | 202.50.73.194     | /26     | 202.50.73.193  | Host LAN interna                 |
| **Cliente VPN**  | e0        | (vía ISP)         | /26     | 202.50.73.1    | Cliente FortiClient Linux/Win    |
| **VPN Pool**     | Virtual   | 202.50.73.208–.220| /26     | —              | IPs asignadas a clientes VPN     |

> El pool VPN usa el rango `202.50.73.208–202.50.73.220` dentro de la subred `/26` de la LAN, permitiendo que los clientes VPN accedan directamente a PC1.

### Tabla de Subredes

| Subred             | Rango Utilizable              | Broadcast       | Uso                         |
|--------------------|-------------------------------|-----------------|------------------------------|
| `192.168.19.0/24`  | 192.168.19.1 – 192.168.19.254 | 192.168.19.255  | Segmento WAN FortiGate ↔ ISP |
| `202.50.73.0/26`   | 202.50.73.1 – 202.50.73.62    | 202.50.73.63    | Red ISP → Cliente            |
| `202.50.73.192/26` | 202.50.73.193 – 202.50.73.254 | 202.50.73.255   | LAN interna + Pool VPN       |

### Credenciales VPN

| Parámetro      | Valor            |
|----------------|------------------|
| Usuario        | `cliente1`       |
| Contraseña     | `cisco123`       |
| Pre-Shared Key | `ITLA2025Arlene` |
| Servidor VPN   | `192.168.19.5`   |

---

## 4. Parámetros Configurados

### Phase 1 — IKEv1

| Parámetro        | Valor             | Descripción                                          |
|------------------|-------------------|------------------------------------------------------|
| IKE Version      | 1                 | Protocolo de negociación IKEv1                       |
| Tipo             | Dialup (client)   | Acepta clientes con IP dinámica                      |
| Proposal         | AES128-SHA1       | Cifrado AES-128 + integridad SHA1                    |
| DH Group         | 2 (1024-bit)      | Intercambio Diffie-Hellman                           |
| Pre-Shared Key   | `ITLA2025Arlene`  | Clave compartida con el cliente FortiClient          |
| Autenticación    | Usuario/contraseña local | Autenticación de identidad del cliente         |
| NAT Traversal    | Habilitado        | Permite túnel a través de NAT                        |

### Phase 2 — IPSec SA

| Parámetro     | Valor          | Descripción                                   |
|---------------|----------------|-----------------------------------------------|
| Proposal      | AES128-SHA1    | Cifrado del tráfico de datos                  |
| PFS           | Group 2        | Perfect Forward Secrecy                       |
| Src Subnet    | 0.0.0.0/0      | Cualquier tráfico del cliente va por el túnel |
| Dst Subnet    | 202.50.73.192/26 | Red LAN interna del FortiGate               |

### IP Pool y Usuario

| Parámetro    | Valor                          |
|--------------|--------------------------------|
| Pool Name    | `POOL-VPN-CLIENTES`            |
| Rango        | 202.50.73.208 – 202.50.73.220  |
| Usuario      | `cliente1` / `cisco123`        |
| Grupo        | `GRP-VPN`                      |

### Políticas de Firewall

| Política       | Src Intf     | Dst Intf | Acción | Descripción                    |
|----------------|--------------|----------|--------|--------------------------------|
| `VPN-a-LAN`    | SSL-VPN/tunnel | port2  | accept | Cliente VPN accede a LAN       |
| `LAN-a-VPN`    | port2        | tunnel   | accept | LAN responde al cliente VPN    |

---

## 5. Scripts de Configuración

### FortiGate — Servidor VPN Client-to-Site

```fortios
! ══════════════════════════════════════════════════════════════
!  FortiGate — VPN Client-to-Site IPSec IKEv1
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

! ── Interfaces ──────────────────────────────────────────────
config system interface
    edit "port1"
        set mode static
        set ip 192.168.19.5 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "port2"
        set mode static
        set ip 202.50.73.193 255.255.255.192
        set allowaccess ping http ssh
        set role lan
    next
end

! ── Ruta por defecto ────────────────────────────────────────
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 192.168.19.2
        set device "port1"
    next
end

! ── Usuario VPN local ────────────────────────────────────────
config user local
    edit "cliente1"
        set type password
        set passwd cisco123
    next
end

! ── Grupo de usuarios VPN ────────────────────────────────────
config user group
    edit "GRP-VPN"
        set member "cliente1"
    next
end

! ── IP Pool para clientes VPN ────────────────────────────────
config firewall ippool
    edit "POOL-VPN-CLIENTES"
        set startip 202.50.73.208
        set endip 202.50.73.220
        set type one-to-one
    next
end

! ── VPN Phase 1 (IKEv1 Dialup) ──────────────────────────────
config vpn ipsec phase1-interface
    edit "VPN-CLIENTES"
        set type dynamic
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device enable
        set proposal aes128-sha1
        set dhgrp 2
        set psksecret ITLA2025Arlene
        set ipv4-start-ip 202.50.73.208
        set ipv4-end-ip 202.50.73.220
        set ipv4-netmask 255.255.255.192
        set usrgrp "GRP-VPN"
        set xauthtype auto
        set authusrgrp "GRP-VPN"
        set nattraversal enable
    next
end

! ── VPN Phase 2 ──────────────────────────────────────────────
config vpn ipsec phase2-interface
    edit "VPN-CLIENTES-P2"
        set phase1name "VPN-CLIENTES"
        set proposal aes128-sha1
        set pfs enable
        set dhgrp 2
        set src-addr-type subnet
        set dst-addr-type subnet
        set src-subnet 0.0.0.0 0.0.0.0
        set dst-subnet 202.50.73.192 255.255.255.192
    next
end

! ── Políticas de Firewall ─────────────────────────────────────
config firewall policy
    edit 1
        set name "VPN-a-LAN"
        set srcintf "VPN-CLIENTES"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 2
        set name "LAN-a-VPN"
        set srcintf "port2"
        set dstintf "VPN-CLIENTES"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end

! ── Logging en memoria ────────────────────────────────────────
config log memory setting
    set status enable
end
```

### ISP

```cisco
! ══════════════════════════════════════════════════════════════
!  ISP — Router de tránsito público
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname ISP

interface Ethernet0/0
 description Hacia-Cliente
 ip address 202.50.73.1 255.255.255.192
 no shutdown

interface Ethernet0/1
 description Hacia-FortiGate
 ip address 192.168.19.2 255.255.255.0
 no shutdown
```

### PC1 — Host LAN interna

```bash
ip 202.50.73.194 255.255.255.192 202.50.73.193
```

---

## 6. Configuración del Cliente FortiClient

FortiClient está disponible para **Windows y Linux** y es el cliente VPN oficial de Fortinet para conectarse a FortiGate.

### Instalación

- **Windows:** Descargar FortiClient VPN desde `https://www.fortinet.com/support/product-downloads`
- **Linux:** Instalar el paquete `forticlient` desde el repositorio oficial o usar el instalador `.deb`/`.rpm`

---

### Crear la conexión VPN en FortiClient

1. Abrir **FortiClient** → pestaña **Remote Access**.
2. Clic en **"+"** para agregar una nueva conexión VPN.
3. Configurar los parámetros:

| Campo | Valor |
|---|---|
| Connection Name | `VPN-ITLA-FortiGate` |
| Remote Gateway | `192.168.19.5` |
| Authentication | Pre-Shared Key |
| Pre-Shared Key | `ITLA2025Arlene` |
| Username | `cliente1` |
| Password | `cisco123` |

4. En la sección **Advanced**:
   - VPN Type: **IPSec VPN**
   - IKE Version: **1**
   - NAT Traversal: **Habilitado**

5. Clic en **Save**.

---

### Conectar

1. Seleccionar `VPN-ITLA-FortiGate` en la lista de conexiones.
2. Clic en **Connect**.
3. FortiClient mostrará el progreso de la negociación IKEv1 y la asignación de IP.
4. Verificar que se asignó una IP del pool `202.50.73.208–.220`.

---

## 7. Verificación mediante GUI

### Estado del túnel en FortiGate

**Ruta:** `VPN → IPsec Tunnels`

El túnel `VPN-CLIENTES` debe aparecer en verde con el número de sesiones activas.

---

### Monitor de sesiones VPN activas

**Ruta:** `Monitor → IPsec Monitor`

Muestra:
- IP remota del cliente conectado.
- Bytes transmitidos y recibidos.
- Estado de Phase 1 y Phase 2.
- Tiempo activo de la sesión.

---

### Verificar IP asignada al cliente

**Ruta:** `Monitor → IPsec Monitor` → expandir el túnel activo.

Debe mostrar la IP del pool asignada al cliente (`202.50.73.208` o la siguiente disponible).

---

### Políticas con tráfico activo

**Ruta:** `Policy & Objects → Firewall Policy`

Los contadores de bytes en las políticas `VPN-a-LAN` y `LAN-a-VPN` deben ser mayores que 0 tras generar tráfico desde el cliente.

---

### Traceroute desde el cliente hacia PC1

**Desde cliente Windows (CMD):**

```cmd
tracert 202.50.73.194
```

**Desde cliente Linux (terminal):**

```bash
traceroute 202.50.73.194
```

Resultado esperado:

```
traceroute to 202.50.73.194 (202.50.73.194)
  1  202.50.73.193   <1 ms    FortiGate port2 (LAN gateway)
  2  202.50.73.194   <2 ms    PC1 (LAN interna)
```

> Si el traceroute llega a PC1 en 1 o 2 saltos a través del túnel VPN, la comunicación Client-to-Site está funcionando correctamente.

---

### Log de eventos VPN

**Ruta:** `Log & Report → Events → VPN Events`

Confirmar:
- `IPsec phase-1 status changed to up` → Phase 1 establecida.
- `IPsec phase-2 status changed to up` → Phase 2 establecida.
- `User cliente1 logged in` → autenticación exitosa.

---

### Tabla de Verificación GUI

| Panel GUI | Ruta | Qué confirma |
|---|---|---|
| IPsec Tunnels | `VPN → IPsec Tunnels` | Túnel en verde con sesiones activas |
| IPsec Monitor | `Monitor → IPsec Monitor` | Phase 1/2 up, IP del cliente, bytes |
| Firewall Policy | `Policy & Objects → Firewall Policy` | Contadores de bytes > 0 |
| VPN Events | `Log & Report → Events → VPN Events` | Login exitoso de `cliente1` |

---

### Troubleshooting Rápido

| Síntoma | Causa probable | Solución |
|---|---|---|
| FortiClient falla en Phase 1 | PSK incorrecta o IKE version distinta | Verificar `ITLA2025Arlene` y que IKE version sea 1 en ambos lados |
| Phase 1 OK pero Phase 2 falla | Proposal Phase 2 no coincide | Confirmar `aes128-sha1` en Phase 2 del FortiGate |
| Autenticación falla | Usuario no existe o contraseña incorrecta | Verificar usuario `cliente1` en `User & Authentication → Local Users` |
| IP no asignada al cliente | Pool agotado o mal configurado | Revisar `POOL-VPN-CLIENTES` en `Policy & Objects → IP Pools` |
| Traceroute no llega a PC1 | Política `VPN-a-LAN` no existe | Verificar ambas políticas bidireccionales en FortiGate |

---

## 8. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.png) | Topología en PNetLab con nombre y matrícula visibles: Cliente, ISP, FortiGate, SW y PC1 encendidos. |
| 2 | [FortiClient conectado](evidencias/2.png) | FortiClient mostrando la conexión `VPN-ITLA-FortiGate` activa con IP asignada del pool. |
| 3 | [GUI FortiGate – IPsec Monitor](evidencias/3.png) | Panel `Monitor → IPsec Monitor` con Phase 1 y Phase 2 activas y cliente `cliente1` conectado. |
| 4 | [Traceroute exitoso](evidencias/4.png) | Resultado del traceroute desde el cliente hacia PC1 (`202.50.73.194`) a través del túnel VPN. |

---

## 9. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](https://youtu.be/7ZMfNEZ1LQc)**

**Duración estimada:** < 8 minutos

---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab Extra: FortiGate VPN Client-to-Site*

</div>
