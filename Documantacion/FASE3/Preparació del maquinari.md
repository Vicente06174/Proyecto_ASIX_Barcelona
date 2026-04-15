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
-1 Patch Panel

-1 switch Cisco

-1 MikroTik 


RACK 2 (más abajo): 
-1 Patch Panel 

-1 switch TP-Link

-3 PC Dell “Srv”

-1 SAI

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
1-12 (planta 1)   13-24 (planta 0)

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

# Plan de recuperación frente a incidencias

El objetivo de este plan es prevenir y minimizar el impacto de incidencias que puedan afectar a los servidores, equipos de red (Mikrotik) y equipos cliente, garantizando la continuidad del servicio.

Protección contra incendios
Se implementan medidas para proteger la infraestructura ante incendios:
Medidas preventivas
Instalación de detectores ópticos de humo en la sala de servidores
Integración con el sistema de monitorización
Notificación automática al responsable
Actuación
Uso de extintores de CO₂ o agente limpio
Intervención manual en caso de incidente

Seguridad física
El acceso a la zona donde se ubican los servidores y equipos de red está restringido mediante un sistema de control de acceso.
Medidas implementadas:
Acceso mediante tarjetas RFID
Cerradura electrónica controlada
Registro de accesos (entrada/salida del personal autorizado)
Ubicación en una zona separada de los puestos de trabajo
Este sistema permite limitar el acceso únicamente al personal autorizado, evitando manipulaciones indebidas de los servidores o del equipo de red (Mikrotik)

Copias de seguridad
Con el objetivo de prevenir la pérdida de información y garantizar la continuidad del servicio, se establecen las siguientes medidas de copias de seguridad:
Medidas preventivas
Se realizan copias de seguridad automáticas de todas las máquinas virtuales y servidores.
Se mantiene más de una copia de los datos para evitar pérdidas por fallos.
Las copias se almacenan en diferentes soportes (almacenamiento en red y externo).
Se dispone de al menos una copia fuera de la empresa para prevenir desastres físicos.
Sistema de copias
Posibilidad de recuperación completa o parcial (archivos, máquinas virtuales).
Monitorización del estado de las copias y notificación de errores.
Planificación

Copias incrementales diarias de lunes a viernes.
Copias completas durante el fin de semana.
Almacenamiento
Copia local en NAS para recuperación rápida.
Copia externa almacenada fuera de la empresa (disco externo o cintas).


Verificación
Revisión periódica del estado de las copias.
Pruebas de restauración para comprobar su correcto funcionamiento.

