# Guía de despliegue del laboratorio

Esta guía documenta cómo reproducir el laboratorio sobre el que se ejecuta y valida el pipeline de Detection-as-Code. Es una versión orientada a comandos y configuración, no a capturas de pantalla — para el registro visual completo del despliegue, ver la documentación técnica del laboratorio.

## Topología de red

```
                Proxmox VE — Red interna 192.168.100.0/24

  ┌────────────┐   ┌─────────────────┐   ┌─────────────┐
  │  pfSense   │   │  Ubuntu Server  │   │    Wazuh    │
  │  Firewall  │   │  DHCP + DNS     │   │    SIEM     │
  │ .1 / .2    │   │  192.168.100.1  │   │ .100.113    │
  └────────────┘   └─────────────────┘   └─────────────┘

  ┌──────────────────────┐   ┌──────────────────────────┐
  │  Windows Server 2022 │   │   Windows 10 Pro Client  │
  │  Active Directory DC │   │   Unido a red.local      │
  │  192.168.100.3       │   │   DHCP (.10 – .50)       │
  └──────────────────────┘   └──────────────────────────┘
```

| VM | Rol | IP |
|---|---|---|
| pfSense | Firewall / Gateway | 192.168.100.1 (LAN) / 192.168.100.2 (WAN) |
| ubuntu-server | DHCP + DNS + Wazuh Agent | 192.168.100.1 |
| wazuh | SIEM (Docker) | 192.168.100.113 |
| win-server | Active Directory DC | 192.168.100.3 |
| win-client | Endpoint de usuario | DHCP (rango .10 – .50) |

Todo se despliega sobre **Proxmox VE**, dominio `red.local`.

---

## 1. pfSense — firewall perimetral

1. Asignar interfaces WAN/LAN durante el arranque inicial desde la consola.
2. Configurar IP estática de LAN para poder acceder al WebGUI.
3. Completar el **Setup Wizard** (9 pasos):
   - Hostname: `pfSense`, dominio: `red.local`, DNS primario: `192.168.100.1`
   - WAN: IP estática `192.168.100.2/24`, gateway `192.168.100.1`
   - LAN: IP estática `192.168.100.1/24`
   - Contraseña de administrador del WebGUI

### Reglas de firewall (interfaz WAN)

Política base: **denegación implícita**. Solo se permite tráfico explícitamente necesario, con logging activado en todas las reglas para su correlación posterior en Wazuh.

| Regla | Protocolo | Origen | Destino | Puerto | Acción |
|---|---|---|---|---|---|
| DNS | TCP/UDP | 192.168.100.0/24 | 192.168.100.1 | 53 | Pass |
| DHCP | TCP/UDP | 192.168.100.0/24 | 192.168.100.1/24 | 67 | Pass |
| SSH | TCP | 192.168.100.0/24 | Any | 22 | Pass |
| Wazuh (agentes) | TCP | 192.168.100.0/24 | 192.168.100.113 | 1514 | Pass |
| Denegación por defecto | TCP | Any | Any | Any | Block |

> La regla de denegación por defecto debe quedar siempre al final del listado; pfSense evalúa las reglas en orden y aplica la primera que coincide.

---

## 2. Ubuntu Server — DHCP y DNS

### Instalación base
Ubuntu Server 24.04 LTS, hostname `ubuntu-server`, usuario `administrador` con sudo, OpenSSH habilitado con autenticación por contraseña.

### Red — doble interfaz
- `ens18`: gestión, DHCP del host Proxmox (o estática según entorno)
- `ens19`: red interna del laboratorio, bridge `vmbr1`, adaptador VirtIO

Configurar `/etc/netplan/50-cloud-init.yaml`:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses: [192.168.1.200/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    ens19:
      addresses: [192.168.100.1/24]
```

Aplicar y verificar:
```bash
sudo netplan apply
ip a
```

### Servidor DHCP (isc-dhcp-server)

```bash
sudo apt install isc-dhcp-server
```

En `/etc/default/isc-dhcp-server`:
```
INTERFACESv4="ens19"
INTERFACESv6=""
```

En `/etc/dhcp/dhcpd.conf`:
```
authoritative;

subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.10 192.168.100.50;
  option routers 192.168.100.1;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  default-lease-time 602;
  max-lease-time 7200;
}
```

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

### Servidor DNS (BIND9)

```bash
sudo apt install bind9 bind9-doc bind9utils dns-root-data
```

En `/etc/bind/named.conf.local`:
```
zone "red.local" {
  type master;
  file "/etc/bind/db.red.local";
};

zone "100.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/db.192.168.100";
};
```

Zona directa (`/etc/bind/db.red.local`, basada en `db.local`):
```
$TTL    604800
@       IN      SOA     ns.red.local. admin.red.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
@       IN      NS      localhost.
ns      IN      A       192.168.100.1
server  IN      A       192.168.100.1
```

Zona inversa (`/etc/bind/db.192.168.100`, basada en `db.127`):
```
$TTL    604800
@       IN      SOA     ns.red.local. admin.red.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
@       IN      NS      ns.red.local.
1       IN      PTR     ns.red.local.
```

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

---

## 3. Wazuh — SIEM

### Instalación base
Ubuntu Server 24.04 LTS, hostname `wazuh`, usuario `administrador` con sudo, OpenSSH habilitado. Esta VM requiere más CPU/RAM que el resto del laboratorio, ya que corre el stack completo de Wazuh en Docker y procesa eventos en tiempo real.

### Despliegue con Docker Compose

```bash
sudo apt install docker-compose
```

`docker-compose.yml`:
```yaml
services:
  wazuh:
    image: ghcr.io/wazuh/wazuh:latest
    container_name: wazuh
    restart: always
    ports:
      - "443:443"
      - "1514:1514/udp"
      - "1515:1515"
    environment:
      - INDEXER_USERNAME=admin
      - INDEXER_PASSWORD=<password_segura>
```

```bash
docker-compose up -d
```

El panel queda accesible en `https://192.168.100.113`.

### Registro de agentes

Desde el WebGUI → **Deploy new agent**, se genera un comando de instalación personalizado por plataforma.

**Ubuntu-server** (Linux DEB amd64, manager `192.168.100.1`):
```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.5-1_amd64.deb
sudo WAZUH_MANAGER='192.168.100.1' WAZUH_AGENT_NAME='ubuntu-server' dpkg -i ./wazuh-agent_4.14.5-1_amd64.deb

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo systemctl status wazuh-agent
```

**Windows-AD** (MSI, manager `192.168.100.3`, PowerShell como Administrador):
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi -OutFile wazuh-agent.msi
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER='192.168.100.3' WAZUH_AGENT_NAME='Windows-AD'

NET START Wazuh
```

---

## 4. Windows Server 2022 — Active Directory

### Instalación del rol AD DS

Desde Server Manager → **Agregar roles y características**, seleccionar:
- Active Directory Domain Services
- Group Policy Management
- Remote Server Administration Tools (AD DS and AD LDS Tools, Active Directory Administrative Center, AD DS Snap-Ins and Command-Line Tools)

### Promoción a controlador de dominio

Asistente de configuración post-despliegue:
- **Agregar un nuevo bosque** → nombre de dominio raíz: `red.local`
- Nivel funcional de bosque y dominio: **Windows Server 2016**
- Activar **Servidor DNS** y **Catálogo Global (GC)**
- No configurar como RODC
- Definir contraseña de modo DSRM
- NetBIOS derivado automáticamente: `RED`
- Rutas por defecto: NTDS y SYSVOL en `C:\Windows\NTDS` / `C:\Windows\SYSVOL`

El servidor reinicia al completar la promoción, quedando activo como DC con AD DS, DNS y Kerberos operativos.

### Creación de usuarios

Desde **Active Directory Users and Computers** → `red.local/Users` → `New Object - User`:
- Nombre: `usuario`, apellido `1`
- UPN: `u1@red.local`
- Login pre-Windows 2000: `RED\u1`
- Marcar **"El usuario debe cambiar la contraseña en el próximo inicio de sesión"**

### Política de contraseñas (GPO)

En **Default Domain Policy** → `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`:

| Parámetro | Valor |
|---|---|
| Historial de contraseñas | 5 |
| Vigencia máxima | 42 días |
| Vigencia mínima | 1 día |
| Longitud mínima | 12 caracteres |
| Complejidad | Habilitada |
| Cifrado reversible | Deshabilitado |

---

## 5. Windows 10 Pro — cliente del dominio

1. Instalar Windows 10 Pro (edición Pro es obligatoria para poder unirse a un dominio).
2. Verificar conectividad y asignación DHCP:
   ```powershell
   ipconfig
   ```
   Debe recibir IP en el rango `192.168.100.10 – .50` con gateway `192.168.100.1`.
3. Unir al dominio: **Configuración → Sistema → Acerca de → Cambiar nombre del equipo o el dominio**
   - Nombre del equipo: `equipo1`
   - Dominio: `red.local`
   - Introducir credenciales con permisos para añadir equipos al dominio
4. Reiniciar.
5. Iniciar sesión como `u1@RED` (se solicitará cambio de contraseña, confirmando que la GPO se aplica).
6. Forzar aplicación de políticas:
   ```powershell
   gpupdate /force
   ```

---

## 6. Verificación del laboratorio

| Componente | Verificación | Comando |
|---|---|---|
| pfSense | Reglas WAN aplicadas, denegación por defecto activa | — (WebGUI) |
| ubuntu-server | DHCP activo | `sudo systemctl status isc-dhcp-server` |
| ubuntu-server | DNS activo | `sudo systemctl status bind9` |
| ubuntu-server | Agente Wazuh activo | `sudo systemctl status wazuh-agent` |
| wazuh | Contenedor Docker activo | `docker ps` |
| wazuh | WebGUI accesible | `https://192.168.100.113` |
| win-server | DC operativo, DC de `red.local` | `nltest /dsgetdc:red.local` |
| win-client | Unido al dominio, GPO aplicadas | `gpupdate /force`, `ipconfig` |

Checklist final esperado:

- [x] pfSense: denegación por defecto + reglas explícitas de paso
- [x] ubuntu-server: DHCP y DNS resolviendo `red.local`
- [x] wazuh: stack Docker activo (443, 1514, 1515)
- [x] win-server: AD DS promovido, GPO de contraseñas aplicada
- [x] win-client: unido al dominio, login como `u1@RED` exitoso
- [x] Ambos agentes (`Ubuntu-server`, `Windows-AD`) registrados en Wazuh

Con este checklist en verde, el laboratorio está listo para la Fase 1 (reglas Sigma) y la Fase 2 (emulación de adversario con Atomic Red Team).
