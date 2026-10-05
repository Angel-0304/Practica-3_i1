# Practica-3_i1
Infraestructura 1


# Laboratorio DMZ con FortiGate

**Angel Alcántara — Matrícula 2024-2356 — ITLA**

Link del video: https://itlaedudo-my.sharepoint.com/:f:/g/personal/20242356_itla_edu_do/IgBIrjfN_D2pRaQ0YzuoF4zeAZHWqsh6jn8hGo0eqXDlMN4?e=uVx1wa

## Descripción

Red empresarial con un FortiGate que separa a los usuarios de los servidores. Los servidores (Sistema de Caja, Sistema de Inventario y Base de Datos) están en una DMZ, y los usuarios están divididos en dos VLAN con permisos distintos:

- La **VLAN 20** puede entrar a las dos páginas y administrar los servidores por SSH.
- La **VLAN 10** solo puede entrar al Sistema de Caja. Si intenta abrir el Sistema de Inventario, el FortiGate le muestra una página de bloqueo.
- La **DMZ** no puede iniciar conexiones hacia los usuarios.

Toda la configuración del FortiGate se hizo por GUI, salvo el acceso de gestión inicial y el ajuste de MTU, que en FortiOS 6.4 solo existe por CLI.

## Topología

![Topología PNETLab](diagramas/topologia-pnetlab.png)

![Diagrama lógico](diagramas/diagrama-logico.png)

## Equipos

| Equipo | Imagen | Función |
|---|---|---|
| ISP | Cisco IOL | Simula Internet (Lo0 8.8.8.8) |
| FG-Empresa | FortiGate VM64-KVM v6.4.0 | Firewall, VLANs, DHCP, NAT y Web Filter |
| SW-Usuarios | Cisco IOL L2 | VLANs, trunk y seguridad de puertos |
| PC VLAN 10 / PC VLAN 20 | Docker pnetlab/ubuntu_sv | Usuarios |
| Web Caja / Inventario | Ubuntu Server 20.04 | Apache con HTTPS |
| DB server | Ubuntu Server 20.04 | MariaDB |

## Direccionamiento

| Segmento | Red | Gateway | Notas |
|---|---|---|---|
| WAN | 20.24.23.0/30 | 20.24.23.1 (ISP) | FG-Empresa port1 .2 |
| VLAN 10 | 10.23.56.0/25 | 10.23.56.1 | DHCP .10 – .100 |
| VLAN 20 | 10.23.57.0/25 | 10.23.57.1 | DHCP .10 – .100 |
| DMZ (VLAN 30) | 10.23.56.128/28 | 10.23.56.129 | Caja .130, Inventario .131, DB .132 |
| Gestión | 192.168.128.0/24 | — | FG-Empresa port3 .140 |

## Políticas del FortiGate

| # | Nombre | Origen → Destino | Servicio | Detalle |
|---|---|---|---|---|
| 1 | Usuarios-a-Internet | VLAN10, VLAN20 → WAN | ALL | NAT |
| 2 | V10-a-Inventario | VLAN10 → Web-Inventario | HTTP | Proxy + Web Filter: página de bloqueo |
| 3 | V20-a-DMZ | VLAN20 → DMZ | HTTPS, SSH | Única con SSH |
| 4 | V10-a-Caja | VLAN10 → Web-Caja | HTTPS | — |
| — | Implicit Deny | any → any | ALL | Con log activado |

## Seguridad del switch

- VLANs separadas para usuarios (10, 20) y servidores (30).
- Trunk con VLAN nativa 99, solo las VLAN necesarias y DTP apagado.
- Port-security con MAC sticky, máximo 1 MAC y modo restrict.
- BPDU Guard y PortFast en los puertos de acceso.
- DHCP snooping, con el trunk como único puerto confiable.
- Puertos sin uso apagados en la VLAN 999.
- Contraseñas de enable y consola, cifradas con `service password-encryption`.

## Resultados de las pruebas

| Prueba | VLAN 10 | VLAN 20 |
|---|---|---|
| HTTPS al Sistema de Caja | ✅ | ✅ |
| Sistema de Inventario | 🚫 Página de bloqueo | ✅ |
| SSH a los servidores | ❌ | ✅ |
| Puerto 3306 del DB | ❌ | ❌ |
| Ping entre VLAN 10 y 20 | ❌ | ❌ |
| DMZ → usuarios | ❌ Bloqueado (Implicit Deny) | ❌ Bloqueado (Implicit Deny) |

## Problemas encontrados

| Problema | Causa | Solución |
|---|---|---|
| Un PC y los servidores sin red al conectarse | El sticky aprendió MACs temporales que PNETLab usa al arrancar el nodo | Reinicio del port-security en los puertos |
| HTTPS se quedaba en el *Client hello* | Paquetes de 1500 bytes perdidos en el trunk por la etiqueta 802.1Q | MTU 1400 en los servidores y en el port2 del FortiGate |
| "Maximum number of entries has been reached" | La licencia de evaluación permite solo 5 políticas | Se combinaron políticas y se usó la Implicit Deny con log |
| La deep-inspection no descifraba el HTTPS | La licencia de evaluación funciona con cifrado bajo (LENC) | La página de bloqueo se muestra sobre HTTP |
| netplan rechazaba el MTU | La consola de PNETLab se comía espacios y la sangría del YAML quedaba mal | `sed` que copia la sangría de la línea `addresses` |

## Running-configs

<details>
<summary><b>ISP</b></summary>

```
hostname ISP
!
no ip domain lookup
ip cef
no ipv6 cef
!
interface Loopback0
 description Simula Internet
 ip address 8.8.8.8 255.255.255.255
!
interface Ethernet0/0
 description Enlace hacia FG-Empresa port1
 ip address 20.24.23.1 255.255.255.252
!
interface Ethernet0/1
 description No usado
 no ip address
 shutdown
!
interface Ethernet0/2
 no ip address
 shutdown
!
interface Ethernet0/3
 no ip address
 shutdown
!
banner motd ^CISP - Laboratorio DMZ - Matricula 2024-2356^C
!
line con 0
 exec-timeout 0 0
 logging synchronous
line vty 0 4
 login
 transport input none
!
end
```
</details>

<details>
<summary><b>SW-Usuarios</b></summary>

```
service password-encryption
!
hostname SW-Usuarios
!
enable secret 5 <ENABLE-SECRET>
!
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option
ip dhcp snooping
no ip domain-lookup
!
spanning-tree mode rapid-pvst
spanning-tree extend system-id
!
interface Ethernet0/0
 description Trunk hacia FG-Empresa port2
 switchport trunk allowed vlan 10,20,30
 switchport trunk encapsulation dot1q
 switchport trunk native vlan 99
 switchport mode trunk
 switchport nonegotiate
 ip dhcp snooping trust
!
interface Ethernet0/1
 description PC VLAN 10
 switchport access vlan 10
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5000.0040.0001
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet0/2
 description PC VLAN 20
 switchport access vlan 20
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5000.003d.0001
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet0/3
 description Web Caja (DMZ)
 switchport access vlan 30
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 50c7.f400.3e00
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet1/0
 description Inventario (DMZ)
 switchport access vlan 30
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 506c.da00.4100
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet1/1
 description DB server (DMZ)
 switchport access vlan 30
 switchport mode access
 switchport nonegotiate
 switchport port-security violation restrict
 switchport port-security mac-address sticky
 switchport port-security mac-address sticky 5082.6600.4200
 switchport port-security
 spanning-tree portfast edge
 spanning-tree bpduguard enable
!
interface Ethernet1/2
 description No usado
 switchport access vlan 999
 switchport mode access
 shutdown
!
interface Ethernet1/3
 description No usado
 switchport access vlan 999
 switchport mode access
 shutdown
!
interface Ethernet2/0
 description No usado
 switchport access vlan 999
 switchport mode access
 shutdown
!
interface Ethernet2/1
 description No usado
 switchport access vlan 999
 switchport mode access
 shutdown
!
interface Ethernet2/2
 description No usado
 switchport access vlan 999
 switchport mode access
 shutdown
!
interface Ethernet2/3
 description No usado
 switchport access vlan 999
 switchport mode access
 shutdown
!
banner motd ^CSW-Usuarios - Laboratorio DMZ - Matricula 2024-2356 - Acceso restringido^C
!
line con 0
 password 7 <CONSOLE-PASSWORD>
 logging synchronous
 login
!
end
```
</details>

<details>
<summary><b>FG-Empresa (secciones relevantes)</b></summary>

```
config system global
    set hostname "FG-Empresa"
end
config system interface
    edit "port1"
        set ip 20.24.23.2 255.255.255.252
        set allowaccess ping
        set alias "WAN"
        set role wan
    next
    edit "port2"
        set mtu-override enable
        set mtu 1400
    next
    edit "port3"
        set ip 192.168.128.140 255.255.255.0
        set allowaccess ping https ssh http
    next
    edit "VLAN10"
        set ip 10.23.56.1 255.255.255.128
        set allowaccess ping
        set alias "USUARIOS-V10"
        set role lan
        set interface "port2"
        set vlanid 10
    next
    edit "VLAN20"
        set ip 10.23.57.1 255.255.255.128
        set allowaccess ping
        set alias "USUARIOS-V20"
        set role lan
        set interface "port2"
        set vlanid 20
    next
    edit "DMZ"
        set ip 10.23.56.129 255.255.255.240
        set allowaccess ping
        set alias "DMZ"
        set role dmz
        set interface "port2"
        set vlanid 30
    next
end
config system dhcp server
    edit 2
        set default-gateway 10.23.56.1
        set netmask 255.255.255.128
        set interface "VLAN10"
        config ip-range
            edit 1
                set start-ip 10.23.56.10
                set end-ip 10.23.56.100
            next
        end
        set dns-server1 8.8.8.8
    next
    edit 3
        set default-gateway 10.23.57.1
        set netmask 255.255.255.128
        set interface "VLAN20"
        config ip-range
            edit 1
                set start-ip 10.23.57.10
                set end-ip 10.23.57.100
            next
        end
        set dns-server1 8.8.8.8
    next
end
config router static
    edit 1
        set gateway 20.24.23.1
        set device "port1"
    next
end
config firewall address
    edit "Web-Caja"
        set subnet 10.23.56.130 255.255.255.255
    next
    edit "Web-Inventario"
        set subnet 10.23.56.131 255.255.255.255
    next
    edit "DB-Server"
        set subnet 10.23.56.132 255.255.255.255
    next
end
config webfilter urlfilter
    edit 1
        set name "Auto-webfilter-urlfilter_jxxnijgvf"
        config entries
            edit 1
                set url "10.23.56.131"
                set action block
            next
        end
    next
end
config webfilter profile
    edit "Bloqueo-Inventario"
        set feature-set proxy
        config web
            set urlfilter-table 1
        end
    next
end
config firewall policy
    edit 1
        set name "Usuarios-a-Internet"
        set srcintf "VLAN10" "VLAN20"
        set dstintf "port1"
        set srcaddr "VLAN10 address" "VLAN20 address"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
    edit 2
        set name "V10-a-Inventario"
        set srcintf "VLAN10"
        set dstintf "DMZ"
        set srcaddr "VLAN10 address"
        set dstaddr "Web-Inventario"
        set action accept
        set schedule "always"
        set service "HTTP"
        set utm-status enable
        set inspection-mode proxy
        set ssl-ssh-profile "certificate-inspection"
        set webfilter-profile "Bloqueo-Inventario"
        set logtraffic all
    next
    edit 3
        set name "V20-a-DMZ"
        set srcintf "VLAN20"
        set dstintf "DMZ"
        set srcaddr "VLAN20 address"
        set dstaddr "DMZ address"
        set action accept
        set schedule "always"
        set service "HTTPS" "SSH"
        set logtraffic all
    next
    edit 4
        set name "V10-a-Caja"
        set srcintf "VLAN10"
        set dstintf "DMZ"
        set srcaddr "VLAN10 address"
        set dstaddr "Web-Caja"
        set action accept
        set schedule "always"
        set service "HTTPS"
        set logtraffic all
    next
end
config log setting
    set fwpolicy-implicit-log enable
end
```

La configuración completa está en `running-configs/FG-Empresa.conf`.
</details>

<details>
<summary><b>Web Caja</b></summary>

```
# /etc/netplan/01-lab.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.23.56.130/28]
      mtu: 1400
      routes:
        - to: 0.0.0.0/0
          via: 10.23.56.129

# Apache: /etc/apache2/sites-available/default-ssl.conf
SSLCertificateFile /etc/ssl/certs/lab.crt
SSLCertificateKeyFile /etc/ssl/private/lab.key

# Certificado
subject=C = DO, ST = Santo Domingo, O = ITLA, OU = Lab DMZ, CN = 10.23.56.130
X509v3 Subject Alternative Name: IP Address:10.23.56.130

# /etc/ssh/sshd_config
PermitRootLogin no
```
</details>

<details>
<summary><b>Inventario</b></summary>

```
# /etc/netplan/01-lab.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.23.56.131/28]
      mtu: 1400
      routes:
        - to: 0.0.0.0/0
          via: 10.23.56.129

# Apache: /etc/apache2/sites-available/default-ssl.conf
SSLCertificateFile /etc/ssl/certs/lab.crt
SSLCertificateKeyFile /etc/ssl/private/lab.key

# Certificado
subject=C = DO, ST = Santo Domingo, O = ITLA, OU = Lab DMZ, CN = 10.23.56.131
X509v3 Subject Alternative Name: IP Address:10.23.56.131

# /etc/ssh/sshd_config
PermitRootLogin no
```
</details>

<details>
<summary><b>DB server</b></summary>

```
# /etc/netplan/01-lab.yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.23.56.132/28]
      mtu: 1400
      routes:
        - to: 0.0.0.0/0
          via: 10.23.56.129

# /etc/mysql/mariadb.conf.d/50-server.cnf
bind-address = 0.0.0.0

# Bases de datos
caja
inventario

# Usuario de las aplicaciones
GRANT ALL PRIVILEGES ON `caja`.* TO `app`@`10.23.56.128/255.255.255.240`
GRANT ALL PRIVILEGES ON `inventario`.* TO `app`@`10.23.56.128/255.255.255.240`

# /etc/ssh/sshd_config
PermitRootLogin no
```
</details>

## Pendiente

- [ ] Política de salida de la DMZ solo hacia los endpoints de actualización.

## Estructura del repositorio

```
Lab-DMZ-FortiGate-2024-2356/
├── README.md
├── documentacion/
├── diagramas/
├── imagenes/
└── running-configs/
    ├── ISP.txt
    ├── SW-Usuarios.txt
    ├── FG-Empresa.conf
    ├── Web-Caja.txt
    ├── Inventario.txt
    └── DB-Server.txt
```
