# MPLS L3VPN - Automatización de Red

Automatización y validación de una red **MPLS L3VPN** (IPv4/IPv6) desplegada sobre **containerlab**, usando **Ansible** para la configuración de los dispositivos y **Python** para tareas de operación complementarias.

## Topología de red

![Topologia](docs/Topologia%20pc3.png)

La red está compuesta por los siguientes roles, todos desplegados como contenedores `clab-MPLS-*` mediante containerlab:

| Grupo Ansible | Dispositivos | Rol |
|---|---|---|
| `PE_routers` | PE1, PE2 | Provider Edge - frontera con clientes, MP-BGP VPNv4/VPNv6 |
| `P_routers` | P1, P2 | Provider - core MPLS, conmutación de etiquetas |
| `RR_routers` | RR1, RR2 | Route Reflectors - distribución de rutas MP-BGP |
| `CPEs` | CPE-1, CPE-2 | Customer Premises Equipment - equipos de cliente |
| `Switches` | SW-1, SW-2 | Conectividad de acceso |

## Estructura del proyecto

```
grupo3/
├── ansible.cfg              # Configuración de Ansible (interpreter, host key checking, etc.)
├── inventory.ini            # Inventario de dispositivos y grupos
├── generar_inventario.py    # Script Python - genera inventario CSV/TXT vía Ansible
├── ScriptParte5.py          # Script Python adicional - auditoría de configuraciones (netmiko)
│
├── host_vars/                  # Variables específicas por dispositivo
│   ├── clab-MPLS-CPE-1.yml
│   ├── clab-MPLS-CPE-2.yml
│   ├── clab-MPLS-P1.yml
│   ├── clab-MPLS-P2.yml
│   ├── clab-MPLS-PE1.yml
│   ├── clab-MPLS-PE2.yml
│   ├── clab-MPLS-RR1.yml
│   ├── clab-MPLS-RR2.yml
│   ├── clab-MPLS-SW-1.yml
│   └── clab-MPLS-SW-2.yml
│
├── playbooks/                  # Playbooks de configuración y validación
│   ├── interfaces.yml          # Configuración de interfaces
│   ├── ospf.yml                # Configuración IGP (OSPF)
│   ├── mpls.yml                # Configuración MPLS
│   ├── bgp.yml                 # Configuración MP-BGP
│   ├── vpn.yml                 # Configuración VPNv4 / VPNv6
│   ├── validate.yml            # Validación del estado de la red
│   └── gather_facts.yml        # Recolección de inventario (cisco.ios.ios_facts)
│
├── reportes/                # Salida de generar_inventario.py y ScriptParte5.py (CSV/TXT/JSON)
│
└── grupo3_napalm/           # Validación del estado de la red con NAPALM
    ├── inventario.py            # Inventario de dispositivos + helpers de conexión NAPALM
    ├── audit_facts.py           # get_facts() - información general del dispositivo
    ├── audit_interfaces.py      # get_interfaces() - estado de interfaces
    ├── audit_interfaces_ip.py   # get_interfaces_ip() - direccionamiento IP
    ├── audit_routes.py          # get_route_to() - tabla de rutas
    ├── audit_bgp.py             # get_bgp_neighbors() - vecinos BGP
    ├── audit_environment.py     # get_environment() - estado de hardware
    ├── check_ospf_status.py     # Validación de vecindades OSPF
    ├── check_bgp_status.py      # Validación de sesiones BGP
    ├── check_ipv4_vpn.py        # Validación de conectividad VPNv4 extremo a extremo
    └── check_ipv6_vpn.py        # Validación de conectividad VPNv6 extremo a extremo
```


## Requisitos previos

- [containerlab](https://containerlab.dev/) con el laboratorio MPLS desplegado y los contenedores `clab-MPLS-*` corriendo.
- Python 3.10+
- Ansible:
  ```bash
  pip install ansible-core
  ansible-galaxy collection install cisco.ios ansible.netcommon
  ```
- Para el auditor de configuraciones (`ScriptParte5.py`):
  ```bash
  pip3 install --user --upgrade netmiko
  ```
- Para la validación con NAPALM (`grupo3_napalm/`), se recomienda un entorno virtual dedicado:
  ```bash
  python3 -m venv venv
  source venv/bin/activate
  pip install napalm netmiko junos-eznc pyparsing==2.4.7 tabulate
  ```

## Automatización con Ansible

Los playbooks viven en `playbooks/` y se ejecutan con `ansible-playbook -i inventory.ini playbooks/<playbook>.yml`.

| Playbook | Función |
|---|---|
| `interfaces.yml` | Configuración de interfaces de los dispositivos |
| `ospf.yml` | Configuración del IGP (OSPF) |
| `mpls.yml` | Configuración MPLS (label switching) |
| `bgp.yml` | Configuración MP-BGP |
| `vpn.yml` | Configuración de VPNv4 y VPNv6 |
| `validate.yml` | Validación del estado operativo posterior a la configuración |
| `gather_facts.yml` | Recolecta hardware/software de cada dispositivo con `cisco.ios.ios_facts` (usado por `generar_inventario.py`) |

Ejemplo de ejecución:

```bash
ansible-playbook -i inventory.ini playbooks/interfaces.yml
```

## Script Python adicional

**`ScriptParte5.py`** — Auditor de configuraciones: se conecta vía SSH (con **netmiko**) a cada dispositivo de la red, descarga su `show running-config`, y busca patrones específicos (interfaces loopback, procesos OSPF/BGP, VRFs, address-family VPNv4/VPNv6, route distinguishers, route targets, etc.) usando expresiones regulares. A partir de esos patrones infiere el rol de cada dispositivo (PE, P, RR) y genera reportes en **TXT**, **CSV** y **JSON**.

### Uso

```bash
python3 ScriptParte5.py
```

No requiere argumentos: recorre todos los dispositivos definidos en la lista `DEVICES` dentro del propio script (espejo del `inventory.ini` de Ansible) y genera los tres reportes en `reportes/` con timestamp:

```
reportes/auditoria_<timestamp>.txt
reportes/auditoria_<timestamp>.csv
reportes/auditoria_<timestamp>.json
```

### Qué reporta

- **Rol inferido** del dispositivo (PE, P, RR) según su configuración.
- Cantidad de interfaces totales y loopbacks.
- VRFs configurados (con listado de ejemplos).
- Si MPLS y LDP están habilitados.
- Proceso y redes OSPF.
- Proceso BGP y vecinos (con listado de ejemplos).
- Address-family VPNv4 / VPNv6.
- Route distinguishers y route targets.
- Resumen global: dispositivos auditados, exitosos, con error, totales agregados de VRFs y vecinos BGP en toda la red.

Si un dispositivo no responde (timeout o fallo de autenticación), se registra el error y la auditoría continúa con el resto.

## Validación con NAPALM

La carpeta `grupo3_napalm/` contiene los scripts de validación del estado operativo de la red usando **NAPALM**, organizados en dos grupos:

- **`audit_*.py`** — uno por cada método NAPALM requerido, recorren todos los dispositivos del inventario y muestran los resultados en tablas (`tabulate`):
  - `audit_facts.py` → `get_facts()`
  - `audit_interfaces.py` → `get_interfaces()`
  - `audit_interfaces_ip.py` → `get_interfaces_ip()`
  - `audit_routes.py` → `get_route_to()`
  - `audit_bgp.py` → `get_bgp_neighbors()`
  - `audit_environment.py` → `get_environment()`

- **`check_*.py`** — validaciones puntuales del estado de los servicios:
  - `check_ospf_status.py` → vecindades OSPF en estado FULL
  - `check_bgp_status.py` → sesiones BGP establecidas
  - `check_ipv4_vpn.py` → conectividad VPNv4 extremo a extremo
  - `check_ipv6_vpn.py` → conectividad VPNv6 extremo a extremo

Todos comparten `inventario.py`, que centraliza la lista de dispositivos y los helpers de conexión NAPALM (sesión por dispositivo y espera de disponibilidad SSH antes de conectar).

### Uso

```bash
cd grupo3_napalm
source ../venv/bin/activate   # si se uso un entorno virtual dedicado

python3 audit_facts.py
```

### Ejemplo: `audit_routes.py`

Verifica `get_route_to()` contra un conjunto de destinos definidos por rol (`DESTINATIONS`), y valida que el protocolo de cada ruta encontrada sea el esperado para ese tipo de nodo (`PROTO_ESPERADO`) — por ejemplo, un P solo debería tener rutas OSPF, mientras que un PE puede tener OSPF y BGP. Cada ruta se marca como `OK` o se señala si el protocolo no es el esperado, y el resultado se imprime en una tabla con `tabulate`.

## Utilidad complementaria: `generar_inventario.py`

Genera un inventario de hardware/software de la red ejecutando `playbooks/gather_facts.yml` (vía `cisco.ios.ios_facts`) y exportando el resultado a CSV o TXT:

```bash
python3 generar_inventario.py --formato csv
```
