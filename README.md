# Proyecto Final: Diseño e Implementación de una Red Empresarial

**Curso:** Redes y Comunicación de Datos, 2026-2
**Equipo:** Gabrielle Romandino, Daniel Ortiz, Jerónimo Coy y Martín Guerrero
**Dominio:** `gdjm.net`
**Herramienta:** Cisco Packet Tracer 9.0.1

---

## 1. Resumen

Se diseñó e implementó en Cisco Packet Tracer una red empresarial con dos intranets (Bogotá y Madrid) y una DMZ en IPv6, interconectadas a través de un Internet de cuatro routers que opera solo en IPv4. La solución incluye VLANs con trunks 802.1Q, enrutamiento inter-VLAN con un Multilayer Switch, OSPF y EIGRP con redistribución, un túnel IPv6 sobre IPv4 protegido con IPsec, DNS y web, ACLs por puerto, SNMP, NTP y una aplicación de seguimiento de empleados (Employee Tracker) con sockets TCP.

---

## 2. Metodología

1. **Diseño del direccionamiento** con la guía de diseño estructurado vista en clase, tomando X = 1 en los prefijos de la Figura 1.
2. **Montaje físico:** 6 routers 2811 (con dos módulos WIC-2T en los routers del Internet), 6 switches 2960 y 1 Multilayer Switch 3650.
3. **Configuración básica:** hostnames, contraseñas, acceso por consola y VTY.
4. **Capa 2:** VLANs, trunks 802.1Q con VLAN nativa 99, y STP con SW-intranet como raíz.
5. **Capa 3:** subinterfaces (router-on-a-stick) en Bogotá, SVIs en el Multilayer Switch en Madrid.
6. **Internet IPv4:** EIGRP (AS 100) y OSPF (proceso 1, área 0) con redistribución mutua en ISP-bog e ISP-esp.
7. **Túnel IPv6 sobre IPv4** (`ipv6ip`) entre R-bog y R-esp, más IPsec sobre las interfaces seriales.
8. **Servicios:** DNS, HTTP/HTTPS, DHCPv6 stateless con SLAAC, SNMP y NTP.
9. **Seguridad:** ACLs IPv6 por VLAN para HTTP/HTTPS y SNMP.
10. **Employee Tracker** con tres aplicaciones en JavaScript.
11. **Verificación** de cada etapa con ping, `show` y capturas.

En cada paso se probó antes de continuar, de capa 1 hacia arriba.

---

## 3. Diseño del direccionamiento

### 3.1 VLANs (iguales en ambas intranets)

| VLAN | Nombre | Uso |
|---|---|---|
| 10 | Guest | Usuarios invitados |
| 20 | Internal | Usuarios internos de la empresa |
| 30 | Server | Servidores |
| 40 | Printer | Impresoras |
| 99 | Native | VLAN nativa de los trunks |

Asignación de equipos (según la Figura 1 del enunciado: Bogotá Guest PC1 y PC3, Internal PC2 y PC4; Madrid Guest PC5 y PC8, Internal PC6 y PC7):

| Sede | Guest (VLAN 10) | Internal (VLAN 20) | Printer (VLAN 40) | Server (VLAN 30) |
|---|---|---|---|---|
| Bogotá | PC1, PC3 | PC2, PC4 | Printer0, Printer1 | Server-dhcp |
| Madrid | PC5, PC8 | PC6, PC7 | Printer2, Printer3 | (sin equipos) |

### 3.2 IPv4: Internet (`86.106.77.0/24`, es decir 76 + X)

Cada enlace serial usa una subred /30 (2 hosts útiles).

| Enlace | Red | Extremo A | Extremo B |
|---|---|---|---|
| ISP-bog con R-bog | 86.106.77.0/30 | ISP-bog .1 | R-bog .2 |
| ISP-bog con ISP-inet | 86.106.77.4/30 | ISP-bog .5 | ISP-inet .6 |
| ISP-bog con ISP-fl | 86.106.77.8/30 | ISP-bog .9 | ISP-fl .10 |
| ISP-inet con ISP-esp | 86.106.77.12/30 | ISP-inet .13 | ISP-esp .14 |
| ISP-inet con ISP-fl | 86.106.77.16/30 | ISP-inet .17 | ISP-fl .18 |
| ISP-fl con ISP-esp | 86.106.77.20/30 | ISP-fl .21 | ISP-esp .22 |
| ISP-esp con R-esp | 86.106.77.24/30 | ISP-esp .25 | R-esp .26 |

El enlace ISP-inet con ISP-fl tiene dirección y responde a ping, pero no se anuncia en ningún protocolo de enrutamiento, porque cruza la frontera entre EIGRP y OSPF y se evitó mezclarlos fuera de los routers de borde.

Red de gestión (solo Madrid, ver sección 5.7): `192.168.120.0/24` en la VLAN 20 del Multilayer Switch.

### 3.3 IPv6 (X = 1)

| Zona | Prefijo | Observación |
|---|---|---|
| Intranet Bogotá | `2001:1200:EA11::/48` | Una /64 por VLAN: `EA11:10::/64`, `:20::`, `:30::`, `:40::` |
| DMZ | `2001:1200:EC11::/48` | `EC11:1::/64` |
| Intranet Madrid | `2001:1200:EB21::/48` | Una /64 por VLAN: `EB21:10::/64`, `:20::`, `:30::`, `:40::` |
| Enlace R-esp con MLSW | `2001:1200:ED21:1::/64` | |
| Loopbacks | `2001:1200:ED11::/48` | R-bog: `ED11:1::1/128` |
| Túnel R-bog con R-esp | `2001:1200:ED11:100::/64` | R-bog `::1`, R-esp `::2` |

Direcciones estáticas de servidores:

| Equipo | IPv6 | Gateway |
|---|---|---|
| Server-web | `2001:1200:EC11:1::10/64` | `2001:1200:EC11:1::1` |
| Server-dns | `2001:1200:EC11:1::11/64` | `2001:1200:EC11:1::1` |
| Server-tracker | `2001:1200:EC11:1::12/64` | `2001:1200:EC11:1::1` |
| Server-dhcp | `2001:1200:EA11:30::10/64` | `2001:1200:EA11:30::1` |
| Printer0 / Printer1 | `EA11:40::10` / `EA11:40::11` | `2001:1200:EA11:40::1` |
| Printer2 / Printer3 | `EB21:40::10` / `EB21:40::11` | `2001:1200:EB21:40::1` |

Los gateways de cada VLAN son `::1`: subinterfaces `Fa0/1.xx` de R-bog en Bogotá y SVIs del MLSW en Madrid.

---

## 4. Topología y conexiones

![Topología completa en Packet Tracer](capturas/01-topologia.png)

### 4.1 Cableado

| Dispositivo A | Puerto A | Dispositivo B | Puerto B | Cable |
|---|---|---|---|---|
| ISP-bog | Se0/3/0 | R-bog | Se0/3/0 | Serial |
| ISP-bog | Se0/3/1 | ISP-inet | Se0/3/0 | Serial |
| ISP-bog | Se0/2/0 | ISP-fl | Se0/3/0 | Serial |
| ISP-inet | Se0/3/1 | ISP-esp | Se0/3/0 | Serial |
| ISP-inet | Se0/2/0 | ISP-fl | Se0/3/1 | Serial |
| ISP-fl | Se0/2/0 | ISP-esp | Se0/3/1 | Serial |
| ISP-esp | Se0/2/0 | R-esp | Se0/3/0 | Serial |
| R-bog | Fa0/0 | SW-dmz | Fa0/1 | Directo |
| R-bog | Fa0/1 | SW-intranet | Fa0/1 | Directo |
| SW-intranet | Fa0/2, Fa0/3 | SW-intranet2, SW-intranet3 | Fa0/1 | Cruzado |
| SW-intranet2 | Fa0/2 | SW-intranet3 | Fa0/2 | Cruzado |
| R-esp | Fa0/0 | MLSW | Gi1/0/1 | Directo |
| MLSW | Gi1/0/2, Gi1/0/3 | SW-intranet4, SW-intranet5 | Fa0/1 | Cruzado |
| SW-intranet4 | Fa0/2 | SW-intranet5 | Fa0/2 | Cruzado |

Los equipos finales van a los puertos Fa0/2 a Fa0/6 de su switch (ver `show vlan brief` en las capturas).

---

## 5. Configuración y verificación

### 5.1 Capa 2: VLANs y trunks

- Trunks 802.1Q con VLAN nativa 99 en todos los enlaces entre switches y hacia el router o MLSW.
- STP (PVST): SW-intranet con prioridad 4096, así que es la raíz en Bogotá. El punto naranja que se ve en la topología es STP bloqueando un enlace del triángulo para evitar bucles.

![show vlan brief y show interfaces trunk en SW-intranet](capturas/25-sw-intranet-vlan-trunks.png)

![SW-intranet es el root bridge de la VLAN 10 (prioridad 4096)](capturas/26-sw-intranet-stp-vlan10.png)

### 5.2 Enrutamiento

| Zona | Protocolo | Detalle |
|---|---|---|
| ISP-inet | EIGRP AS 100 | Redes `.4/30` y `.12/30` |
| ISP-fl | OSPF 1, área 0 | Redes `.8/30` y `.20/30` |
| ISP-bog | Frontera | EIGRP para `.0/30` y `.4/30`; OSPF para `.8/30`; redistribución mutua |
| ISP-esp | Frontera | EIGRP para `.12/30` y `.24/30`; OSPF para `.20/30`; redistribución mutua |
| R-bog, R-esp | Ruta por defecto | Hacia su ISP (`.1` y `.25`) |
| Intranets y DMZ | Rutas IPv6 estáticas | Hacia el otro extremo del túnel |

Decisión de diseño: en IPv6 se usaron rutas estáticas por el túnel para priorizar la estabilidad. El enunciado sugiere OSPFv3 o EIGRPv6, y queda como mejora futura.

![show ip route en ISP-bog: rutas D (EIGRP) y O (OSPF)](capturas/22-isp-bog-ip-route.png)

![Vecinos EIGRP y OSPF de ISP-bog](capturas/23-isp-bog-eigrp-ospf-vecinos.png)

![Ping de R-bog a R-esp (86.106.77.26): 100 %](capturas/24-ping-rbog-resp.png)

### 5.3 Asignación de direcciones

- **SLAAC** en todos los PCs e impresoras de Bogotá y Madrid (los routers y el MLSW anuncian el prefijo).
- **DHCPv6 stateless** en R-bog (pool `BOG-STATELESS`): entrega DNS `2001:1200:EC11:1::11` y dominio `gdjm.net`, con `other-config-flag` en las VLAN 10 y 20.
- En Madrid, el MLSW solo anuncia SLAAC, así que el DNS se configura a mano en los PCs.
- Servidores e impresoras: IPv6 estática.

Justificación: SLAAC evita administrar un pool de direcciones y DHCPv6 stateless cubre los datos adicionales (DNS y dominio).

![PC7 (Madrid) con IPv6 automática (SLAAC)](capturas/19-pc7-ipconfig.png)

![ipconfig de un PC de Internal: dirección SLAAC y gateway](capturas/27-pc4-ping-gdjm-ipconfig.png)

### 5.4 DNS y servidor web

- **Server-dns:** servicio DNS activo con un registro AAAA `gdjm.net` apuntando a `2001:1200:EC11:1::10`.
- **Server-web:** HTTP y HTTPS activos, con `index.html` personalizado con los nombres del equipo.

![PC4 abriendo http://gdjm.net](capturas/02-web-http-pc4.png)

![PC4 abriendo https://gdjm.net](capturas/03-web-https-pc4.png)

![ping gdjm.net resuelto a 2001:1200:EC11:1::10, 0 % de pérdida](capturas/27-pc4-ping-gdjm-ipconfig.png)

### 5.5 ACLs (filtros de paquetes)

| Origen | HTTP (80) | HTTPS (443) | Dónde se aplica |
|---|---|---|---|
| Bogotá Guest | Permitido | Bloqueado | `BOG-GUEST` in, `Fa0/1.10` de R-bog |
| Bogotá Internal | Bloqueado | Permitido | `BOG-INTERNAL` in, `Fa0/1.20` de R-bog |
| Madrid Guest | Permitido | Bloqueado | `MAD-FILTRO` in, `Fa0/0` de R-esp |
| Madrid Internal | Bloqueado | Permitido | `MAD-FILTRO` in, `Fa0/0` de R-esp |

Además, las ACL de Guest bloquean SNMP (UDP, puerto 161): `BOG-GUEST` en Bogotá y `MAD-FILTRO` (para las VLAN 10 y 40) en Madrid. Todas terminan en `permit ipv6 any any`, para no romper DNS, ping, SLAAC ni el Tracker (TCP 5000). Las ACLs se colocan lo más cerca posible del origen del tráfico.

**Bogotá Guest (PC1):** HTTP permitido, HTTPS bloqueado.

![PC1 http permitido](capturas/04-acl-pc1-http-ok.png)

![PC1 https bloqueado (sin respuesta)](capturas/05-acl-pc1-https-bloqueado.png)

**Bogotá Internal (PC4):** HTTP bloqueado, HTTPS permitido.

![PC4 http bloqueado](capturas/06-acl-pc4-http-bloqueado.png)

![PC4 https permitido](capturas/07-acl-pc4-https-ok.png)

**Madrid Guest (PC5):** HTTP permitido, HTTPS bloqueado.

![PC5 http permitido](capturas/08-acl-pc5-http-ok.png)

![PC5 https: Request Timeout](capturas/09-acl-pc5-https-timeout.png)

**Madrid Internal (PC7):** HTTP bloqueado, HTTPS permitido.

![PC7 http: Request Timeout](capturas/10-acl-pc7-http-timeout.png)

![PC7 https permitido](capturas/11-acl-pc7-https-ok.png)

![show ipv6 access-list MAD-FILTRO en R-esp: la regla de SNMP para Guest registra 1 coincidencia](capturas/46-resp-mad-filtro-snmp.png)

![show ipv6 access-list en R-bog: los deny de 443 (Guest) y www (Internal) registran coincidencias](capturas/32-rbog-ipv6-access-list.png)

### 5.6 Migración IPv6 y VPN IPsec

- **Migración:** túnel manual `ipv6ip` entre R-bog y R-esp (fuentes `86.106.77.2` y `86.106.77.26`), que transporta el IPv6 sobre el Internet IPv4.
- **IPsec:** ISAKMP con AES, SHA, autenticación por clave precompartida y Diffie-Hellman grupo 2; transform-set `esp-aes esp-sha-hmac`; crypto map `CMAP` aplicado en `Se0/3/0` de los dos routers. La ACL `VPN-GRE` (nombre heredado de un intento con GRE) cifra el tráfico IP entre ambas seriales.
- **Resultado:** `show crypto isakmp sa` muestra `QM_IDLE / ACTIVE`, y `show crypto ipsec sa` registra paquetes cifrados y descifrados (10 en la verificación). PC7 llega a Server-web con IPsec activo.

![Ping de PC7 (Madrid) a Server-web por el túnel](capturas/29-pc7-ping-server-web.png)

![show crypto isakmp sa: QM_IDLE / ACTIVE](capturas/33-rbog-crypto-isakmp.png)

![show crypto ipsec sa: 181 paquetes cifrados y 181 descifrados](capturas/34-rbog-crypto-ipsec.png)

En modo Simulation, los paquetes que cruzan el Internet IPv4 van protegidos con IPsec: en ISP-bog y en ISP-esp aparecen con protocolo `0x32` (ESP), de `86.106.77.2` a `86.106.77.26` y de regreso. El contenido, incluido el IPv6 del túnel, no es visible para los ISP.

![Paquete ESP en ISP-bog: 86.106.77.2 a 86.106.77.26, protocolo 0x32](capturas/35-esp-isp-bog.png)

![Paquete ESP de regreso en ISP-esp: 86.106.77.26 a 86.106.77.2, protocolo 0x32](capturas/36-esp-isp-esp.png)

### 5.7 SNMP

| Elemento | Resultado |
|---|---|
| Bogotá: `get` de `sysName` desde PC4 (Internal) a R-bog por IPv6 | Devuelve `R-bog` |
| Bogotá: `set` de `sysName` desde PC4 | Devuelve el nuevo valor (`R-bog-gdjm`) |
| Bogotá: `get` desde PC1 (Guest) | Timeout: `deny udp any any eq snmp` en `BOG-GUEST` registra coincidencias |
| Madrid: `get` de `sysName` desde PC7 (Internal) a R-esp por IPv6 (`2001:1200:ED21:1::1`) | Devuelve `R-esp` |
| Madrid: `set` de `sysName` desde PC7 | Devuelve el nuevo valor (`R-esp-gdjm`), y un `get` posterior lo confirma |
| Madrid: `get` desde PC5 (Guest) | Timeout: `deny udp 2001:1200:EB21:10::/64 any eq snmp` en `MAD-FILTRO` |

En Madrid, el equipo gestionado es **R-esp**, un router 2811 igual que R-bog. Primero se probó el Multilayer Switch (`192.168.120.1`), que tiene la comunidad `redinterna` configurada y responde a ping, pero no contestó SNMP (timeout con la comunidad configurada en el MIB Browser). Como el enunciado pide gestionar "los dispositivos" de cada intranet con `get` y `set` sin exigir que sea el switch, se configuró `snmp-server community redinterna RW` en R-esp y desde PC7 se comprobaron `get` y `set`. El comportamiento del SNMP del MLSW en Packet Tracer no se pudo determinar, y queda como límite observado.

Decisión de diseño: solo la VLAN Internal puede gestionar; las VLAN Guest (en ambas sedes) se bloquean con una ACL. En Madrid, la regla se añadió a `MAD-FILTRO` (`deny udp` desde las VLAN 10 y 40 con destino al puerto SNMP). La VLAN 30 de Bogotá y las de servidores no llevan ACL de SNMP, porque solo tienen servidores e impresoras que no gestionan equipos.

![Get de sysName desde PC4: R-bog](capturas/12-snmp-get-pc4.png)

![Set de sysName a R-bog-gdjm](capturas/13-snmp-set-dialogo.png)

![Get posterior al set: R-bog-gdjm](capturas/14-snmp-get-posterior-set.png)

![PC1 (Guest): timeout bloqueado por la ACL](capturas/15-snmp-pc1-timeout.png)

Madrid. Primero, el MLSW (`192.168.120.1`) no respondió:

![PC7 a MLSW por IPv4 192.168.120.1: timeout](capturas/41-snmp-madrid-mlsw-timeout.png)

Gestión de R-esp desde PC7 (Internal): `get`, `set` y `get` posterior.

![Get de sysName desde PC7 a R-esp: R-esp](capturas/42-snmp-madrid-get-resp.png)

![Set de sysName a R-esp-gdjm](capturas/43-snmp-madrid-set.png)

![Get posterior al set: R-esp-gdjm](capturas/44-snmp-madrid-get-posterior.png)

Desde PC5 (Guest), el mismo `get` da timeout porque `MAD-FILTRO` bloquea SNMP:

![PC5 (Guest de Madrid): timeout bloqueado por la ACL](capturas/45-snmp-madrid-pc5-timeout.png)

![BOG-GUEST bloquea HTTPS (25 coincidencias) y SNMP (1 coincidencia)](capturas/32-rbog-ipv6-access-list.png)

### 5.8 Servicio adicional: NTP

R-bog es servidor NTP (`ntp master 3`) y R-esp es cliente (`ntp server 86.106.77.2`). Resultado: R-esp `synchronized`, stratum 4.

**Justificación:** mantener alineados los relojes de las dos sedes permite correlacionar registros y capturas de ambos lados. Es un requisito básico para diagnóstico y auditoría en una red empresarial.

![show ntp status en R-bog (servidor, stratum 3)](capturas/31-rbog-ntp-status.png)

![show ntp status en R-esp (sincronizado, stratum 4)](capturas/30-resp-ntp-status.png)

### 5.9 Employee Tracker

Tres aplicaciones en JavaScript (pestaña Programming) que se comunican por sockets TCP en el puerto 5000:

| Dispositivo | Aplicación | Función |
|---|---|---|
| PC4 (Bogotá) | Tracker App | Cada segundo genera `ID;X;Y;ESTADO` (X, Y entre 0 y 100; estado aleatorio entre "En reposo", "En movimiento" y "Alta actividad") y lo envía al relay |
| Server-tracker (DMZ) | Tracker Relay | Recibe de la App, responde `ACK` y reenvía al Dashboard |
| PC7 (Madrid) | Tracker Dashboard | Muestra la posición y el estado en tiempo real y genera alertas |

Alertas del Dashboard:
- Permanencia prolongada en la zona restringida. El umbral es configurable con la constante `LIMITE_SEG` del Dashboard; el enunciado define 60 segundos y en la demostración se dejó en **10 segundos** para poder mostrar la alerta sin esperar un minuto.
- Actividad anormal: cambio de "En reposo" a "Alta actividad".

El beacon BLE se simula por código dentro de la App (ID `BEACON01` y coordenadas generadas con un movimiento aleatorio acotado), tal como permite el enunciado.

![Tracker Dashboard en PC7 conectado al relay](capturas/18-tracker-dashboard-pc7.png)

**PC4: Tracker App**

```javascript
var RELAY_IP = "2001:1200:EC11:1::12";
var RELAY_PORT = 5000;
var BEACON_ID = "BEACON01";
var ESTADOS = ["En reposo", "En movimiento", "Alta actividad"];

var client = new TCPClient();
var conectado = false;
var x = 50, y = 50;

function limitar(v) { return Math.max(0, Math.min(100, v)); }

function setup() {
  client.onReceive = function (datos) {
    console.log("Respuesta del relay: " + datos);
  };
  conectado = client.connect(RELAY_IP, RELAY_PORT);
  console.log("Conexion con relay: " + (conectado ? "OK" : "fallo"));
}

function loop() {
  if (!conectado) {
    conectado = client.connect(RELAY_IP, RELAY_PORT);
    console.log("Reintentando conexion: " + conectado);
    delay(2000);
    return;
  }
  x = limitar(x + Math.floor(Math.random() * 11) - 5);
  y = limitar(y + Math.floor(Math.random() * 11) - 5);
  var estado = ESTADOS[Math.floor(Math.random() * 3)];
  var msg = BEACON_ID + ";" + x + ";" + y + ";" + estado + "\n";
  client.send(msg);
  console.log("Enviado: " + msg);
  delay(1000);
}
```

**Server-tracker: Tracker Relay**

```javascript
var server = new TCPServer();
var dashboards = [];

function yaRegistrado(c) {
  for (var i = 0; i < dashboards.length; i++) {
    if (dashboards[i] === c) return true;
  }
  return false;
}

function setup() {
  server.onNewClient = function (cliente) {
    console.log("Nuevo cliente: " + cliente.remoteIP());
    cliente.onReceive = function (datos) {
      datos = String(datos);
      if (datos.indexOf("DASH") >= 0) {
        if (!yaRegistrado(cliente)) {
          dashboards.push(cliente);
          console.log("Dashboard registrado: " + cliente.remoteIP());
        }
        return;
      }
      console.log("De la App: " + datos);
      cliente.send("ACK\n");
      for (var i = 0; i < dashboards.length; i++) {
        dashboards[i].send(datos);
      }
    };
  };
  console.log("Relay escuchando: " + server.listen(5000));
}

function loop() { delay(1000); }
```

**PC7: Tracker Dashboard**

```javascript
var RELAY_IP = "2001:1200:EC11:1::12";
var RELAY_PORT = 5000;
var client = new TCPClient();
var conectado = false;
var enZona = {};
var ultimo = {};
var ticks = 0;

// Zona restringida (ajustable)
function restringida(x, y) { return x >= 40 && x <= 60 && y >= 50 && y <= 70; }

// Segundos permitidos en zona restringida (el enunciado indica 60; en la demo se usa 10)
var LIMITE_SEG = 10;

function procesar(datos) {
  var lineas = String(datos).split("\n");
  for (var i = 0; i < lineas.length; i++) {
    if (lineas[i].length == 0) continue;
    var p = lineas[i].split(";");
    if (p.length < 4) continue;
    var id = p[0], x = parseInt(p[1]), y = parseInt(p[2]), est = p[3];
    console.log("[" + id + "] pos(" + x + "," + y + ") " + est);

    if (restringida(x, y)) {
      enZona[id] = (enZona[id] || 0) + 1;
      if (enZona[id] > LIMITE_SEG) {
        console.log("ALERTA: " + id + " lleva " + enZona[id] + " s en zona restringida");
      }
    } else {
      enZona[id] = 0;
    }
    if (ultimo[id] == "En reposo" && est == "Alta actividad") {
      console.log("ALERTA: actividad anormal de " + id + " (reposo a alta actividad)");
    }
    ultimo[id] = est;
  }
}

function conectar() {
  conectado = client.connect(RELAY_IP, RELAY_PORT);
  if (conectado) client.send("DASH\n");
  console.log("Dashboard conectado: " + conectado);
}

function setup() {
  client.onReceive = procesar;
  conectar();
}

function loop() {
  if (!conectado) { conectar(); delay(2000); return; }
  ticks++;
  if (ticks % 5 == 0) client.send("DASH\n");
  delay(1000);
}
```

---

## 6. Resultados y análisis

### 6.1 Flujo bidireccional del Tracker

PC4 (Bogotá) abre una conexión TCP hacia Server-tracker (DMZ), le envía los datos cada segundo y recibe un `ACK`. El relay mantiene además una conexión con PC7 (Madrid) y reenvía cada paquete. El tráfico de PC4 a la DMZ cruza `R-bog`; el de PC7 a la DMZ cruza el MLSW, R-esp y el túnel IPv6 sobre IPv4 cifrado con IPsec.

**Paquete de regreso (Server-tracker → PC4), visto en SW-intranet.** La cabecera IPv6 muestra origen `2001:1200:EC11:1::12` (Server-tracker) y `NEXT: 0x06` (TCP):

![Cabecera IPv6 del ACK: origen Server-tracker, siguiente cabecera TCP](capturas/37-tcp-ack-ipv6.png)

La cabecera TCP muestra puerto de origen 5000 y el número de confirmación (`ACKNOWLEDGEMENT NUMBER: 52`) que corresponde a los datos recibidos:

![Cabecera TCP del ACK: puerto origen 5000](capturas/38-tcp-ack-cabecera.png)

**Paquete de ida (PC4 → Server-tracker), visto en SW-intranet3.** Puerto de origen 1028 (temporal de PC4) y puerto de destino 5000 (relay):

![Cabecera TCP del paquete de ida: destino puerto 5000](capturas/39-tcp-ida-cabecera.png)

Los números de secuencia y de confirmación de ida (249/126) y de regreso (1/52) corresponden a los dos sentidos de la misma conexión TCP.

### 6.2 Flujo SNMP

**Solicitud SNMP de PC4 hacia R-bog, vista en SW-intranet3.** El paquete usa SNMP versión 1 sobre UDP y lleva la comunidad `redinterna` en texto claro:

![Paquete SNMP v1 con la comunidad redinterna](capturas/40-snmp-request.png)

Observación de seguridad: en SNMPv1 la comunidad viaja sin cifrar, por lo que cualquier equipo en el camino puede leerla. Por eso la gestión se restringe a la VLAN Internal con ACL, y SNMPv3 sería la mejora natural en un entorno real.

La respuesta de R-bog y el `set` se verifican en la sección 5.7 con el MIB Browser (get, set y get posterior).

---

## 7. Retos y cómo se resolvieron

| Reto | Solución |
|---|---|
| Los puertos seriales no existían | Se agregaron módulos WIC-2T (dos por router del Internet) y los puertos quedaron en `Se0/2/x` y `Se0/3/x` |
| El lado DCE de los cables seriales no coincidía con lo esperado | Se verificó con `clock rate` y el error "applies only to DCE" |
| Un router perdió su configuración al reiniciarse | Se repitió y desde entonces se guarda con `copy running-config startup-config` |
| El ping de R-bog a R-esp daba "U" | Faltaba la ruta por defecto en R-esp para el regreso |
| Los puertos de PCs de Madrid estaban en la VLAN 1 | SW-intranet5 no tenía el bloque de configuración aplicado; se reaplicó |
| Las impresoras no respondían ping desde otras VLANs | Les faltaba el gateway IPv6 |
| Los PCs de Madrid no resolvían `gdjm.net` | El MLSW no reparte DNS; se configuró a mano |
| Los eventos de JavaScript daban `undefined is not a function` | En Packet Tracer los callbacks se asignan como propiedades (`client.onReceive = function...`) |
| El Dashboard no recibía datos del relay | Se reenvía el mensaje de registro `DASH` periódicamente |
| GRE con IPsec cortó el IPv6 | Se volvió a `ipv6ip`, y IPsec cifra el tráfico IP entre las seriales |
| El MLSW de Madrid no respondía SNMP aunque sí ping | Se gestionó R-esp en su lugar (`snmp-server community redinterna RW`), con `get` y `set` desde PC7 |
| En Madrid, un PC de Guest también respondía SNMP | `MAD-FILTRO` solo filtraba HTTP/HTTPS; se añadió `deny udp` hacia el puerto SNMP para las VLAN 10 y 40, y se rehízo la ACL (el simulador no acepta números de secuencia en IPv6) |
| El `set` del MIB Browser daba "Bad value error" | Había que elegir el tipo `OctetString` en lugar de `BITS` |
| El MIB Browser no respondía con la comunidad por defecto | Se configura la comunidad `redinterna` y v1 en Advanced, en cada PC |

---

## 8. Conclusiones y recomendaciones

- **Verificar por capas ahorra tiempo.** Casi todos los fallos (puertos mal nombrados, rutas por defecto ausentes, bloques de configuración pegados en el equipo equivocado) se resolvieron comprobando primero el enlace, luego el direccionamiento, luego el enrutamiento y al final los servicios.
- **Guardar la configuración es parte del trabajo.** Una vez se perdió la configuración de R-bog al reiniciar sin `copy running-config startup-config`, y hubo que reaplicarla. Desde entonces se guarda después de cada bloque.
- **La VLAN nativa y los trunks se comprueban, no se suponen.** `show interfaces trunk` mostró que los tres enlaces usaban 802.1Q con la VLAN nativa 99 y todas las VLAN activas, y `show spanning-tree` confirmó que SW-intranet es la raíz.
- **IPsec sobre un túnel `ipv6ip` fue lo que funcionó.** Con GRE el simulador no transportó el IPv6 y, al añadir IPsec, se perdió incluso el ping IPv4. Con `ipv6ip` y una ACL que cifra el tráfico IP entre las dos seriales, el túnel quedó cifrado (181 paquetes cifrados y descifrados).
- **Las ACLs deben terminar en `permit ipv6 any any`.** Sin esa línea se rompen DNS, SLAAC y el propio Tracker. Los contadores de `show ipv6 access-list` fueron la prueba objetiva de que los filtros actúan.
- **El Tracker mostró las particularidades de la API de JavaScript de Packet Tracer:** los eventos son propiedades a las que se asigna una función, y el registro del Dashboard en el relay necesita reenviarse porque el relay puede no estar listo en el primer intento.
- **Las ACL se prueban con el caso que debe fallar.** El SNMP de Guest en Madrid respondía hasta que probamos con PC5 y añadimos la regla a `MAD-FILTRO`. Sin esa prueba habríamos entregado una ACL incompleta.
- **Límite observado:** el Multilayer Switch de Madrid no contestó SNMP aunque tenía la comunidad configurada y sí responde ping; no se pudo determinar la causa en el simulador, y se gestionó R-esp.

Recomendaciones:
- Implementar OSPFv3 o EIGRPv6 en lugar de rutas IPv6 estáticas.
- Resolver el SNMP del Multilayer Switch o desplegar un servidor de gestión dedicado, y usar SNMPv3.
- Cifrar las contraseñas (`service password-encryption`) y usar autenticación centralizada en producción.
- Añadir ACLs también en las VLAN 30 y 40.

---

## 9. Roles y contribuciones

| Integrante | Rol | Actividades específicas y verificables |
|---|---|---|
| Gabrielle Romandino | Configuración de red (Cisco) | Configuración de los equipos finales en Packet Tracer: PCs e impresoras (direccionamiento IPv6 por SLAAC y estático, gateways, DNS en los PCs de Madrid) y las aplicaciones Tracker App (PC4) y Tracker Dashboard (PC7) |
| Daniel Ortiz | Documentación | Elaboración de la documentación técnica del proyecto |
| Jerónimo Coy | Configuración de red (Cisco) | Configuración de routers, switches, servidores y Multilayer Switch: VLANs y trunks, enrutamiento (EIGRP/OSPF), túnel IPv6 e IPsec, ACLs, SNMP, NTP, DNS, web, DHCPv6 y Tracker Relay |
| Martín Guerrero | Pruebas | Ejecución y verificación de las pruebas de conectividad, servicios, ACLs, IPsec y Tracker |

---

## 10. Uso de herramientas de Inteligencia Artificial

**IA consultada:** Claude (Anthropic).

**Para qué se usó:**
- Apoyo en el diseño del direccionamiento IPv4 e IPv6.
- Generación de comandos de configuración de routers y switches.
- Generación del código JavaScript del Employee Tracker.
- Depuración de errores a partir de las salidas de los equipos.
- Borrador de la estructura de esta wiki.

**Cómo validamos las respuestas:** cada configuración propuesta se aplicó en Packet Tracer y se comprobó con ping, `show` y capturas antes de continuar. Cuando una respuesta falló, se pegó el error exacto y se corrigió.

Ejemplos concretos:
- Algunos puertos de la IA no coincidían con el simulador (puertos del MLSW, `clock rate`, módulos seriales), y se ajustaron al comprobar los reales con `show ip interface brief`.
- El código del Tracker tenía nombres de la API que no existían, y se corrigieron al probar `typeof` en el simulador.
- Un intento de IPsec sobre GRE cortó la conectividad, y se revirtió y rehízo sobre el túnel `ipv6ip`.
- La configuración final se revisó contra el `show running-config` real de los equipos, no contra lo propuesto.

---

## 11. Referencias (formato IEEE)

[1] S. Deering and R. Hinden, "Internet Protocol, Version 6 (IPv6) Specification," IETF, RFC 8200, Jul. 2017.
[2] R. Hinden and S. Deering, "IP Version 6 Addressing Architecture," IETF, RFC 4291, Feb. 2006.
[3] S. Thomson, T. Narten, and T. Jinmei, "IPv6 Stateless Address Autoconfiguration," IETF, RFC 4862, Sep. 2007.
[4] E. Nordmark and R. Gilligan, "Basic Transition Mechanisms for IPv6 Hosts and Routers," IETF, RFC 4213, Oct. 2005.
[5] S. Kent and K. Seo, "Security Architecture for the Internet Protocol," IETF, RFC 4301, Dec. 2005.
[6] J. Moy, "OSPF Version 2," IETF, RFC 2328, Apr. 1998.
[7] D. Mills et al., "Network Time Protocol Version 4: Protocol and Algorithms Specification," IETF, RFC 5905, Jun. 2010.
[8] "IEEE Standard for Local and Metropolitan Area Networks: Bridges and Bridged Networks," IEEE Std 802.1Q-2018.
[9] `[Material de clase, guía de diseño estructurado y documentación de Cisco Packet Tracer]`

---

## 12. Archivos entregados

- Configuraciones exportadas (`show running-config`) de los 13 equipos: R-bog, R-esp, ISP-bog, ISP-inet, ISP-fl, ISP-esp, MLSW, SW-dmz, SW-intranet, SW-intranet2, SW-intranet3, SW-intranet4 y SW-intranet5.
- Archivo del proyecto de Packet Tracer (`.pkt`).
- Video demostrativo de 10 minutos.
