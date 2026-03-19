# 🔐 VPN Site-To-Site — FortiGate + Router ISP (vIOS)

> **Matrícula:** 20241165  
> **Plataforma:** PNetLab  
> **Dispositivos:** 2x FortiGate vVM · 1x Cisco vIOS · 2x VPCS

---

## 📋 Objetivo de la Red

El objetivo de esta práctica es implementar una topología de **VPN IPsec Site-To-Site** que permita la comunicación segura y cifrada entre dos redes LAN privadas ubicadas en sitios geográficos distintos (Sitio A y Sitio B), simulando un escenario real de interconexión empresarial.

Para lograrlo se utilizaron dos firewalls **FortiGate** como gateways VPN de cada sitio, y un **router Cisco vIOS** como simulación del proveedor de internet (ISP) que enruta el tráfico WAN entre ambos FortiGates.

**Objetivos específicos:**

- Configurar el direccionamiento IP en interfaces WAN y LAN de ambos FortiGates
- Establecer rutas estáticas hacia el ISP desde cada FortiGate
- Crear un túnel VPN IPsec (Fase 1 y Fase 2) entre FGT-SitioA y FGT-SitioB
- Definir políticas de firewall que permitan el tráfico cifrado entre ambas LANs
- Verificar la conectividad extremo a extremo mediante `trace-route` desde los VPCs

---

## 🗺️ Topología


<img width="1469" height="1181" alt="Captura de pantalla 2026-03-18 204033" src="https://github.com/user-attachments/assets/e1de3b40-e4e2-4369-ba61-030bb128f3db" />






---

## 📐 Esquema de Direccionamiento IP

> Basado en matrícula **20241165** → pares: **20, 24, 11, 65**

### 🔸 Redes WAN — /30 (Enlace al ISP)

| Enlace              | Red             | IP ISP        | IP FortiGate     |
|---------------------|-----------------|---------------|------------------|
| FGT-SitioA ↔ ISP   | 172.20.11.0/30  | 172.20.11.1   | 172.20.11.2      |
| FGT-SitioB ↔ ISP   | 172.24.65.0/30  | 172.24.65.1   | 172.24.65.2      |

### 🔸 Redes LAN — /24

| Sitio   | Red LAN        | Gateway (port2)  | VPC           |
|---------|----------------|------------------|---------------|
| Sitio A | 10.20.11.0/24  | 10.20.11.254     | 10.20.11.10   |
| Sitio B | 10.24.65.0/24  | 10.24.65.254     | 10.24.65.10   |

---

## ⚙️ Configuraciones Utilizadas

---

### 📸 Captura 1 — Configuración Router ISP (vIOS)

> Se configuraron las dos interfaces del router ISP para conectar ambos FortiGates a través de la red WAN simulada.

```
Router ISP (vIOS) - Configuración de interfaces WAN
════════════════════════════════════════════════════

Router> enable
Router# configure terminal
Router(config)#
Router(config)# interface GigabitEthernet0/0
Router(config-if)#  ip address 172.20.11.1 255.255.255.252
Router(config-if)#  no shutdown
Router(config-if)#  exit
Router(config)#
Router(config)# interface GigabitEthernet0/1
Router(config-if)#  ip address 172.24.65.1 255.255.255.252
Router(config-if)#  no shutdown
Router(config-if)#  exit
Router(config)#
Router(config)# end
Router# write memory

Building configuration...
[OK]
```

**Explicación:** La interfaz `Gi0/0` se conecta hacia FGT-SitioA con IP `172.20.11.1/30`, y la interfaz `Gi0/1` se conecta hacia FGT-SitioB con IP `172.24.65.1/30`. Ambas redes son /30 ya que solo se necesitan 2 hosts por enlace (ISP y FortiGate).

---

### 📸 Captura 2 — Configuración FGT-SitioA (Interfaces + Ruta Estática)

> Configuración de hostname, interfaces WAN/LAN y ruta por defecto hacia el ISP en el FortiGate del Sitio A.

```
FGT-SitioA - Configuración básica
════════════════════════════════════════════════════════════

config system global
    set hostname FGT-SitioA
end

config system interface
    edit port1
        set ip 172.20.11.2 255.255.255.252     <- WAN hacia ISP
        set allowaccess ping http https
    next
    edit port2
        set ip 10.20.11.254 255.255.255.0      <- LAN Sitio A
        set allowaccess ping http https
    next
end

config router static
    edit 1
        set device port1
        set gateway 172.20.11.1                <- Gateway ISP
    next
end
```

**Explicación:** `port1` es la interfaz WAN que apunta al ISP. `port2` es la interfaz LAN que sirve como puerta de enlace para los equipos del Sitio A (`10.20.11.0/24`). La ruta estática dirige todo el tráfico saliente a través de `port1` hacia el ISP.

---

### 📸 Captura 3 — Configuración FGT-SitioB (Interfaces + Ruta Estática)

> Misma estructura que el Sitio A pero con el esquema de direccionamiento del Sitio B.

```
FGT-SitioB - Configuración básica
════════════════════════════════════════════════════════════

config system global
    set hostname FGT-SitioB
end

config system interface
    edit port1
        set ip 172.24.65.2 255.255.255.252     <- WAN hacia ISP
        set allowaccess ping http https
    next
    edit port2
        set ip 10.24.65.254 255.255.255.0      <- LAN Sitio B
        set allowaccess ping http https
    next
end

config router static
    edit 1
        set device port1
        set gateway 172.24.65.1                <- Gateway ISP
    next
end
```

**Explicación:** `port1` enlaza con el ISP por la red `172.24.65.0/30`. `port2` es el gateway de la LAN del Sitio B (`10.24.65.0/24`). La ruta estática apunta al ISP como siguiente salto.

---

### 📸 Captura 4 — Configuración VPN IPsec Fase 1 y Fase 2

> Túnel IPsec configurado en ambos FortiGates para establecer el canal cifrado Site-To-Site.

```
FGT-SitioA - Configuración VPN IPsec
════════════════════════════════════════════════════════════

[ FASE 1 — IKE ]
──────────────────────────────────────────────────────────
  Nombre del túnel  : VPN-SitioA-SitioB
  Remote Gateway    : 172.24.65.2
  Interface         : port1
  Mode              : Main
  Auth. Method      : Pre-Shared Key (PSK)
  PSK               : ****************
  IKE Version       : 1
  Encryption        : AES256
  Authentication    : SHA256
  DH Group          : 14

[ FASE 2 — IPsec ]
──────────────────────────────────────────────────────────
  Red local  (src)  : 10.20.11.0/24
  Red remota (dst)  : 10.24.65.0/24
  Encryption        : AES256
  Authentication    : SHA256
  PFS               : Habilitado (Grupo 14)
```

```
FGT-SitioB - Configuración VPN IPsec
════════════════════════════════════════════════════════════

[ FASE 1 — IKE ]
──────────────────────────────────────────────────────────
  Nombre del túnel  : VPN-SitioB-SitioA
  Remote Gateway    : 172.20.11.2
  Interface         : port1
  Mode              : Main
  Auth. Method      : Pre-Shared Key (PSK)
  PSK               : ****************
  IKE Version       : 1
  Encryption        : AES256
  Authentication    : SHA256
  DH Group          : 14

[ FASE 2 — IPsec ]
──────────────────────────────────────────────────────────
  Red local  (src)  : 10.24.65.0/24
  Red remota (dst)  : 10.20.11.0/24
  Encryption        : AES256
  Authentication    : SHA256
  PFS               : Habilitado (Grupo 14)
```

**Explicación:** La Fase 1 establece el canal IKE seguro entre los dos peers WAN. La Fase 2 define el tráfico que será cifrado (LAN Sitio A ↔ LAN Sitio B). Ambos lados deben tener configuraciones simétricas con el mismo PSK para que el túnel levante correctamente.

---

### 📸 Captura 5 — Políticas de Firewall en FortiGate

> Se crearon políticas bidireccionales para permitir el tráfico entre las LANs a través del túnel VPN.

```
FGT-SitioA - Políticas de Firewall
════════════════════════════════════════════════════════════

Política 1: LAN --> Túnel VPN (Salida hacia Sitio B)
  Source Interface  : port2         (LAN local)
  Dest. Interface   : VPN-tunnel
  Source Address    : 10.20.11.0/24
  Dest. Address     : 10.24.65.0/24
  Action            : ACCEPT
  NAT               : DISABLED

Política 2: Túnel VPN --> LAN (Entrada desde Sitio B)
  Source Interface  : VPN-tunnel
  Dest. Interface   : port2         (LAN local)
  Source Address    : 10.24.65.0/24
  Dest. Address     : 10.20.11.0/24
  Action            : ACCEPT
  NAT               : DISABLED
```

**Explicación:** Sin estas políticas el FortiGate bloquea el tráfico aunque el túnel esté activo. Es fundamental deshabilitar el NAT para que las IPs privadas de ambas LANs se comuniquen directamente sin traducción de direcciones.

---

### 📸 Captura 6 — Configuración de VPCs

> Asignación de IP estática en cada VPC para que puedan enrutar tráfico a través de su FortiGate local.

```
VPC Sitio A
════════════════════════════════════════
VPCS> ip 10.20.11.10 255.255.255.0 10.20.11.254
Checking for duplicate address...
PC1 : 10.20.11.10 255.255.255.0 gateway 10.20.11.254

VPCS> save
Saving startup configuration to startup.vpc
.  done


VPC Sitio B
════════════════════════════════════════
VPCS> ip 10.24.65.10 255.255.255.0 10.24.65.254
Checking for duplicate address...
PC1 : 10.24.65.10 255.255.255.0 gateway 10.24.65.254

VPCS> save
Saving startup configuration to startup.vpc
.  done
```

**Explicación:** Cada VPC usa el `port2` de su FortiGate local como puerta de enlace. El comando `save` persiste la configuración para que no se pierda al reiniciar el nodo en PNetLab.

---

### 📸 Captura 7 — Verificación de Conectividad (Trace-route)

> Prueba de conectividad extremo a extremo desde VPC Sitio A hacia VPC Sitio B, confirmando que el tráfico atraviesa el túnel VPN.

```
Desde VPC Sitio A --> VPC Sitio B
════════════════════════════════════════════════════════════

VPCS> trace 10.24.65.10
trace to 10.24.65.10, 8 hops max, press Ctrl+C to stop

 1   10.20.11.254    2.318 ms   1.987 ms   2.104 ms   <- FGT-SitioA (LAN gateway)
 2   10.24.65.254    8.612 ms   7.934 ms   8.201 ms   <- FGT-SitioB (VPN exit)
 3   10.24.65.10     9.445 ms   8.876 ms   9.012 ms   <- VPC Sitio B [OK]
```

```
Desde VPC Sitio B --> VPC Sitio A
════════════════════════════════════════════════════════════

VPCS> trace 10.20.11.10
trace to 10.20.11.10, 8 hops max, press Ctrl+C to stop

 1   10.24.65.254    2.201 ms   2.087 ms   2.334 ms   <- FGT-SitioB (LAN gateway)
 2   10.20.11.254    8.102 ms   7.756 ms   8.344 ms   <- FGT-SitioA (VPN exit)
 3   10.20.11.10     9.234 ms   8.901 ms   9.115 ms   <- VPC Sitio A [OK]
```

**Explicación:** El trace-route demuestra que el tráfico salta directamente de un FortiGate al otro sin mostrar las IPs WAN del ISP, lo cual confirma que el túnel VPN IPsec está activo y cifrando el tráfico de forma transparente. La comunicación entre ambas LANs privadas es exitosa.

---

## 📁 Estructura del Repositorio

```
VPN-Site-To-Site/
├── Router ISP (vIOS)/
│   └── Configuracion
├── Fortinet SitioA
├── Fortinet-SitioB
├── PCS
└── README.md
```

---

## 🛠️ Herramientas Utilizadas

| Herramienta     | Uso                                    |
|-----------------|----------------------------------------|
| PNetLab         | Plataforma de simulación de red        |
| FortiGate vVM   | Firewall / Gateway VPN IPsec           |
| Cisco vIOS      | Router ISP simulado                    |
| VPCS            | Simulación de equipos cliente          |
| GitHub          | Control de versiones y documentación   |

---

*Documentación técnica — Topología VPN Site-To-Site · Matrícula 20241165*
