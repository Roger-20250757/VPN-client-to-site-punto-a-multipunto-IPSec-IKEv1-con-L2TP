# VPN Client to Site - L2TP sobre IPSec (IKEv1)

**Autor:** Roger Rodriguez  
**Matrícula:** 2025-0757  
**Fecha:** Julio 2026  
**Link:** https://youtu.be/pLqrnkUnj9Q

---

## Objetivo del laboratorio

Demostrar la implementación de una VPN **client-to-site punto a multipunto**,
utilizando **L2TP sobre IPSec con IKEv1**, donde un router GW actúa como
concentrador VPN aceptando conexiones dinámicas de clientes remotos (sin
conocer su IP pública de antemano). El cliente remoto es un **Kali Linux**,
conectado mediante **NetworkManager (GUI)**.

---

## Objetivo de la configuración

1. Establece la Fase 1 (ISAKMP) con cifrado AES 256, hash SHA-256 y llave
   precompartida, aceptando cualquier peer (`address 0.0.0.0 0.0.0.0`)
2. Define un `crypto dynamic-map` con transform-set en modo transporte, sin
   restricción de tráfico por ACL (patrón estándar en VPN de acceso remoto)
3. Habilita VPDN con protocolo L2TP, apuntando a una interfaz virtual (PPP)
4. Asigna direcciones IP a los clientes desde un pool local, con
   autenticación de usuario vía MS-CHAP-v2/CHAP

---

## Parámetros usados

| Parámetro | Valor | Descripción |
|---|---|---|
| `IKE` | `IKEv1` | Protocolo de intercambio de llaves |
| Tipo de VPN | Client to site | L2TP sobre IPSec |
| `ENCRYPTION` | `AES 256` | Cifrado |
| `HASH` | `SHA-256` | Integridad |
| `AUTH` (IPSec) | `pre-share` | Llave precompartida |
| `TRANSFORM-SET` | `esp-aes 256 esp-sha256-hmac` | Modo transporte |
| `PFS` | Deshabilitado | Ver nota técnica |
| Protocolo túnel L2 | `L2TP` | UDP 1701 |
| Autenticación PPP | `ms-chap-v2 chap` | Usuario/contraseña |
| Usuario de prueba | `vpnuser` | |
| Pool de direcciones | `10.7.59.129 - 10.7.59.142` | Asignación dinámica |
| Cliente VPN | Kali Linux | NetworkManager + strongSwan |

---

## Requisitos para utilizar la configuración

### Software / Plataforma
- EVE-NG, Cisco IOS c7206vxr
- Kali Linux con `network-manager-l2tp` instalado:
```bash
sudo apt update
sudo apt install -y network-manager-l2tp network-manager-l2tp-gnome
```

### Acceso a modo de configuración (router)
```bash
enable
configure terminal
```

---

## Documentación del funcionamiento

### ¿Cómo funciona L2TP sobre IPSec?

IPSec en modo transporte cifra el tráfico L2TP (UDP 1701) entre el cliente
y el GW. Dentro de ese túnel cifrado, L2TP transporta una sesión PPP, que
se encarga de autenticar al usuario y asignarle una IP interna. El
resultado es que el cliente remoto "aparece" como un host más dentro de la
red interna, con tráfico cifrado extremo a extremo hasta el GW.

### Flujo de conexión

```
Kali --IKE Fase 1 (ISAKMP)--> GW
Kali --IKE Fase 2 (IPSec transporte, protege UDP/1701)--> GW
Kali --L2TP tunnel + PPP (usuario/contraseña)--> GW
GW --asigna IP del pool--> Kali (interfaz ppp0)
Kali (ppp0) --ping-->  LAN interna del GW
```

---

## Documentación de la red

### Topología

|<img width="430" height="361" alt="image" src="https://github.com/user-attachments/assets/a98c50a1-4651-4d99-828c-067cd4512ddb" />
|

### Interfaces y direccionamiento

| Dispositivo | Interfaz | IP | Conecta a |
|---|---|---|---|
| GW | Fa0/0 | 10.7.59.1/30 | ISP Fa0/0 |
| GW | Fa1/0 | 10.7.59.33/27 | SW-LAN |
| ISP | Fa0/0 | 10.7.59.2/30 | GW |
| ISP | Fa1/0 | 10.7.59.5/30 | Kali eth0 |
| Kali | eth0 | 10.7.59.6/30 | ISP Fa1/0 (simula internet) |
| Kali | eth1 | DHCP | Gestión / SSH |
| Kali | eth2 | DHCP | Internet real (instalación paquetes) |
| PC-LAN | eth0 | 10.7.59.34/27 | SW-LAN |

---

```

### Configuración del cliente (Kali)
```bash
sudo nmcli connection add \
  type vpn \
  vpn-type l2tp \
  con-name "L2TP-GW" \
  ifname "*" \
  vpn.data 'gateway=10.7.59.1, user=vpnuser, ipsec-enabled=yes, ipsec-psk=Cisco0757, password-flags=0, ipsec-ike=aes256-sha256-modp2048!, ipsec-esp=aes256-sha256!'

sudo nmcli connection modify "L2TP-GW" +vpn.secrets "password=Cisco0757"
```

Conectar desde la GUI: ícono de red → **L2TP-GW** → conectar.

### Resultado esperado
```
GW#show crypto isakmp sa
dst             src             state          conn-id status
10.7.59.1       10.7.59.6       QM_IDLE           ACTIVE

GW#show vpdn session all
Session state is established
Session username is vpnuser

# En Kali:
$ ip addr show ppp0
inet 10.7.59.129 peer 10.7.59.33/32
```

---

## Nota técnica de resolución de problemas

Durante la implementación se presentaron 3 errores en cascada en la Fase 2
(IPSec Quick Mode), diagnosticados con `debug crypto isakmp` en el router:

1. **ACL de proxy-identity mal formada** → error "proxy identities not
   supported". Solución: eliminar la ACL, dejar el dynamic-map sin
   restricción (estándar en VPN de acceso remoto).
2. **Cliente proponiendo AES-128 en vez de 256** → error "transform
   proposal not supported". Solución: forzar algoritmos exactos en el
   cliente con la sintaxis estricta de strongSwan (`!` al final).
3. **PFS habilitado en el router sin el intercambio KE correspondiente del
   cliente** → error "invalid transform proposal flags". Solución:
   deshabilitar PFS en el `crypto dynamic-map`.

---
