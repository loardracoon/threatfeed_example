# Firewall Class 102 - Student Book (V 0.2)

## 1. Introducción
Este material fue desarrollado para guiar al estudiante a través de un conjunto completo de actividades prácticas utilizando el Sophos Firewall en un entorno de laboratorio. Cada etapa fue revisada para garantizar claridad técnica, objetividad y consistencia operativa.

El contenido refleja situaciones reales de administración de firewall, incluyendo conectividad, seguridad, autenticación, inspección, SD-WAN, VPN y alta disponibilidad.

## 2. Objetivos del Entrenamiento
Esta guía tiene como objetivo:
* Aprender a inicializar y preparar correctamente un Sophos Firewall.
* Configurar zones, interfaces, DNS, DHCP y reglas fundamentales de seguridad.
* Implementar políticas de Firewall, NAT y prácticas de hardening.
* Integrar servicios como Threat Feeds, DNS Security, Web Filtering y SSL Inspection.
* Utilizar SD-WAN para failover, balanceo y políticas basadas en SLA.
* Registrar, gestionar y operar el firewall vía Sophos Central.
* Configurar y probar SSL-VPN, recursos avanzados y HA.
* Desarrollar experiencia práctica aplicable en entornos reales.

---

## 3. Información del Entorno

### Dispositivos de Red
* **Firewall-A**
    * **OS/Version:** XGS v21.5 (Factory Reset)
    * **IP Address:** 172.16.16.16/24
    * **Web Admin:** `https://172.16.16.16:4444`
* **Firewall-B**
    * **Versión:** XGS v21.5 (Factory Reset)
    * **IP Address:** 172.16.16.16/24
    * **Web Admin:** `https://172.16.16.16:4444`
* **Router (VyOS)**
    * **OS/Version:** VyOS 1.5
    * **IP Address:**
        * ETH0: DHCP 10.0.137.x/24
        * ETH1: 10.100.100.1/30
        * ETH2: 10.200.200.1/30
    * **Default GW:** 10.0.137.254 via ETH0
    * **Credenciales:** `vyos` | `vyos`

### Servidores y Clientes
* **User-01**
    * **IP Address:** DHCP (172.16.16.x/24), GTW 172.16.16.16
    * **Credenciales:** `.\Administrator` | `Sophos@1985*!`
* **DC-01**
    * **IP Address:** Static 192.168.101.10, GTW 192.168.101.1
    * **Credenciales:** `sophizo\Administrator` | `Sophos@1985*!`
* **WEB-01**
    * **IP Address:** Static 192.168.101.101, GTW 192.168.101.1
    * **Credenciales:** `eve` | `eve`
* **APP-1**
    * **IP Address:** Static 10.90.90.90, GTW 10.90.90.1
    * **Credenciales:** `root` | `no-password`

### Proveedores de Internet
* **ISP-1:** DHCP, 50Mbps bandwidth, 30ms latency, 0ms jitter.
* **ISP-2:** Static IP, 30Mbps bandwidth, 60ms latency, 5ms jitter.

---

## 4. Actividades Prácticas

### Preparación
1.  Para iniciar el laboratorio, active todos los nodos del entorno en el PNETLab (**Setup Nodes > Start up All Nodes**).
2.  Espere hasta que todos los nodos cambien de estado.
3.  **Importante:** Haga clic derecho sobre el **Firewall-B** y seleccione **Stop** (este no será utilizado inicialmente).
4.  Abra la consola del Router (parte superior de la topología).
5.  Pruebe la conectividad externa:
    ```bash
    ping 8.8.8.8
    ```
6.  Si falla, verifique la ruta por defecto (`run show ip route`) o agréguela:
    ```bash
    set protocols static route 0.0.0.0/0 next-hop 10.0.137.254
    commit
    ```
7.  Repita la prueba de ping. Si funciona, inicie el laboratorio.

---

### ACTIVIDAD 1: FIREWALL START-UP
1.  Acceda al computador **User-01**.
    * Username: `.\administrator`
    * Password: `Sophos@1985*!`
2.  En el navegador, abra `https://172.16.16.16:4444` (Ignore la alerta de certificado).
3.  Ejecute el Wizard inicial con los datos:
    * **Password do admin:** `Sophos@1985*!`
    * **MasterBoot Key:** `Sophos@1985*!`
    * **Nome do firewall:** `FW-STDNT-[ID]`
    * **Timezone:** São Paulo
    * **Registro:** “I don’t have a serial number (start a trial)”
    * **LAN IP:** `192.168.51.1/24`
    * **DHCP Range:** `192.168.51.100` – `192.168.51.200`
    * **Perfil de segurança:** Ninguno
4.  Después del reinicio, renueve la IP del User-01:
    ```powershell
    ipconfig /renew
    ```
5.  Verifique el acceso al Firewall en la nueva dirección: `https://192.168.51.1:4444`.

### ACTIVIDAD 2: BACKUP LOCAL DO FIREWALL
1.  Navegue hasta **System > Backup & Firmware > Backup & Restore**.
2.  Configure el Backup Local:
    * **Prefijo:** ej. “stdnt_”
    * **Frecuencia:** ej. semanal
    * **Contraseña de encriptación:** `Sophos@1985*!`
3.  Ejecute **“Backup Now”**. Cuando le pregunten si desea usar otra contraseña puntualmente, responda **no**.
4.  Descargue y guarde el archivo de backup.

### ACTIVIDAD 3: ZONES & INTERFACES
1.  Navegue hasta **Configure > Network > Zones** y agregue una nueva zona:
    * **Name:** MGMT
    * **Type:** DMZ
    * **Access:** Ping, SSH
2.  Acceda a **Network > Interfaces** y configure:

| Puerto | Zona | Label | IP Address | Gateway Name | Gateway IP |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PortA** | LAN | LAN-EMPLOYEES | 192.168.51.1/24 | - | - |
| **PortB** | WAN | WAN-INTERNET-A | DHCP (10.100.100.2/30) | INTERNET-A | 10.100.100.1 |
| **PortC** | WAN | WAN-INTERNET-B | Static (10.200.200.2/30) | INTERNET-B | 10.200.200.1 |
| **PortD** | DMZ | DMZ-SERVERS | 192.168.101.1/24 | - | - |
| **PortF** | MGMT | MGMT-HASYMC | 172.16.0.1/30 | - | - |

3.  En **System > Administration > Administrative Access**, garantice que todas las zonas tengan el servicio de **Ping** habilitado.
4.  Realice pruebas de conectividad desde:
    * User-01 hacia 192.168.51.1
    * Web-01 hacia 192.168.101.1
    * APP-01 hacia 10.100.100.2 y 10.200.200.2

### ACTIVIDAD 4: DNS E DNS REQUEST ROUTE
1.  User-01: Ping a `sophizo.local` (debe fallar).
2.  Firewall: **Configure > Network > DNS**.
    * Configure DNS IPv4: `8.8.8.8` y `8.8.4.4`.
    * **DNS Request Route:** Agregue ruta para el dominio `sophizo.local` apuntando a la IP `192.168.101.10`.
3.  **Monitor > Diagnostic**: Ping a `google.com` y `sophizo.local` (debe funcionar).

### ACTIVIDAD 5: DHCP SERVER
1.  **Configure > Network > DHCP**.
2.  Verifique los leases activos.
3.  Edite el DHCP Server predeterminado de la interfaz **LAN-EMPLOYEES**:
    * **Range:** 192.168.51.100 – 192.168.51.200
    * **DNS:** 192.168.51.1
4.  User-01: Renueve IP y pruebe:
    ```powershell
    ipconfig /renew
    ping [www.google.com](https://www.google.com)
    ping sophizo.local
    ```

### ACTIVIDAD 6: FIREWALL RULES
1.  **Protect > Rules and Policies > Firewall rules**.
2.  Analice y luego **elimine** todas las reglas de Firewall y NAT existentes.
3.  Repita las pruebas (todo debe fallar).
4.  Agregue las siguientes reglas (cree los objetos necesarios):

| Nombre | Origen | Destino | Servicio | Acción |
| :--- | :--- | :--- | :--- | :--- |
| **EMPLOYEES-to-INTERNET** | ZONE: LAN<br>ADDR: ANY | ZONE: WAN<br>ADDR: ANY | ANY | Permitir |
| **EMPLOYEES-to-SERVERS** | ZONE: LAN<br>ADDR: ANY | ZONE: DMZ<br>ADDR: ANY | ANY | Permitir |
| **SERVERS-to-EMPLOYEES** | ZONE: DMZ<br>ADDR: ANY | ZONE: LAN<br>ADDR: ANY | ANY | Permitir |
| **SERVERS-to-INTERNET** | ZONE: DMZ<br>ADDR: ANY | ZONE: WAN<br>ADDR: ANY | ANY | Permitir |

5.  Repita las pruebas.

### ACTIVIDAD 7: NAT RULES
1.  **Configure > Rule and Policies > NAT Rules**.
2.  Cree regla NAT:
    * **Source Zone:** LAN y DMZ
    * **Outbound Interface:** INTERNET-A y INTERNET-B
    * **Translated Source (SNAT):** MASQ
3.  Pruebe la conectividad (User-01 y Web-01).

### ACTIVIDAD 8: FIREWALL HARDENING
1.  Habilite **logging** en todas las reglas.
2.  Clone la regla `Employee-to-Servers` y edítela (colóquela arriba/clone above):
    * **Name:** Employee-to-WEB01
    * **Source:** Employee_Net
    * **Destination:** Web-01 (192.168.101.101)
    * **Services:** HTTP, HTTPS, ICMP
3.  Clone `Employee-to-Servers` nuevamente y edite:
    * **Name:** Employee-to-DC01
    * **Source:** Employee_Net
    * **Destination:** DC-01 (192.168.101.10)
    * **Services:** ICMP, DNS, SMB*, LDAP, LDAPS*, RDP*, Kerberos*
4.  Clone `Server-to-Internet` y edite:
    * **Name:** Web01-to-Internet
    * **Source:** Web-01
    * **Services:** ICMP, DNS, HTTP, HTTPS
5.  Edite la regla `Server-to-Employees`:
    * **Exclusion Source:** Web-01
6.  **Desactive** las reglas genéricas: `Server-to-Internet` y `Employee-to-Server`.
7.  Pruebe la conectividad desde User-01, DC-01 y Web-01.

### ACTIVIDAD 9: THREAT FEEDS
1.  Acceda al repositorio: `https://github.com/loardracoon/threatfeed_example`.
2.  Note los 3 archivos: `Domain.txt`, `Ipaddress.txt`, `url.txt`.
3.  En el Firewall: **Protect > Active Threat Response > Third Party Threat Feeds**.
4.  Inserte un nuevo feed para cada archivo (use el botón **RAW** en GitHub para obtener la URL):
    * **Action:** Block
    * **Position:** Top
    * **Polling Interval:** 5 min
5.  Realice pruebas (se espera bloqueo en los ítems de la lista):
    * `ping www.yahoo.com`
    * `ping www.uol.com`
6.  Solicite al instructor la adición de una nueva IP/Dominio y verifique la actualización.

### ACTIVIDAD 10: DNS SECURITY
1.  Sophos Central: **My Products > DNS Security > Installer**. Descargue el certificado.
2.  Firewall: Cambie el DNS del Sistema a `193.84.4.4` y `192.84.5.5`.
3.  User-01:
    * Acceda a `https://dns.access.sophos.com`.
    * Acceda a `https://www.sentinelone.com` (debe dar error de certificado).
4.  Importe el certificado descargado en el navegador (`chrome://certificate-manager`).
5.  Verifique si la página carga o es bloqueada según la categoría.

### ACTIVIDAD 11: USER-BASED POLICY
1.  **Configure > Authentication > Users**.
2.  Agregue usuario local:
    * **Username:** student
    * **Name:** Su Nombre
    * **Password:** `Sophos@1985*!`
    * **Group:** Open Group
3.  Acceda al Portal de Usuario (`https://192.168.51.1`) e inicie sesión.
4.  Edite la regla `Employees-to-Internet`:
    * Active **Match Known users**.
    * Active **Use web authentication for unknown users**.
    * **Group:** Open Group.
5.  Pruebe el acceso a `https://sophostest.com`.

### ACTIVIDAD 12: ACTIVE DIRECTORY SYNC
1.  **Configure > Authentication > Server**.
2.  Agregue servidor AD:
    * **Name:** DC-01 | **IP:** 192.168.101.10
    * **Connection:** Plaintext
    * **NetBIOS:** SOPHIZO | **Domain:** sophzio.local
    * **User:** ad-sync | **Pass:** `Sophos@1985*!`
    * **Query:** `OU=departments,DC=sophizo,DC=local`
3.  Pruebe la conexión y guarde.
4.  Haga clic en el botón **Importar** y seleccione los grupos: Sales, Engineer, HR, Remote.
5.  En **Authentication > Services**, coloque DC-01 en el tope de la lista de autenticación.
6.  Pruebe inicio de sesión en el Portal de Usuario (`https://192.168.51.1:4443`) con:

| Nombre | User | Grupo | Contraseña |
| :--- | :--- | :--- | :--- |
| Amy Fox | afox | Sales, Remote | `Sophos@1985*!` |
| Ben Cole | bcole | Engineer, Remote | `Sophos@1985*!` |
| Jay Lee | jlee | Engineer, Remote | `Sophos@1985*!` |
| Mia Ray | mray | hr | `Sophos@1985*!` |
| Tom Nash | tnash | sales | `Sophos@1985*!` |

### ACTIVIDAD 13: ACTIVE DIRECTORY SSO
*(Placeholder técnico para expansión futura)*

### ACTIVIDAD 14: WEB FILTERING
1.  **Protect > Web > User Activities**: Agregue "Lab User Activity" (Categories: News, Online Shopping, Social Network).
2.  **Web > URL Groups**: Agregue "Allowed URLs" (cnn.com, amazon.com, myspace.com).
3.  **Web > Policies**: Clone "Default Workplace Policy" a "Lab Web Policy".
    * Regla 1: Lab User Activity -> **Block HTTP**.
    * Regla 2: Lab Allowed URLs -> **Warn HTTP**.
4.  Aplique "Lab Web Policy" en la regla de firewall `Employees-to-Internet`.
5.  Active opciones de Scan y Decrypt.
6.  Pruebe accesos (msnbc, cnn, amazon, facebook, myspace).

### ACTIVIDAD 15: SSL INSPECTION
1.  **Web > URL Groups**: Edite “Local TLS Exclusion”, agregue `docs.sophos.com`, `id.sophos.com`.
2.  **SSL/TLS Inspection rules**: Agregue regla "Lab SSL TLS":
    * **Action:** Decrypt
    * **Profile:** Maximum Compatibility
    * **Source:** LAN / Employees_Net
3.  Acceda a Google (error de certificado esperado).
4.  Descargue CA en **System > Certificates > Certificate Authorities** e instale en el navegador.
5.  Pruebe acceso nuevamente.

### ACTIVIDAD 16: SD-WAN FAILOVER
1.  **Configure > Routing > SD-WAN Profiles**.
2.  Nuevo Perfil "INTERNET-FL":
    * **Strategy:** First available.
    * **Gateways:** INTERNET-A, INTERNET-B.
    * **SLA:** Best Quality (Latency).
3.  **Routing > SD-WAN Routes**:
    * Ruta "INTERNET-DEFAULT": Origem/Destino ANY, Profile "INTERNET-FL".
4.  User-01: Ping extendido a 8.8.8.8.
5.  Ejecute el script `ISP1-LATENCY` en el escritorio. Observe el cambio de enlace y logs.
6.  Ejecute `ISP1-RESET` para volver a la normalidad.
7.  Ajuste "Sample Size for SLA" a 5 y repita la prueba.

### ACTIVIDAD 17: SD-WAN LOAD BALANCING
1.  Cree SD-WAN Profile "INTERNET-LB":
    * **Strategy:** Load Balancing (Round-robin).
    * **Weights:** A=80 / B=20.
2.  Cree SD-WAN Route "APP1-TRAFFIC":
    * **Destination:** 10.90.90.90
    * **Profile:** INTERNET-LB.
3.  Pruebas vía `tracert` y WinMRT.
4.  Agregue latencia y pruebe el comportamiento.

### ACTIVIDAD 18: REMOTE FIREWALL MANAGEMENT
1.  Firewall: **System > Sophos Central**. Registre con sus credenciales.
2.  Active "Sophos Central services" y configure (Reports, Manage, Backups).
3.  Sophos Central: **My Products > Firewall Management > Firewalls**.
4.  Apruebe el firewall pendiente.
5.  Acceda a la consola del firewall remotamente por Central.

### ACTIVIDAD 19: FIREWALL GROUP
1.  Sophos Central: **Firewall Management > Dynamic Object > Interfaces**.
    * Cree objetos dinámicos para LAN, DMZ, WAN-A, WAN-B mapeando a los puertos del firewall `FW-STDNT-[ID]`.
2.  Cree **Firewall Group** "STDNT-[ID]".
3.  En el grupo, cree regla de Firewall "APP2-DROP" (LAN hacia WAN 10.90.90.90, Acción: Drop).
4.  Agregue su firewall al grupo y espere la sincronización (**Tasks Queue**).
5.  Verifique si la regla apareció en el firewall local.

### ACTIVIDAD 20: DESTINATION NAT / VIP
1.  En el Firewall Group, cree regla NAT "APP2-INBOUND".
    * **Original Dest:** 10.200.200.2
    * **Translated Dest:** 192.168.101.101 (Web-01)
    * **Interfaces:** Inbound (WAN-B), Outbound (DMZ).
2.  User-01: Cambie a la interfaz Ethernet-1 y pruebe acceso a `http://10.200.200.2`.

### ACTIVIDAD 21: SSL-VPN
1.  Consola Firewall (vía Central): **Configure > Remote Access VPN > SSL VPN**.
2.  Agregue "LAB_SSLVPN":
    * **Groups:** Remote.
    * **Resources:** 192.168.101.0/24.
    * **Tunnel mode:** Split tunnel.
3.  User-01: Acceda al Portal VPN (`https://192.168.51.1`), descargue config y conecte vía Sophos Connect.
4.  Pruebe acceso a `http://192.168.101.101`.

### ACTIVIDAD 22: ADVANCED SSL-VPN FEATURES
1.  User-01: Importe `sslvpn-advanced.pro` en Sophos Connect.
2.  Conecte y verifique el servidor conectado.
3.  Simule falla en el enlace principal (desactive eth1 en el Router).
4.  La VPN debe reconectar por el enlace secundario.

### ACTIVIDAD 23: HIGH AVAILABILITY
1.  Firewall-A: **Configure > System Service > High Availability**.
    * **Role:** Primary (active-passive).
    * **Mode:** quickHA.
    * **Port:** PortF.
2.  Encienda el **Firewall-B** en el PNETLab.
3.  Acceda a la CLI del FW-B, configure IP del PortA como `192.168.51.2/24`.
4.  Acceda a WebAdmin del FW-B (`https://192.168.51.2:4444`) y configure como **Standby** (QuickHA).
5.  Prueba de failover: Inicie un video en el User-01 y apague el Firewall-A. El video no debe detenerse.

## 5. Conclusión
Con la conclusión de este laboratorio, el estudiante recorrió todos los pilares fundamentales e intermedios de la administración del Sophos Firewall. La práctica ejecutada aquí ofrece una base sólida para entornos reales, preparación para certificaciones y evolución técnica en operaciones de seguridad de red.
