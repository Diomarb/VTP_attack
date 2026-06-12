# ⚠️ VTP Attack — Agregar y Eliminar VLANs

> **Lab de Seguridad en Redes — ITLA 2024-1185**  
> Herramienta educativa para demostrar el envenenamiento del protocolo VTP en entornos controlados (GNS3).

---

## 📋 Descripción

Este script demuestra el ataque de **VTP Domain Takeover** mediante el envío de frames VTP falsificados desde Kali Linux hacia switches Cisco IOU. Al inyectar anuncios con un número de revisión superior al actual, el atacante puede:

- ✅ **Agregar** una VLAN arbitraria a todos los switches del dominio
- ❌ **Eliminar** una VLAN existente de todos los switches del dominio

---

## 🗂️ Estructura del Repositorio

```
vtp-attack/
├── vtp_attack.py        # Script principal del ataque
├── README.md            # Este archivo
└── docs/
    └── VTP_Attack_Documentacion.docx
```

---

## ⚙️ Requisitos

| Requisito | Detalle |
|---|---|
| Sistema operativo | Kali Linux (en GNS3 o físico) |
| Python | 3.x |
| Librería | Scapy |
| Privilegios | root (sudo) |
| Topología | Switch Cisco IOU con VTP habilitado |
| Conexión | Puerto trunk 802.1Q entre Kali y el switch |

### Instalar dependencias

```bash
pip install scapy --break-system-packages
```

---

## 🚀 Uso

```bash
sudo python3 vtp_attack.py
```

Al ejecutar, el script solicita:

```
Interfaz de red (Enter = eth0):   ← nombre de tu interfaz (ip link show)
Dominio VTP     (Enter = LAB):    ← nombre exacto del dominio VTP
```

Luego detecta automáticamente la revisión VTP actual (sniff 6s) y presenta el menú:

```
┌─────────────────────────────────────┐
│  1. Agregar VLAN                    │
│  2. Eliminar VLAN                   │
│  3. Secuencia completa (add + del)  │
│  0. Salir                           │
└─────────────────────────────────────┘
```

### Opción 1 — Agregar VLAN

```
VLAN ID  (ej: 100): 100
Nombre   (ej: VENTAS): VENTAS
```

### Opción 2 — Eliminar VLAN

```
VLAN ID a eliminar (ej: 200): 200
```

---

## 🌐 Topología de Red

```
  ┌──────────┐        ┌─────────┐      ┌─────────┐
  │   KALI   │ trunk  │  IOU1   │trunk │  IOU2   │
  │  LINUX   ├────────┤ VTP     ├──────┤ VTP     │
  │ ATACANTE │        │ SERVER  │      │ CLIENT  │
  └──────────┘        └────┬────┘      └─────────┘
   10.11.85.X              │trunk
                      ┌────┴────┐
                      │  IOU3   │
                      │  VTP    │
                      │  CLIENT │
                      └─────────┘
  Red: 10.11.85.0/24  |  Dominio VTP: LAB
```

---

## 🔧 Configuración previa en switches

```
! Resetear revisión antes del ataque (en cada switch)
Switch(config)# vtp mode transparent
Switch(config)# vtp mode server    ! o client según corresponda

! Configurar dominio
Switch(config)# vtp domain LAB
Switch(config)# vtp version 2

! Verificar
Switch# show vtp status
Switch# show vlan brief
```

---

## 🔬 ¿Cómo funciona el ataque?

1. **Sniff** — Escucha anuncios VTP para detectar la revisión actual del dominio
2. **Summary Advertisement** — Envía un frame anunciando revisión `actual + 20` con MD5 calculado
3. **Subset Advertisement** — Envía las entradas de VLAN deseadas (add) o solo VLAN 1 (delete)
4. Los switches VTP Client actualizan su base de datos automáticamente al recibir mayor revisión

> **Clave técnica:** El MD5 digest es calculado correctamente sobre `dominio + revisión + entradas VLAN`. Sin este digest válido, los switches Cisco IOU rechazan los frames silenciosamente.

---

## 🛡️ Contramedidas

| Contramedida | Comando |
|---|---|
| Modo Transparent | `vtp mode transparent` |
| VTP Password | `vtp password <secret>` |
| VTP Version 3 | `vtp version 3` + `vtp primary vlan` |
| Port Security | `switchport port-security` |

---

## ⚠️ Aviso Legal

> Este script es **exclusivamente para uso educativo** en entornos de laboratorio controlados.  
> Su uso en redes de producción o sin autorización expresa es **ilegal**.  
> ITLA — Instituto Tecnológico de las Américas | Seguridad en Redes

---

## 📹 Video de Demostración

> 🎥 [Ver en YouTube](#) ← *(agregar enlace después de subir)*

---

*Desarrollado por estudiante 2024-1185 — ITLA*
