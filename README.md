# 🔐 IPSec IKEv2 — VPN Site-to-Site Basada en Enrutamiento

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![IPSec](https://img.shields.io/badge/Protocolo-IPSec%20IKEv2-blue?style=for-the-badge&logo=cisco)
![Type](https://img.shields.io/badge/Tipo-Site--to--Site-brightgreen?style=for-the-badge)
![Mode](https://img.shields.io/badge/Modo-Route--Based-orange?style=for-the-badge)
![Platform](https://img.shields.io/badge/Plataforma-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Verificación del Túnel](#6-verificación-del-túnel)
7. [Capturas de Pantalla](#7-capturas-de-pantalla)
8. [Video Demostrativo](#8-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Site-to-Site basada en enrutamiento (route-based)** utilizando **IPSec con IKEv2** en routers Cisco IOS dentro de PNetLab. La práctica cubre:

- Configuración de **IKEv2** mediante `crypto ikev2 proposal`, `crypto ikev2 policy`, `crypto ikev2 keyring` y `crypto ikev2 profile`.
- Creación de una **Virtual Tunnel Interface (VTI)** con `tunnel mode ipsec ipv4` y `tunnel protection ipsec profile`, que aplica el cifrado directamente sobre la interfaz lógica sin necesidad de ACL de tráfico interesante.
- Protección del tráfico entre `202.50.73.128/25` (Site A) y `202.50.73.0/25` (Site B) a través del segmento `192.168.19.0/24` simulado por la nube NAT de PNetLab.
- Validación del túnel con comandos `show ikev2` y `show interface tunnel`, y pruebas de conectividad entre hosts.

### ¿Qué combina este lab?

Este laboratorio es la combinación de los dos modelos más modernos: la **negociación eficiente de IKEv2** (4 mensajes vs 9 de IKEv1) con la **flexibilidad del modelo route-based** (tabla de enrutamiento como selector de tráfico, soporte de protocolos dinámicos sobre el túnel). Es el enfoque recomendado para entornos de producción actuales.

```
PC1 → R1 → [Ruta hacia 202.50.73.0/25 apunta a Tunnel0]
               ↓ tunnel protection ipsec profile (IKEv2)
          [IKEv2 cifra el paquete con ESP] → ISP → R2 → Descifra → PC2
```

---

## 2. Topología

```
                              [ INTERNET / ISP ]
                               192.168.19.0/24
                              (Cloud/NAT: 192.168.19.2)
                                      │
                  ┌───────────────────┴───────────────────┐
                  │ e0/0: 192.168.19.5                     │ e0/0: 192.168.19.6
          ┌───────┴────────┐                      ┌────────┴───────┐
          │   R1 (Peer A)  │◄══ Tunnel0 (VTI) ════►  R2 (Peer B)  │
          │                │     IPSec IKEv2       │               │
          └───────┬────────┘     Route-Based       └────────┬──────┘
                  │ e0/1: 202.50.73.129/25                  │ e0/1: 202.50.73.1/25
                  │                                         │
          ┌───────┴────────┐                      ┌─────────┴──────┐
          │     SW1        │                      │      SW2       │
          └───────┬────────┘                      └────────┬───────┘
                  │                                        │
          ┌───────┴────────┐                      ┌────────┴───────┐
          │      PC1       │                      │      PC2       │
          │ 202.50.73.130  │                      │  202.50.73.2   │
          └────────────────┘                      └────────────────┘
           ◄── SITE A ──►                          ◄── SITE B ──►
           202.50.73.128/25                        202.50.73.0/25
```

> El túnel VTI `Tunnel0` se levanta entre `192.168.19.5` (R1) y `192.168.19.6` (R2), usando `10.0.0.0/30` como red interna del túnel.  
> El cifrado IKEv2 se aplica automáticamente vía `tunnel protection ipsec profile` — no se usa Crypto Map ni ACL.

**Flujo de establecimiento:**
1. R1 tiene una ruta estática hacia `202.50.73.0/25` apuntando a `Tunnel0`.
2. Al recibir tráfico de PC1, lo encamina por `Tunnel0` → el perfil IPSec IKEv2 lo cifra automáticamente.
3. IKEv2 negocia el túnel en 4 mensajes → SA establecida en estado `READY`.
4. El paquete cifrado viaja por el Cloud NAT hasta R2 → descifra → entrega a PC2.

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo | Interfaz    | Dirección IP      | Máscara | Gateway        | Rol                         |
|-------------|-------------|-------------------|---------|----------------|-----------------------------|
| Cloud (NAT) | —           | 192.168.19.2      | /24     | —              | Gateway Cloud NAT (PNetLab) |
| **R1**      | **e0/0**    | **192.168.19.5**  | **/24** | 192.168.19.2   | WAN Peer A → Cloud          |
| **R1**      | **e0/1**    | **202.50.73.129** | **/25** | —              | Gateway LAN Site A          |
| **R1**      | **Tunnel0** | **10.0.0.1**      | **/30** | —              | Túnel VTI hacia R2          |
| **R2**      | **e0/0**    | **192.168.19.6**  | **/24** | 192.168.19.2   | WAN Peer B → Cloud          |
| **R2**      | **e0/1**    | **202.50.73.1**   | **/25** | —              | Gateway LAN Site B          |
| **R2**      | **Tunnel0** | **10.0.0.2**      | **/30** | —              | Túnel VTI hacia R1          |
| SW1         | —           | —                 | —       | —              | Capa 2 Site A               |
| SW2         | —           | —                 | —       | —              | Capa 2 Site B               |
| PC1         | eth0        | 202.50.73.130     | /25     | 202.50.73.129  | Host Site A                 |
| PC2         | eth0        | 202.50.73.2       | /25     | 202.50.73.1    | Host Site B                 |

### Tabla de Subredes

| Subred             | Rango Utilizable              | Broadcast       | Uso                   |
|--------------------|-------------------------------|-----------------|-----------------------|
| `192.168.19.0/24`  | 192.168.19.1 – 192.168.19.254 | 192.168.19.255  | Segmento WAN/Cloud    |
| `202.50.73.0/25`   | 202.50.73.1 – 202.50.73.126   | 202.50.73.127   | LAN Site B            |
| `202.50.73.128/25` | 202.50.73.129 – 202.50.73.254 | 202.50.73.255   | LAN Site A            |
| `10.0.0.0/30`      | 10.0.0.1 – 10.0.0.2           | 10.0.0.3        | Interfaz Tunnel (VTI) |

---

## 4. Parámetros Configurados

### IKEv2 Proposal

| Parámetro  | Valor               | Descripción                                             |
|------------|---------------------|---------------------------------------------------------|
| Nombre     | `IKEv2_PROP`        | Identificador del proposal                              |
| Cifrado    | AES-CBC-256         | Cifrado simétrico del canal IKEv2                       |
| Integridad | SHA-256             | Verificación de integridad de mensajes IKEv2            |
| Grupo DH   | Group 14 (2048-bit) | Intercambio Diffie-Hellman para derivar clave de sesión |

### IKEv2 Policy

| Parámetro | Valor        | Descripción                                  |
|-----------|--------------|----------------------------------------------|
| Nombre    | `IKEv2_POL`  | Política que asocia el proposal al peer       |
| Proposal  | `IKEv2_PROP` | Vincula la política al conjunto de algoritmos |

### IKEv2 Keyring

| Parámetro      | Valor            | Descripción                        |
|----------------|------------------|------------------------------------|
| Keyring        | `IKEv2_KEYRING`  | Almacén de claves pre-compartidas  |
| Peer R1→R2     | 192.168.19.6     | IP del peer remoto desde R1        |
| Peer R2→R1     | 192.168.19.5     | IP del peer remoto desde R2        |
| Pre-Shared Key | `ITLA2025Arlene` | Clave idéntica en ambos routers    |

### IKEv2 Profile

| Parámetro      | Valor            | Descripción                                          |
|----------------|------------------|------------------------------------------------------|
| Nombre         | `IKEv2_PROFILE`  | Agrupa keyring, identidad y autenticación            |
| Match Identity | address (IP WAN) | Identifica al peer por su dirección IP pública       |
| Authentication | pre-share        | Método de autenticación local y remota               |
| Keyring        | `IKEv2_KEYRING`  | Referencia al keyring con la PSK                     |

### IPSec Profile (VTI)

| Parámetro     | Valor               | Descripción                                         |
|---------------|---------------------|-----------------------------------------------------|
| Nombre        | `IPSEC_PROFILE_VTI` | Perfil aplicado sobre `Tunnel0` con `tunnel protection` |
| Transform Set | `TS_AES256_SHA256`  | ESP-AES-256 + ESP-SHA256-HMAC, modo Tunnel          |
| IKEv2 Profile | `IKEv2_PROFILE`     | Vincula el perfil IKEv2 al perfil IPSec             |
| Lifetime SA   | 3600 s (1 h)        | Duración del túnel de datos antes de renegociar     |

### Interfaz Tunnel (VTI)

| Parámetro          | R1            | R2            |
|--------------------|---------------|---------------|
| IP Tunnel          | 10.0.0.1/30   | 10.0.0.2/30   |
| Tunnel Source      | Ethernet0/0   | Ethernet0/0   |
| Tunnel Destination | 192.168.19.6  | 192.168.19.5  |
| Tunnel Mode        | ipsec ipv4    | ipsec ipv4    |
| Tunnel Protection  | IPSEC_PROFILE_VTI | IPSEC_PROFILE_VTI |

---

## 5. Scripts de Configuración

### R1 — Site A

```cisco
! ══════════════════════════════════════════════════════════════
!  R1 — Site A | IPSec IKEv2 Route-Based VPN (VTI)
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R1

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Cloud
 ip address 192.168.19.5 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteA
 ip address 202.50.73.129 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: IKEv2 Proposal ──────────────────────────────────
crypto ikev2 proposal IKEv2_PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

! ── Paso 2: IKEv2 Policy ────────────────────────────────────
crypto ikev2 policy IKEv2_POL
 proposal IKEv2_PROP

! ── Paso 3: IKEv2 Keyring (PSK) ─────────────────────────────
crypto ikev2 keyring IKEv2_KEYRING
 peer R2
  address 192.168.19.6
  pre-shared-key ITLA2025Arlene

! ── Paso 4: IKEv2 Profile ───────────────────────────────────
crypto ikev2 profile IKEv2_PROFILE
 match identity remote address 192.168.19.6 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEv2_KEYRING

! ── Paso 5: Transform Set ───────────────────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 6: IPSec Profile (se aplica en Tunnel0) ────────────
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set ikev2-profile IKEv2_PROFILE
 set security-association lifetime seconds 3600

! ── Paso 7: Interfaz de Túnel Virtual (VTI) ──────────────────
interface Tunnel0
 description VTI-IKEv2-hacia-R2
 ip address 10.0.0.1 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.6
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE_VTI

! ── Paso 8: Ruta estática hacia LAN remota vía Tunnel0 ───────
ip route 202.50.73.0 255.255.255.128 Tunnel0
```

### R2 — Site B

```cisco
! ══════════════════════════════════════════════════════════════
!  R2 — Site B | IPSec IKEv2 Route-Based VPN (VTI)
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

hostname R2

! ── Interfaces físicas ──────────────────────────────────────
interface Ethernet0/0
 description WAN-hacia-Cloud
 ip address 192.168.19.6 255.255.255.0
 no shutdown

interface Ethernet0/1
 description LAN-SiteB
 ip address 202.50.73.1 255.255.255.128
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.19.2

! ── Paso 1: IKEv2 Proposal ──────────────────────────────────
crypto ikev2 proposal IKEv2_PROP
 encryption aes-cbc-256
 integrity sha256
 group 14

! ── Paso 2: IKEv2 Policy ────────────────────────────────────
crypto ikev2 policy IKEv2_POL
 proposal IKEv2_PROP

! ── Paso 3: IKEv2 Keyring (PSK) ─────────────────────────────
crypto ikev2 keyring IKEv2_KEYRING
 peer R1
  address 192.168.19.5
  pre-shared-key ITLA2025Arlene

! ── Paso 4: IKEv2 Profile ───────────────────────────────────
crypto ikev2 profile IKEv2_PROFILE
 match identity remote address 192.168.19.5 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEv2_KEYRING

! ── Paso 5: Transform Set ───────────────────────────────────
crypto ipsec transform-set TS_AES256_SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel

! ── Paso 6: IPSec Profile (se aplica en Tunnel0) ────────────
crypto ipsec profile IPSEC_PROFILE_VTI
 set transform-set TS_AES256_SHA256
 set ikev2-profile IKEv2_PROFILE
 set security-association lifetime seconds 3600

! ── Paso 7: Interfaz de Túnel Virtual (VTI) ──────────────────
interface Tunnel0
 description VTI-IKEv2-hacia-R1
 ip address 10.0.0.2 255.255.255.252
 tunnel source Ethernet0/0
 tunnel destination 192.168.19.5
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile IPSEC_PROFILE_VTI

! ── Paso 8: Ruta estática hacia LAN remota vía Tunnel0 ───────
ip route 202.50.73.128 255.255.255.128 Tunnel0
```

### Hosts (VPCs en PNetLab)

```bash
# PC1 — Site A
ip 202.50.73.130 255.255.255.128 202.50.73.129

# PC2 — Site B
ip 202.50.73.2 255.255.255.128 202.50.73.1
```

---

## 6. Verificación del Túnel

### Estado de la interfaz Tunnel

```cisco
R1# show interface tunnel 0
```

Salida esperada:

```
Tunnel0 is up, line protocol is up
  Internet address is 10.0.0.1/30
  Tunnel source 192.168.19.5, destination 192.168.19.6
  Tunnel protocol/transport IPSEC/IP
```

> `line protocol is up` confirma que el túnel VTI con IKEv2 está operativo.

---

### Estado de la sesión IKEv2

```cisco
R1# show crypto ikev2 sa
```

Salida esperada:

```
 IPv4 Crypto IKEv2  SA

Tunnel-id Local                 Remote                fvrf/ivrf            Status
1         192.168.19.5/500      192.168.19.6/500      none/none            READY
      Encr: AES-CBC, keysize: 256, PRF: SHA256, Hash: SHA256, DH Grp:14, Auth sign: PSK, Auth verify: PSK
      Life/Active Time: 86400/98 sec
```

---

### Estado IPSec SA

```cisco
R1# show crypto ipsec sa
```

Salida esperada (fragmento):

```
interface: Tunnel0
   Crypto map tag: Tunnel0-head-0, local addr 192.168.19.5

    #pkts encaps: 20, #pkts encrypt: 20, #pkts digest: 20
    #pkts decaps: 20, #pkts decrypt: 20, #pkts verify: 20
```

> El Crypto Map se genera automáticamente asociado a `Tunnel0` — no se configura manualmente.

---

### Verificar tabla de enrutamiento

```cisco
R1# show ip route static
```

Salida esperada:

```
S    202.50.73.0/25 is directly connected, Tunnel0
```

---

### Conectividad extremo a extremo

```bash
# Desde PC1 hacia PC2
PC1> ping 202.50.73.2

# Ping al extremo del túnel desde R1
R1# ping 10.0.0.2 source 10.0.0.1
```

Resultado esperado:

```
!!!!!!!!!!
Success rate is 100 percent (10/10)
```

---

### Tabla de Comandos de Verificación

| Comando | Qué verifica |
|---|---|
| `show interface tunnel 0` | Estado del VTI (debe ser `up/up`). |
| `show crypto ikev2 sa` | Estado de la sesión IKEv2. Debe mostrar `READY`. |
| `show crypto ikev2 sa detailed` | Algoritmos negociados, lifetime y autenticación del peer. |
| `show crypto ipsec sa` | SAs IPSec activas asociadas a `Tunnel0`. |
| `show ip route static` | Confirma ruta hacia LAN remota apuntando a `Tunnel0`. |
| `show crypto session` | Resumen rápido del estado de la sesión IPSec/IKEv2. |

---

---

## 7. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.png) | Topología en PNetLab con nombre y matrícula visibles, todos los nodos encendidos. |
| 2 | [Config R1 – IKEv2 + VTI](evidencias/2.png) | Consola R1: `ikev2 profile`, `ipsec profile` y `Tunnel0` con `tunnel protection` configurados. |
| 3 | [IKEv2 SA READY + Tunnel up](evidencias/3.png) | Salida de `show crypto ikev2 sa` (`READY`) y `show interface tunnel 0` (`up/up`). |
| 4 | [Ping exitoso](evidencias/4.png) | Ping exitoso de PC1 (`202.50.73.130`) a PC2 (`202.50.73.2`) con VTI IKEv2 activo. |

---

## 8. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](#)**

---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 05: IPSec IKEv2 Route-Based VPN*

</div>
