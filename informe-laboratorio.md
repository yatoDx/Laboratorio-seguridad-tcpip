# Informe final — Laboratorio de monitoreo de seguridad de red

**Caso:** C0 · **Fecha:** 2026-11-18 · **Entorno:** VirtualBox (Windows) · **Red aislada:** 192.168.50.0/24

---

## 1. Objetivo

Montar un laboratorio de red aislado para el análisis de tráfico y monitoreo de
seguridad, compuesto por cuatro máquinas virtuales Ubuntu Server y un equipo
analista (host Windows) con herramientas de captura y análisis. El laboratorio
opera en modo **solo-anfitrión (host-only)**, sin salida a Internet, con
servicios de infraestructura (NTP, DNS, web y XMPP) y un firewall perimetral.

---

## 2. Topología y direccionamiento

```
                        Red aislada 192.168.50.0/24 (solo-anfitrión)
        ┌────────────────────────────────────────────────────────────┐
        │                                                            │
   ┌────┴─────┐      ┌──────────┐   ┌──────────┐   ┌──────────┐     │
   │ gateway  │      │   web    │   │   dns    │   │   xmpp   │     │
   │  .1      │      │  .20     │   │  .53     │   │  .30     │     │
   │ chrony   │      │ nginx    │   │ dnsmasq  │   │ prosody  │     │
   │ nftables │      │ :80      │   │ :53      │   │ :5222    │     │
   └────┬─────┘      └────┬─────┘   └────┬─────┘   └────┬─────┘     │
        │                 │              │               │           │
        └─────────────────┴──────────────┴───────────────┘           │
                                 │                                   │
                          ┌──────┴──────┐                            │
                          │  analista   │  host Windows              │
                          │   .10       │  Wireshark + Python        │
                          └─────────────┘                            │
        └────────────────────────────────────────────────────────────┘
```

| Rol | VM | IP | Servicio(s) |
|-----|----|----|-------------|
| gateway | lab-gateway | 192.168.50.1 | chrony (NTP), nftables |
| web | lab-web | 192.168.50.20 | nginx (HTTP :80) |
| dns | lab-dns | 192.168.50.53 | dnsmasq (DNS :53) |
| xmpp | lab-xmpp | 192.168.50.30 | prosody (XMPP c2s :5222) |
| analista | host Windows | 192.168.50.10 | Wireshark, Python 3 |

Cada VM tiene dos adaptadores de red:

- **Adaptador 1 (NAT, enp0s3):** usado únicamente durante la instalación para
  descargar paquetes. Se mantiene con DHCP pero **sin rutas**
  (`dhcp4-overrides: use-routes: false`), de modo que no introduce rutas por
  defecto ni rutas estáticas de clase y no permite salida a Internet.
- **Adaptador 2 (solo-anfitrión, enp0s8):** IP estática en 192.168.50.0/24, es
  la red operativa del laboratorio.

---

## 3. Inventario de software

### Plataforma de virtualización
- VirtualBox **7.2.22** (r175245).

### Máquinas virtuales (4 × 1 vCPU / 1 GB RAM)
- Ubuntu Server **24.04.5 LTS**, kernel **6.8.0-149-generic**.

| Componente | Versión |
|------------|---------|
| chrony | 4.5-1ubuntu4.2 |
| nftables | (nativo del kernel) |
| nginx | 1.24.0 |
| dnsmasq | 2.91 |
| prosody | 0.12.4 |

### Host analista (Windows)
| Herramienta | Versión |
|-------------|---------|
| Wireshark | 4.6.9 |
| Npcap | 1.90 |
| Python | 3.14.7 |
| scikit-learn | 1.9.1 |
| pandas | 3.1.1 |
| matplotlib | 3.11.2 |

---

## 4. Configuración por rol

### 4.1 Gateway (192.168.50.1)

- **Zona horaria:** `America/Bogota`.
- **NTP:** chrony sirve hora al laboratorio (`allow 192.168.50.0/24`,
  `local stratum 10`), escuchando en el puerto 123.
- **Reenvío deshabilitado:** `net.ipv4.ip_forward = 0`.
- **Firewall (nftables):**

```
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    iif lo accept
    ct state established,related accept
    ip saddr 192.168.50.0/24 icmp type echo-request accept
    ip saddr 192.168.50.0/24 udp dport 123 accept
    ip saddr 192.168.50.0/24 tcp dport 22 accept
  }
  chain forward { type filter hook forward priority 0; policy drop; }
}
```

### 4.2 Web (192.168.50.20)

- nginx activo y escuchando en el puerto 80. Responde `HTTP/1.1 200 OK` en
  `http://192.168.50.20/`.

### 4.3 DNS (192.168.50.53)

- dnsmasq escuchando en el puerto 53 (UDP/TCP), configuración
  (`/etc/dnsmasq.d/lab.conf`):

```
listen-address=192.168.50.53
bind-interfaces
no-resolv
address=/web.lab.local/192.168.50.20
```

- Resolución verificada: `web.lab.local → 192.168.50.20`.

### 4.4 XMPP (192.168.50.30)

- prosody activo, escuchando en el puerto 5222 (cliente-servidor c2s).

### 4.5 Analista (192.168.50.10)

- Adaptador solo-anfitrión con IP estática 192.168.50.10/24 y DNS 192.168.50.53.
- Regla de firewall de Windows que permite ICMP entrante desde 192.168.50.0/24
  (`Lab-ICMPv4-Echo-In-192.168.50.0-24`, Inbound, Allow).
- Wireshark + Npcap instalados; captura en vivo sobre el adaptador
  "Ethernet 2" verificada.

---

## 5. Seguridad de red

- **Aislamiento:** sin ruta por defecto en las VMs; la prueba de aislamiento
  (ping a `192.0.2.1`) no responde en ninguna VM.
- **Firewall perimetral:** política `drop` en las cadenas `input` y `forward`
  del gateway; solo se acepta tráfico de gestión (ICMP echo, NTP 123, SSH 22)
  desde la red del laboratorio.
- **Sin reenvío de paquetes:** `ip_forward = 0` (el gateway no enruta).
- **Identidades SSH únicas:** las claves de host SSH se regeneraron en las VMs
  clonadas, de modo que cada VM tiene una huella distinta.

---

## 6. Sincronización de tiempo (NTP)

- El gateway actúa como servidor NTP del laboratorio (chrony, `local stratum 10`).
- Los clientes (web, dns, xmpp) sincronizan con el gateway mediante
  systemd-timesyncd (`NTP=192.168.50.1`).
- Estado verificado en todas las VMs: `System clock synchronized: yes`,
  zona horaria `America/Bogota`.

---

## 7. Verificación

Se ejecutó `verificar_laboratorio.sh <rol>` en las cuatro VMs. Resultado:
**todos los chequeos `[OK]`** — conectividad completa de la malla (gateway,
analista, web, dns, xmpp responden entre sí), aislamiento confirmado y
servicios activos por rol (chrony/NTP, nftables, nginx, dnsmasq, prosody).

| VM | IP | Servicio verificado |
|----|----|---------------------|
| gateway | .1 | chrony (123), nftables, ip_forward=0 |
| web | .20 | nginx (80), HTTP 200 OK |
| dns | .53 | dnsmasq (53), web.lab.local → .20 |
| xmpp | .30 | prosody (5222) |

---

## 8. Evidencia forense

Los artefactos de evidencia se encuentran en `lab/evidencia/` y sus resúmenes
criptográficos (SHA-256, SHA-1, MD5 + tamaño + fecha) en `lab/evidencia.log`.

### Capturas de texto

| Archivo | SHA-256 |
|---------|---------|
| verificacion-gateway.txt | `389c3189ada8cd32cfd7fc2220ca594ee006ae5aa35018061b06ef88661ce8bf` |
| verificacion-web.txt | `e77c8b08c586b2ecea304458f1115498bfb7d1fd039db698b083a9a3e73f2eb6` |
| verificacion-dns.txt | `8cc8a414f1ac9944148d2d7ba3704e017b4e870da3d2da014909876b41f0fe48` |
| verificacion-xmpp.txt | `22746f6daa1df58e0680728f6d8a843d640bc7a1182cb0daf65225f4a4e8899b` |
| config-gateway.txt | `587fb41aeba65ea5156ee1c3767624ca4c445dc03bcb84160360879d614abcf2` |
| config-web.txt | `64d8876a874c6de64d49917c8044be7aa9c061a72925c3e85ab10c9fe4a1fe4a` |
| config-dns.txt | `996bbfcdd3d1269f1575d9bf439e861645a0ef12f2f4a36343e8976525343158` |
| config-xmpp.txt | `68bb31dd143f9b507173b334c1a7e23c58d0c39480b1b3abb1b13574ad947e97` |
| config-host.txt | `a76c2cbc84772eeabec41611b9bf10af7d2384662f07e0729889e6e203c1cd17` |
| wireshark-version.txt | `d62e5610c4ce1bf6f9d3c1047c7023152f19d6a11920272077cc0830fbfd8433` |

### Capturas de pantalla (PNG)

| Archivo | Contenido | SHA-256 |
|---------|-----------|---------|
| gateway-firewall.png | nftables + ip_forward=0 | `f69c81637db7d3700cbf7c39eeaa96bff4923f0f84234b04f99e40a257541cc2` |
| web-nginx.png | IP, nginx active, HTTP 200 | `a0db955414fc1178247bfe44bbdc4ba852c49c24b5403e1b5feefb5030b1d063` |
| dns-resolucion.png | dig web.lab.local → .20 | `2187a183d6a7a5e76b5f6e45f1cdc6fc6e9fc50e29685ea171c258953e338b06` |
| xmpp-prosody.png | prosody active, :5222 | `86a96a0a62f82a995827e787648a1761c80590d19ad892e2d253f79b95788904` |
| host-wireshark.png | captura en vivo (ICMP/HTTP/DNS/ARP) | `82455f8af4cb0ee86c4437e5c2e14546dc8f18d1f332f9f97c2b43b90583d5ab` |

### Snapshots de VirtualBox

Snapshot `lab-base-aislado` tomado en las cuatro VMs:

| VM | UUID |
|----|------|
| lab-gateway | `ca2594b3-1ae3-4645-ae2f-bd470c40d7fd` |
| lab-web | `e10550ab-4905-4e6d-b095-00233fc459ac` |
| lab-dns | `55768ab5-9fdc-4c80-ae9f-28191827db78` |
| lab-xmpp | `bfe946eb-9790-430d-aefd-db6de7a6a736` |

---

## 9. Observaciones técnicas

- **Rutas "fantasma" del DHCP de VirtualBox NAT:** el servidor DHCP del adaptador
  NAT inyectaba rutas estáticas de clase (opción 121) hacia `192.168.50.53` y
  `192.168.0.1` vía `10.0.2.2`, lo que rompía el enrutamiento asimétrico entre
  la VM DNS y el resto del laboratorio. Solución permanente:
  `dhcp4-overrides: use-routes: false` en el netplan de las cuatro VMs.
- **Puerto 53 ocupado por systemd-resolved:** se añadió `bind-interfaces` a la
  configuración de dnsmasq para evitar el conflicto en 127.0.0.53.
- **ICMP del host bloqueado por Windows Firewall:** se creó una regla que permite
  echo ICMP entrante desde 192.168.50.0/24 para que el analista responda a ping.
- **Claves SSH duplicadas en clones:** regeneradas para que cada VM tenga una
  identidad única.
- **Npcap:** no venía incluido en la instalación de Wireshark; se instaló
  Npcap 1.90 (npcap.com) y se verificó captura sin privilegios de administrador.

---

## 10. Conclusión

El laboratorio quedó operativo y verificado: las cuatro VMs cumplen su rol,
la red está aislada, el firewall del gateway aplica la política de
denegación por defecto, la hora está sincronizada y el equipo analista cuenta
con las herramientas de captura (Wireshark/Npcap) y análisis (Python con
scikit-learn, pandas y matplotlib). La evidencia (salidas de verificación,
configuraciones y capturas de pantalla) quedó resguardada con resúmenes
criptográficos y snapshots para reproducibilidad.
