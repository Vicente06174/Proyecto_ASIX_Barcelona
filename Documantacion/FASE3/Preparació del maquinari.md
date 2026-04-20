# Implantación básica del sistema

**MATERIAL DISPONIBLE**

- 2 patch panels
- 1 MikroTik que hace de servidor de comunicaciones con el exterior
- 1 switch Cisco 1U configurable que es el switch del edificio 1
- 1 switch TP-Link 1U configurable que es el switch del edificio 2
- 1 switch TP-Link configurable de 8 puertos no enrackable que es el switch de campus
- 1 switch 1U no configurable
- 3 puntos de acceso sin hilos 
- 3 PC Dell que harán de servidores de la sede 
- 4 PC que hacen de clientes de la red  
- 5 monitores,5 teclados y 5 ratones
- 1 SAI
- 9 Adaptadores (3 x ordenador)
- Cables de red


# Distribución RACK

**Campus:**

Switch Distribución: 

-1 switch TP-Link configurable de 8 puertos no enrackable


RACK 1 (más arriba):
- 1 Patch Panel
- 1 switch Cisco
- 1 Mikrotik


RACK 2 (más abajo): 
- 1 Patch Panel
- 1 switch TP-Link
- 3 PC Dell “Srv”
- 1 SAI

---

**Edificio 1:**  

1.- Switch Cisco de 24 (Distribuidor Edificio 1):  

Puertos:  

Recepción: Puerto 1 (Configurado a VLAN 20)  

Gerencia: Puerto 4 (Configurado a VLAN 10)  

Administración: Puerto 5 (Configurado a VLAN 10)  

SAT: Puertos 8 y 9 (Configurados a VLAN 40)  

CPD: Puertos 12-20 (Configurados a VLAN 100)  

Connexión con el distribuidor de campus: Puerto 24 (Mode Trunk)  


2.- Patch Panel
Puertos:
1-12 (planta 1)    
13-24 (planta 0)

---

**Edificio 2**  

1.- Switch TP-Link de 16/24 (Distribuidor Edificio 2):  

Puertos: 
Sala vigilancia: Puerto 1 (Configurado a VLAN 50)  

Departamento comercial: Puerto 3 (Configurado a VLAN 20)  

Producción: Puerto 6 (Configurado a VLAN 30)  

Almacenamiento: Encargado de operaciones: Puerto 9 (Configurado a VLAN 30) y  Administrativo: Puerto 10 (Configurado a VLAN 30)  
 
Connexión con el distribuidor de campus: Puerto 16 (Mode Trunk)


2.- Patch Panel
Puertos:
Usaremos los mismos números que en el switch  

Switch TP-Link de 8 (Distribuidor de campus):
Connexión con el distribuidor Edificio 1: Port 1 (Mode Trunk)  

Connexión con el distribuidor Edificio 2: Port 3 (Mode Trunk)  

Connexión con Router Mikrotik: Puerto 8 (Mode Trunk)  


 Router Mikrotik:
SNAT (tràfic d'eixida): Puerto 1 (WAN)  

Troncal y cortafuegos entre VLANs: Puerto 2 (LAN)



# Sistema de Alimentación Ininterrumpida (SAI)

Se ha implementado un sistema de alimentación basado en dos unidades del modelo Phasak OnLine PH 9230 3000VA, configuradas en paralelo junto con un sistema ATS para garantizar la continuidad del servicio.
El SAI es de tipo Online (doble conversión), lo que proporciona suministro eléctrico continuo sin interrupciones.

**Configuración del SAI (conexión USB)**

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

- SERVIDOR FÍSICO 3: Almacenamiento Centralizado (SAN)
  
VM 5: Cabina de discos (SAN)  

VM 6: Datos (DFS)  


- SERVIDOR FÍSICO 2: Aplicaciones y Backup
  
VM 3: BDC (Controlador de dominio secundario)

VM 4: Aplicaciones y backup

Estos servidores contienen los datos y servicios críticos de la empresa, por lo que deben apagarse en último lugar.


### Servidores secundarios (Media prioridad)

- SERVIDOR FÍSICO 1: Gestión de Identidad y Red
  
VM 1: Servidor empresarial (PDC)  

VM 2: Monitorización (Zabbix)  


Aunque incluye servicios importantes, se ha considerado de menor prioridad para permitir la correcta parada del resto de sistemas.


### Equipos cliente (Baja prioridad)
Ordenadores de usuario y puestos de trabajo.

Son los primeros en apagarse, ya que no son críticos para la integridad del sistema.

## Configuración del apagado ante corte eléctrico
El sistema sigue un apagado escalonado controlado por el servidor con NUT:  

- Secuencia de actuación
  
- Detección del corte eléctrico por el SAI
  
- Espera de seguridad (2–3 minutos) para evitar microcortes.
  

**Fase 1: Apagado de clientes**
Apagado inmediato de equipos cliente  .

Objetivo: reducir consumo rápidamente.

**Fase 2: Servidores secundarios**
SERVIDOR FÍSICO 1 (PDC + Zabbix)
Apagado cuando: Batería al ~50% o tras 10 minutos

**Fase 3: Servidores críticos**
SERVIDOR FÍSICO 2 (Apps + Backup)  

SERVIDOR FÍSICO 3 (SAN + DFS)  

Apagado cuando: Batería al 10–20%.

**Fase 4: Servidor de gestión**
Último en apagarse. Garantiza que todas las órdenes se ejecuten correctamente

---

# Plan de prevención frente a incidencias

**Fallo de hardware (servidores)**
- Uso de RAID: Implementar niveles adecuados según criticidad: RAID 1 para redundancia básica; RAID 5/6 para equilibrio entre rendimiento y tolerancia a fallos; RAID 10 para sistemas críticos.
- Verificación periódica del estado del RAID (rebuilds, inconsistencias).
- Monitorización del estado de discos.

**Fallo de red (Mikrotik)**
- Copia de seguridad de la configuración: Backups automáticos programados, almacenamiento externo seguro (NAS o nube), versionado de configuraciones.
- Sustitución rápida del equipo: Disponer de un equipo Mikrotik de respaldo preconfigurado, documentación clara del proceso de reemplazo, pruebas periódicas de recuperación.
- Monitorización de red: Uso de herramientas como SNMP, NetFlow o Zabbix, alertas en tiempo real ante caídas o saturación, control de ancho de banda y latencias.

  
**Fallo eléctrico**
- Uso de SAI Online redundante: Dimensionamiento adecuado según consumo,c onfiguración en alta disponibilidad, monitorización del estado de baterías.
- Apagado controlado de sistemas según prioridad: Scripts automáticos de apagado por prioridad: Sistemas no críticos, Servicios secundarios, Sistemas críticos.

**Fallo de equipos cliente**
- Sustitución rápida: Stock mínimo de equipos de reemplazo, equipos preconfigurados listos para uso inmediato, procedimiento estándar de cambio.
- Configuración centralizada:Uso de Active Directory o gestión MDM, perfiles de usuario centralizados, aplicaciones desplegadas automáticamente.


# ESQUEMA DISEÑO CPD

A continuación se muestra el esquema general del diseño del CPD, incluyendo la distribución de racks, switches, servidores y el SAI, entre otros.

## 📷 Esquema visual del CPD

![Esquema CPD](https://drive.google.com/uc?export=view&id=12cYCpXAcUTs-oKG1dgZq2uS50IHCZVi0)

## 📝 Explicación del esquema

El diagrama representa la estructura física del CPD, mostrando:

- **RACK 1**: Patch panel, switch Cisco y router MikroTik como punto principal de comunicaciones.  
- **RACK 2**: Patch panel, switch TP-Link, servidores Dell y el SAI que garantiza continuidad eléctrica.  
- **Switch de Campus**: Une los edificios y distribuye la red troncal.  
- **Interconexiones Trunk**: Enlaces principales entre edificios y el router.  
- **Distribución por VLAN**: Cada área de la empresa está segmentada para mejorar seguridad y rendimiento.

## 🗂️ Leyenda del esquema

| Elemento | Descripción |
|---------|-------------|
| **PP** | Patch Panel |
| **SW Cisco** | Switch principal del Edificio 1 |
| **SW TP-Link** | Switch del Edificio 2 y Campus |
| **MikroTik** | Router y firewall entre VLANs |
| **Srv Dell** | Servidores físicos con máquinas virtuales |
| **SAI** | Sistema de alimentación ininterrumpida |
| **Trunk** | Enlace de transporte de VLANs |
| **VLAN X** | Segmentación lógica de red |

---


