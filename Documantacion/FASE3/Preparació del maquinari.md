# Implantación básica del sistema

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
El Controlador de Dominio requiere I/O aleatorio rápido debido a






# Distribución RACK










# Sistema de Alimentación Ininterrumpida (SAI)

Se ha implementado un sistema de alimentación basado en dos unidades del modelo Phasak OnLine PH 9230 3000VA, configuradas en paralelo junto con un sistema ATS para garantizar la continuidad del servicio.
El SAI es de tipo Online (doble conversión), lo que proporciona suministro eléctrico continuo sin interrupciones.
Configuración del SAI (conexión USB)

El sistema se ha configurado mediante conexión USB directa a un servidor de gestión.
Se utiliza el software NUT (Network UPS Tools) para:
Monitorizar el estado del SAI (batería, carga, eventos)
Detectar cortes eléctricos
Ejecutar apagados automáticos
Enviar órdenes al resto de servidores por red

## Clasificación de servidores según importancia

Se han definido dos niveles de prioridad basados en la función de cada servidor:

### Servidores críticos (Alta prioridad)
Son los sistemas que deben mantenerse activos el mayor tiempo posible:

SERVIDOR FÍSICO 3: Almacenamiento Centralizado (SAN)
VM 5: Cabina de discos (SAN)
VM 6: Datos (DFS)

SERVIDOR FÍSICO 2: Aplicaciones y Backup
VM 3: BDC (Controlador de dominio secundario)
VM 4: Aplicaciones y backup

Estos servidores contienen los datos y servicios críticos de la empresa, por lo que deben apagarse en último lugar.


### Servidores secundarios (Media prioridad)

- SERVIDOR FÍSICO 1: Gestión de Identidad y Red
VM 1: Servidor empresarial (PDC)
VM 2: Monitorización (Zabbix)

Aunque incluye servicios importantes, se ha considerado de menor prioridad para permitir la correcta parada del resto de sistemas.


### Equipos cliente (Baja prioridad)
Ordenadores de usuario
Puestos de trabajo
Son los primeros en apagarse, ya que no son críticos para la integridad del sistema.

## Configuración del apagado ante corte eléctrico
El sistema sigue un apagado escalonado controlado por el servidor con NUT:
Secuencia de actuación
Detección del corte eléctrico por el SAI
Espera de seguridad (2–3 minutos) para evitar microcortes

- Fase 1: Apagado de clientes
Apagado inmediato de equipos cliente
Objetivo: reducir consumo rápidamente

- Fase 2: Servidores secundarios
SERVIDOR FÍSICO 1 (PDC + Zabbix)
Apagado cuando:
Batería al ~50%
o tras 10 minutos

- Fase 3: Servidores críticos
SERVIDOR FÍSICO 2 (Apps + Backup)
SERVIDOR FÍSICO 3 (SAN + DFS)
Apagado cuando:
Batería al 10–20%

- Fase 4: Servidor de gestión
Último en apagarse
Garantiza que todas las órdenes se ejecuten correctamente









# Plan de recuperación frente a incidencias
