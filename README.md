# 🔐 FortiGate — VPN Client-to-Site SSL VPN

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![FortiGate](https://img.shields.io/badge/Plataforma-FortiGate-red?style=for-the-badge&logo=fortinet)
![SSLVPN](https://img.shields.io/badge/Protocolo-SSL%20VPN-blue?style=for-the-badge)
![Client](https://img.shields.io/badge/Cliente-FortiClient%20Linux-orange?style=for-the-badge&logo=linux)
![Platform](https://img.shields.io/badge/Entorno-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Configuración del Cliente FortiClient en Linux](#6-configuración-del-cliente-forticlient-en-linux)
7. [Verificación mediante GUI](#7-verificación-mediante-gui)
8. [Capturas de Pantalla](#8-capturas-de-pantalla)
9. [Video Demostrativo](#9-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Client-to-Site SSL VPN** entre un cliente **Linux con FortiClient** y un firewall **FortiGate** en PNetLab. La práctica cubre:

- Configuración del **FortiGate** como servidor SSL VPN con portal en modo túnel, escuchando en el puerto `10443`.
- Creación de un **pool de IPs** para clientes VPN (`10.73.30.200–10.73.30.210`) y un portal SSL VPN con `split-tunneling disable` (todo el tráfico del cliente pasa por el túnel).
- Autenticación de usuarios mediante **base de datos local** del FortiGate (usuario `ArleneVPN`).
- Configuración de **política de firewall** que permite el tráfico desde la interfaz `ssl.root` hacia la LAN interna (`port2`).
- Conexión desde **Linux** usando **FortiClient** y verificación de conectividad hacia PC1 mediante **traceroute** a través de la GUI del FortiGate.

### ¿Por qué SSL VPN en lugar de IPSec?

SSL VPN usa TLS sobre el puerto `10443` (HTTPS), lo que lo hace más sencillo de configurar en clientes Linux sin drivers adicionales y más compatible con entornos con NAT o firewalls intermedios. FortiClient en Linux se conecta directamente al portal SSL del FortiGate sin necesidad de configurar PSK ni Phase 1/Phase 2 manualmente.

```
[Cliente Linux — FortiClient]
    ↓ Conexión HTTPS al puerto 10443 (SSL/TLS)
    ↓ Autenticación usuario ArleneVPN / MiClaveVPN123
[Recibe IP del pool 10.73.30.200–10.73.30.210]
    ↓ Interfaz ssl.root en FortiGate
    ↓ Política SSLVPN-LAN permite tráfico hacia port2
[Acceso a LAN interna 202.50.73.0/24 → PC1]
```

---

## 2. Topología

```
                         [ ISP ]
                        /       \
                  e0/0 /         \ e0/1
           202.50.73.x/24    192.168.19.2/24
                    │                │
                   e0              port1: 192.168.19.5/24
           ┌────────┴───────┐  ┌────┴──────────────────┐
           │    Cliente     │  │      FortiGate         │
           │  Linux +       │  │   (Servidor SSL VPN)   │
           │  FortiClient   │  └────────────┬───────────┘
           └────────────────┘            port2: 202.50.73.1/24
                                              │
                                         ┌────┴────┐
                                         │   SW    │ e0/0 ↔ e0/2
                                         └────┬────┘
                                            eth0
                                         ┌────┴────┐
                                         │   PC1   │
                                         └─────────┘

  Túnel SSL VPN:
  Cliente (e0 → ISP) ══ TLS puerto 10443 ══ FortiGate port1
  Interfaz virtual del túnel: ssl.root
  IP asignada al cliente: pool 10.73.30.200–10.73.30.210
```

> La interfaz virtual `ssl.root` es creada automáticamente por FortiGate cuando un cliente SSL VPN se conecta. Las políticas de firewall usan esta interfaz como origen del tráfico VPN.

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo      | Interfaz  | Dirección IP       | Máscara | Gateway        | Rol                           |
|------------------|-----------|--------------------|---------|----------------|-------------------------------|
| ISP              | e0/0      | (red del cliente)  | —       | —              | Hacia cliente Linux           |
| ISP              | e0/1      | 192.168.19.2       | /24     | —              | Hacia FortiGate (gateway WAN) |
| **FortiGate**    | **port1** | **192.168.19.5**   | **/24** | 192.168.19.2   | WAN → ISP (role: wan)         |
| **FortiGate**    | **port2** | **202.50.73.1**    | **/24** | —              | LAN interna → SW (role: lan)  |
| **FortiGate**    | ssl.root  | —                  | —       | —              | Interfaz virtual SSL VPN      |
| SW               | e0/0      | —                  | —       | —              | Uplink → FortiGate port2      |
| SW               | e0/2      | —                  | —       | —              | Downlink → PC1                |
| PC1              | eth0      | 202.50.73.x        | /24     | 202.50.73.1    | Host LAN interna              |
| **Cliente VPN**  | tun       | 10.73.30.200–.210  | —       | —              | IP asignada por SSL VPN pool  |

### Tabla de Subredes

| Subred            | Rango Utilizable             | Uso                           |
|-------------------|------------------------------|-------------------------------|
| `192.168.19.0/24` | 192.168.19.1 – 192.168.19.254| Segmento WAN FortiGate ↔ ISP  |
| `202.50.73.0/24`  | 202.50.73.1 – 202.50.73.254  | LAN interna (port2)           |
| `10.73.30.200/32` | 10.73.30.200 – 10.73.30.210  | Pool IPs clientes SSL VPN     |

### Credenciales VPN

| Parámetro    | Valor            |
|--------------|------------------|
| Usuario      | `ArleneVPN`      |
| Contraseña   | `MiClaveVPN123`  |
| Servidor VPN | `192.168.19.5`   |
| Puerto       | `10443`          |
| Portal       | `VPN_PORTAL`     |

---

## 4. Parámetros Configurados

### SSL VPN Settings

| Parámetro          | Valor               | Descripción                                          |
|--------------------|---------------------|------------------------------------------------------|
| Certificado        | `Fortinet_Factory`  | Certificado TLS del servidor SSL VPN                 |
| Puerto             | `10443`             | Puerto HTTPS del portal SSL VPN                      |
| Source Interface   | port1, port2        | Interfaces que aceptan conexiones SSL VPN            |
| Source Address     | all                 | Cualquier IP puede iniciar la conexión               |
| Default Portal     | `VPN_PORTAL`        | Portal asignado por defecto a los usuarios           |
| IP Pool            | `VPN_POOL`          | Pool de IPs asignadas a clientes conectados          |

### Portal SSL VPN

| Parámetro       | Valor        | Descripción                                               |
|-----------------|--------------|-----------------------------------------------------------|
| Nombre          | `VPN_PORTAL` | Nombre del portal SSL VPN                                 |
| Tunnel Mode     | Habilitado   | El cliente recibe una IP virtual y accede a la LAN        |
| IP Pool         | `VPN_POOL`   | Pool `10.73.30.200–10.73.30.210`                          |
| Split Tunneling | Deshabilitado| Todo el tráfico del cliente pasa por el túnel VPN         |

> Con `split-tunneling disable`, todo el tráfico del cliente Linux (incluyendo internet) pasa por el FortiGate. Esto permite que el FortiGate inspecicone y logee toda la actividad del cliente VPN.

### IP Pool

| Parámetro  | Valor          |
|------------|----------------|
| Nombre     | `VPN_POOL`     |
| Tipo       | iprange        |
| Inicio     | 10.73.30.200   |
| Fin        | 10.73.30.210   |

### Usuario y Grupo

| Parámetro   | Valor         |
|-------------|---------------|
| Usuario     | `ArleneVPN`   |
| Contraseña  | `MiClaveVPN123` |
| Tipo        | password (local) |
| Grupo       | `VPN_USERS`   |

### Política de Firewall

| Campo     | Valor          | Descripción                                  |
|-----------|----------------|----------------------------------------------|
| Nombre    | `SSLVPN-LAN`   | Permite tráfico SSL VPN hacia la LAN         |
| Src Intf  | `ssl.root`     | Interfaz virtual de clientes SSL VPN         |
| Dst Intf  | `port2`        | LAN interna                                  |
| Grupos    | `VPN_USERS`    | Solo usuarios del grupo VPN pueden acceder   |
| NAT       | Habilitado     | Traduce la IP del pool al segmento LAN       |
| Acción    | accept         | Permite el tráfico                           |

---

## 5. Scripts de Configuración

### FortiGate — Servidor SSL VPN

```fortios
! ══════════════════════════════════════════════════════════════
!  FortiGate — SSL VPN Client-to-Site
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

! ── Interfaces ──────────────────────────────────────────────
config system interface
    edit "port1"
        set mode static
        set ip 192.168.19.5 255.255.255.0
        set allowaccess ping http https ssh
        set role wan
    next
    edit "port2"
        set mode static
        set ip 202.50.73.1 255.255.255.0
        set allowaccess ping http https ssh
        set role lan
        unset device-identification
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

! ── IP Pool para clientes SSL VPN ───────────────────────────
config firewall address
    edit "VPN_POOL"
        set type iprange
        set start-ip 10.73.30.200
        set end-ip 10.73.30.210
    next
end

! ── Usuario local VPN ────────────────────────────────────────
config user local
    edit "ArleneVPN"
        set type password
        set passwd MiClaveVPN123
    next
end

! ── Grupo de usuarios ────────────────────────────────────────
config user group
    edit "VPN_USERS"
        set member "ArleneVPN"
    next
end

! ── Portal SSL VPN ───────────────────────────────────────────
config vpn ssl web portal
    edit "VPN_PORTAL"
        set tunnel-mode enable
        set ip-pools "VPN_POOL"
        set split-tunneling disable
    next
end

! ── Configuración global SSL VPN ─────────────────────────────
config vpn ssl settings
    set servercert "Fortinet_Factory"
    set tunnel-ip-pools "VPN_POOL"
    set port 10443
    set source-interface "port1" "port2"
    set source-address "all"
    set default-portal "VPN_PORTAL"
end

! ── Política de firewall SSL VPN → LAN ───────────────────────
config firewall policy
    edit 1
        set name "SSLVPN-LAN"
        set srcintf "ssl.root"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set groups "VPN_USERS"
        set nat enable
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
hostname ISP

interface Ethernet0/0
 description Hacia-Cliente-Linux
 no shutdown

interface Ethernet0/1
 description Hacia-FortiGate
 ip address 192.168.19.2 255.255.255.0
 no shutdown
```

---

## 6. Configuración del Cliente FortiClient en Linux

### Instalación de FortiClient en Linux

```bash
# Debian / Ubuntu
wget https://filestore.fortinet.com/forticlient/forticlient_vpn_7.x_amd64.deb
sudo dpkg -i forticlient_vpn_7.x_amd64.deb
sudo apt-get install -f

# Red Hat / CentOS / Fedora
sudo rpm -i forticlient_vpn_7.x.x86_64.rpm
```

> Alternativamente se puede usar el paquete `openfortivpn` para conexión por CLI:
> ```bash
> sudo apt install openfortivpn
> ```

---

### Opción A — Conectar con FortiClient GUI en Linux

1. Abrir **FortiClient** desde el menú de aplicaciones.
2. Ir a la pestaña **Remote Access**.
3. Clic en **"+"** → **Add new connection** → tipo **SSL-VPN**.
4. Configurar:

| Campo | Valor |
|---|---|
| Connection Name | `VPN-ITLA-SSL` |
| Remote Gateway | `192.168.19.5` |
| Port | `10443` |
| Certificate | (aceptar el de fábrica) |
| Username | `ArleneVPN` |

5. Clic en **Save** → **Connect**.
6. Ingresar contraseña `MiClaveVPN123` cuando se solicite.
7. Verificar que se asigna una IP del pool `10.73.30.200–10.73.30.210`.

---

### Opción B — Conectar con openfortivpn (CLI)

```bash
sudo openfortivpn 192.168.19.5:10443 \
  --username=ArleneVPN \
  --password=MiClaveVPN123 \
  --trusted-cert=<hash-del-certificado>
```

Una vez conectado, verificar la IP asignada:

```bash
ip addr show ppp0
# o
ip addr show tun0
```

---

### Verificar conectividad desde Linux

```bash
# Traceroute hacia PC1 en la LAN interna
traceroute 202.50.73.x

# Ping de prueba
ping -c 4 202.50.73.x
```

Resultado esperado del traceroute:

```
traceroute to 202.50.73.x (202.50.73.x), 30 hops max
  1  202.50.73.1    <1 ms    FortiGate port2 (LAN gateway)
  2  202.50.73.x    <2 ms    PC1 (LAN interna)
```

---

## 7. Verificación mediante GUI

### Estado del túnel SSL VPN

**Ruta:** `VPN → SSL-VPN → Monitor`

Muestra los clientes conectados actualmente:
- Nombre de usuario (`ArleneVPN`)
- IP asignada del pool (`10.73.30.200`)
- IP remota del cliente
- Duración de la sesión
- Bytes transmitidos / recibidos

---

### Configuración del portal activo

**Ruta:** `VPN → SSL-VPN Portals`

Verificar que el portal `VPN_PORTAL` tiene:
- Tunnel Mode: ✅ habilitado
- IP Pools: `VPN_POOL`
- Split Tunneling: ✅ deshabilitado

---

### Política de firewall con tráfico activo

**Ruta:** `Policy & Objects → Firewall Policy`

La política `SSLVPN-LAN` debe mostrar contadores de bytes y sesiones activas mayores que 0 tras generar tráfico desde el cliente.

---

### Log de eventos SSL VPN

**Ruta:** `Log & Report → Events → VPN Events`

Confirmar:
- `SSL VPN tunnel up` — túnel SSL establecido.
- `User ArleneVPN logged in` — autenticación exitosa.
- `IP 10.73.30.200 assigned` — IP asignada del pool.

---

### Tabla de Verificación GUI

| Panel GUI | Ruta | Qué confirma |
|---|---|---|
| SSL-VPN Monitor | `VPN → SSL-VPN → Monitor` | Cliente `ArleneVPN` conectado con IP del pool |
| SSL-VPN Portals | `VPN → SSL-VPN Portals` | Portal `VPN_PORTAL` con tunnel mode activo |
| Firewall Policy | `Policy & Objects → Firewall Policy` | Contadores de bytes > 0 en política `SSLVPN-LAN` |
| VPN Events | `Log & Report → Events → VPN Events` | Login exitoso y túnel establecido |

---

### Troubleshooting Rápido

| Síntoma | Causa probable | Solución |
|---|---|---|
| FortiClient no conecta al puerto 10443 | Puerto bloqueado o mal configurado | Verificar `set port 10443` en `vpn ssl settings` y que `port1` tenga `allowaccess https` |
| Error de certificado en FortiClient | Certificado auto-firmado no aceptado | Marcar "trust this certificate" en FortiClient o aceptar con `--trusted-cert` en openfortivpn |
| Autenticación falla | Usuario no existe en el grupo correcto | Verificar que `ArleneVPN` esté en el grupo `VPN_USERS` |
| IP no asignada | Pool agotado o `tunnel-ip-pools` no configurado | Revisar `set tunnel-ip-pools "VPN_POOL"` en `vpn ssl settings` |
| Sin acceso a LAN tras conectar | Política `SSLVPN-LAN` falta o srcintf incorrecto | Verificar que `srcintf` sea `ssl.root` y que el grupo `VPN_USERS` esté aplicado |

---

## 8. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.png) | Topología en PNetLab con nombre y matrícula visibles: Cliente Linux, ISP, FortiGate, SW y PC1 encendidos. |
| 2 | [FortiClient Linux conectado](evidencias/2.png) | FortiClient en Linux mostrando la conexión `VPN-ITLA-SSL` activa con IP asignada del pool `10.73.30.x`. |
| 3 | [GUI FortiGate – SSL-VPN Monitor](evidencias/3.png) | Panel `VPN → SSL-VPN → Monitor` con usuario `ArleneVPN` conectado, IP asignada y bytes transmitidos. |
| 4 | [Traceroute exitoso](evidencias/4.png) | Resultado del traceroute desde el cliente Linux hacia PC1 en la LAN interna a través del túnel SSL VPN. |

---

## 9. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](https://youtu.be/7ZMfNEZ1LQc)**



---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab Extra: FortiGate SSL VPN Client-to-Site (Linux)*

</div>
