# 🔐 VPN Site-To-Site — FortiGate con Router ISP (vIOS)

> **Matrícula:** 20241165  
> **Topología:** VPN Site-To-Site con dos FortiGates, un router ISP y dos redes LAN  
> **Plataforma:** GNS3 / EVE-NG

---

## 📋 Objetivo de la Red

Configurar una topología de **VPN Site-To-Site** entre dos firewalls FortiGate (FGT-SitioA y FGT-SitioB), interconectados a través de un router ISP simulado (Cisco vIOS), con el fin de establecer comunicación cifrada entre dos redes LAN privadas ubicadas en sitios geográficos distintos.

Los objetivos específicos son:

- Configurar interfaces WAN y LAN en ambos FortiGates
- Establecer conectividad entre los FortiGates y el router ISP
- Crear un túnel VPN IPsec Site-To-Site entre ambos sitios
- Verificar comunicación entre las redes LAN mediante `trace-route`

---

## 🗺️ Topología

<img width="1469" height="1181" alt="Captura de pantalla 2026-03-18 204033" src="https://github.com/user-attachments/assets/e1fa3045-0692-45e7-8116-2fb17098ae95" />







> La topología fue implementada en PNELAB. Cada FortiGate tiene:
> - **port1** → Enlace WAN hacia el ISP
> - **port2** → Red LAN local
> - **port3** → (Reservado / enlace adicional)

---

## 📐 Esquema de Direccionamiento IP

Basado en la matrícula **20241165**, dividida en pares: **20, 24, 11, 65**

### Redes WAN (Enlace al ISP) — /30

| Enlace              | Red             | ISP (Gateway)    | FortiGate        |
|---------------------|-----------------|------------------|------------------|
| FGT-SitioA ↔ ISP   | 172.20.11.0/30  | 172.20.11.1      | 172.20.11.2      |
| FGT-SitioB ↔ ISP   | 172.24.65.0/30  | 172.24.65.1      | 172.24.65.2      |

### Redes LAN — /24

| Sitio     | Red LAN         | Gateway (FortiGate port2) |
|-----------|-----------------|---------------------------|
| Sitio A   | 10.20.11.0/24   | 10.20.11.254              |
| Sitio B   | 10.24.65.0/24   | 10.24.65.254              |

### PCs (VPCs)

| VPC         | IP              | Gateway         |
|-------------|-----------------|-----------------|
| VPC Sitio A | 10.20.11.10/24  | 10.20.11.254    |
| VPC Sitio B | 10.24.65.10/24  | 10.24.65.254    |

---

## ⚙️ Configuraciones Utilizadas

### 🔹 Router ISP (vIOS)

```
enable
configure terminal

interface GigabitEthernet0/0
 ip address 172.20.11.1 255.255.255.252
 no shutdown
 exit

interface GigabitEthernet0/1
 ip address 172.24.65.1 255.255.255.252
 no shutdown
 exit

end
write memory
```

---

### 🔹 FortiGate — Sitio A (`FGT-SitioA`)

**Configuración básica de sistema e interfaces:**

```
config system global
    set hostname FGT-SitioA
end

config system interface
    edit port1
        set ip 172.20.11.2 255.255.255.252
        set allowaccess ping http https
    next
    edit port2
        set ip 10.20.11.254 255.255.255.0
        set allowaccess ping http https
    next
end

config router static
    edit 1
        set device port1
        set gateway 172.20.11.1
    next
end
```

---

### 🔹 FortiGate — Sitio B (`FGT-SitioB`)

**Configuración básica de sistema e interfaces:**

```
config system global
    set hostname FGT-SitioB
end

config system interface
    edit port1
        set ip 172.24.65.2 255.255.255.252
        set allowaccess ping http https
    next
    edit port2
        set ip 10.24.65.254 255.255.255.0
        set allowaccess ping http https
    next
end

config router static
    edit 1
        set device port1
        set gateway 172.24.65.1
    next
end
```

---

### 🔹 VPCs — Configuración de IP

```bash
# En la VPC conectada a Fortinet SitioA
ip 10.20.11.10 255.255.255.0 10.20.11.254
save

# En la VPC conectada a SitioB
ip 10.24.65.10 255.255.255.0 10.24.65.254
save
```

---

## 🔒 Configuración VPN IPsec Site-To-Site

La VPN fue configurada a través de la interfaz web de FortiGate (GUI) o CLI, estableciendo:

| Parámetro        | Valor                    |
|------------------|--------------------------|
| Tipo de VPN      | IPsec Site-To-Site       |
| Autenticación    | Pre-Shared Key (PSK)     |
| Peer remoto A    | 172.20.11.2              |
| Peer remoto B    | 172.24.65.2              |
| Red local A      | 10.20.11.0/24            |
| Red local B      | 10.24.65.0/24            |
| Fase 1 (IKE)     | AES256 / SHA256 / DH14   |
| Fase 2 (IPsec)   | AES256 / SHA256           |

---

## ✅ Verificación de Conectividad

La conectividad fue verificada mediante `trace-route` desde cada VPC hacia la red LAN del sitio remoto:

```bash
# Desde VPC Sitio A → LAN Sitio B
trace 10.24.65.10

# Desde VPC Sitio B → LAN Sitio A
trace 10.20.11.10
```

El tráfico atraviesa el túnel VPN cifrado, pasando por ambos FortiGates y el router ISP.

---

## 📁 Estructura del Repositorio

```
VPN-Site-To-Site/
├── Router ISP (vIOS)/
│   └── Configuracion
├── Fortinet SitioA
├── Fortinet-SitioB
└── PCS
README.md
```

---

## 🛠️ Herramientas Utilizadas

| Herramienta       | Uso                              |
|-------------------|----------------------------------|
|        EVE-NG     | Simulación de la topología       |
| FortiGate vVM     | Firewall / Gateway VPN           |
| Cisco vIOS        | Router ISP simulado              |
| VPCS              | Simulación de PCs cliente        |
| GitHub            | Control de versiones             |

---

*Documentación generada para la asignación de Redes y Seguridad — Topología VPN Site-To-Site*
