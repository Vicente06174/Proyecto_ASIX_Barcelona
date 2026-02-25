# SERVIDOR FÍSICO 1: Gestión de Identidad y Red

**Hardware Disponible:**  
16 GB RAM | NVMe 256 GB | SSD 256 GB (LVM)

---

## Tabla de Servidores

| CARACTERÍSTICAS | VM 1: Servidor Empresarial (PDC) | VM 2: Monitorización (Zabbix) |
|-----------------|----------------------------------|-------------------------------|
| **FUNCIONALIDAD** | Controlador de Dominio, WDS (Instalación por red) y WSUS (Actualizaciones). | Monitorización de equipos de la organización y dispositivos de red. |
| **S.O.** | Windows Server (GUI). | GNU/Linux (Debian/Ubuntu). |
| **NÚCLEOS** | 2 vCPUs | 1 vCPU |
| **RAM** | 6 GB | 3 GB |
| **DISCOS** | NVMe: 80 GB (C:\ Sistema y AD).<br>SSD (LVM): 120 GB (D:\ Repositorio WDS e ISOs). | NVMe: 40 GB (/ sistema y base de datos).<br>SSD (LVM): 0 GB. |
| **TARJETAS DE RED** | Ethernet 2 | Ethernet 3 |
| **OBSERVACIONES** | Crítico para el arranque del sistema; las ISOs se gestionan vía iSCSI. | Clasifica equipos por categorías (clientes, servidores, etc.). |

---

## Explicación técnica del reparto de discos

### NVMe
El Controlador de Dominio requiere I/O aleatorio rápido debido a:

- NTDS.dit (base de datos AD)
- SYSVOL
- LDAP, Kerberos, replicación
- Arranque del sistema

El NVMe ofrece:

- Baja latencia  
- Alto IOPS  
- Mejor rendimiento en I/O aleatorio  

### SSD (LVM)
Se usa para:

- Imágenes WDS  
- ISOs  
- Archivos grandes  

No requieren baja latencia → ideal para SSD SATA.

---

# SERVIDOR FÍSICO 2: Aplicaciones y Backup

**Hardware total:**  
16 GB RAM | NVMe 256 GB | SSD 256 GB (LVM)

---

## Tabla de Servidores

| CARACTERÍSTICAS | VM 3: BDC (Secundario) | VM 4: Apps y Backup |
|-----------------|------------------------|----------------------|
| **FUNCIONALIDAD** | Controlador de dominio secundario redundante. | Publicación de aplicaciones (LibreOffice/Gimp) y gestión de copias. |
| **S.O.** | Windows Server (Core). | Windows Server (Apps) / Linux (Backup). |
| **NÚCLEOS** | 1 vCPU | 2 vCPUs |
| **RAM** | 4 GB | 6 GB |
| **DISCOS** | NVMe: 40 GB (C:\ Sistema). | NVMe: 100 GB (C:\ Apps y S.O.).<br>SSD (LVM): 140 GB (Z:\ Repositorio Backup Local). |
| **TARJETAS DE RED** | Ethernet 2 | Ethernet 3 |
| **OBSERVACIONES** | Siempre encendido para garantizar redundancia del dominio. | Gestiona copias locales y replicación a AWS. |

---

## Justificación del dimensionamiento

### BDC
- 1 vCPU y 4 GB RAM → suficiente para autenticación y replicación AD.

### Apps/Backup
- 2 vCPU y 6 GB RAM → necesario para publicar aplicaciones y ejecutar copias.

### Consumo total del host
- **RAM:** 4 + 6 = 10 GB / 16 GB  
- **vCPU:** 3 vCPU asignadas  
- **NVMe:** 40 + 100 = 140 GB / 256 GB  
- **SSD:** 140 GB / 256 GB  

➡️ El host no se satura y mantiene margen de seguridad.

---

# SERVIDOR FÍSICO 3: Almacenamiento Centralizado (SAN)

**Hardware total:**  
16 GB RAM | NVMe 256 GB | SSD 256 GB (LVM)

---

## Tabla de Servidores

| CARACTERÍSTICAS | VM 5: Cabina Discos (SAN) | VM 6: Datos (DFS) |
|-----------------|---------------------------|-------------------|
| **FUNCIONALIDAD** | Cabina iSCSI para toda la red. | Servidor de ficheros con DFS. |
| **S.O.** | TrueNAS (BSD). | Windows Server 2025. |
| **NÚCLEOS** | 2 vCPUs | 2 vCPUs |
| **RAM** | 5 GB | 5 GB |
| **DISCOS** | NVMe: 30 GB (/ sistema).<br>SSD: 200 GB (ZPool iSCSI). | NVMe: 60 GB (C:\ Sistema).<br>LUN iSCSI: E:\ Datos DFS. |
| **TARJETAS DE RED** | Ethernet 2 | Ethernet 3 |
| **OBSERVACIONES** | Sirve LUNs a toda la infraestructura. | Organiza carpetas corporativas mediante DFS. |

---

## VM 5 – Cabina SAN
Se asignan 2 vCPU y 5 GB de RAM porque la cabina SAN necesita recursos moderados para gestionar ZFS, iSCSI y la caché ARC, pero no ejecuta aplicaciones pesadas.

---

## VM 6 – Servidor de Datos (DFS)
Gestiona carpetas de empresa, departamentos y perfiles mediante DFS.  
Depende de la cabina SAN para su almacenamiento.

➡️ El host mantiene margen de seguridad en RAM, CPU y discos.

---

# SERVIDOR ENRACKABLE: DMZ

**Hardware:**  
Servidor físico dedicado fuera del rack principal.

---

## Tabla de Servidor

| Característica | VM 7: Web Intranet |
|----------------|---------------------|
| **Funcionalidad** | Alojamiento de la intranet en zona DMZ aislada por firewall. |
| **S.O.** | GNU/Linux |
| **Núcleos** | 2 vCPUs |
| **RAM** | 4 GB |
| **Discos** | `/` (ext4): 40 GB<br>`/var/www` (xfs): 40 GB |
| **Tarjetas de red** | Ethernet 2 (DMZ) |
| **Observaciones** | Hardening, firewall local, servicios mínimos. |
