# Implementación de NIDS/NIPS con Suricata, Alloy, Loki y Grafana

## 1. Objetivo de la práctica

El objetivo de esta práctica es montar un entorno de laboratorio con **Suricata** funcionando primero como IDS y después como IPS, usando **NFQUEUE** y **nftables** para inspeccionar y bloquear tráfico entre dos redes internas.

Además, se configura una pila de monitorización con **Alloy, Loki y Grafana** para visualizar los eventos generados por Suricata de una forma más clara y útil.

La idea principal es simular un escenario en el que una máquina atacante intenta comunicarse con una máquina víctima, mientras una tercera máquina actúa como router de seguridad y analiza el tráfico que pasa entre ambas redes.

---

## 2. Arquitectura del laboratorio

Para esta práctica se utilizan tres máquinas virtuales en Isard, todas con **Ubuntu 24.04.3 LTS**.

| Máquina | Función | IP principal | Red |
|---|---|---:|---|
| Atacante | Generar tráfico de prueba | `192.168.40.2` | `192.168.40.0/24` |
| Suricata | Router, IDS/IPS y monitorización | `192.168.40.1` / `192.168.50.1` | Ambas redes |
| Víctima | Recibir tráfico y servicios de prueba | `192.168.50.2` | `192.168.50.0/24` |

> Nota: si la arquitectura final de la práctica usa otras subredes, solo habría que adaptar las IPs en `netplan`, `HOME_NET`, `nftables` y las reglas locales.

---

## 3. Máquinas virtuales y recursos

### 3.1 Máquina atacante

Nombre de la máquina:

```text
dosorio-atacante
```

Interfaces de red:

- `Default`
- `Wireguard VPN`
- `Personal1` con IP `192.168.40.2`

Recursos asignados:

```text
RAM: 5 GB
CPU: 3
```

Configuración de red en `netplan`:

```yaml
network:
  ethernets:
    enp1s0:
      dhcp4: true
    enp2s0:
      dhcp4: true
    enp3s0:
      dhcp4: false
      addresses:
        - 192.168.40.2/24
      routes:
        - to: 192.168.50.0/24
          via: 192.168.40.1
  version: 2
```

Esta ruta permite que la máquina atacante pueda llegar a la red de la víctima pasando por el servidor Suricata.

---

### 3.2 Máquina Suricata

Nombre de la máquina:

```text
dosorio-suricata
```

Interfaces de red:

- `Default`
- `Wireguard VPN`
- `Personal1` con IP `192.168.40.1`
- `Personal2` con IP `192.168.50.1`

Recursos asignados:

```text
RAM: 6 GB
CPU: 3
```

Configuración de red en `netplan`:

```yaml
network:
  ethernets:
    enp1s0:
      dhcp4: true
    enp2s0:
      dhcp4: true
    enp3s0:
      dhcp4: false
      addresses:
        - 192.168.40.1/24
    enp4s0:
      dhcp4: false
      addresses:
        - 192.168.50.1/24
  version: 2
```

Esta máquina es la más importante del laboratorio, porque actúa como router entre ambas redes y también como sistema de detección y prevención.

---

### 3.3 Máquina víctima

Nombre de la máquina:

```text
dosorio-victima
```

Interfaces de red:

- `Default`
- `Wireguard VPN`
- `Personal2` con IP `192.168.50.2`

Recursos asignados:

```text
RAM: 5 GB
CPU: 2
```

Configuración de red en `netplan`:

```yaml
network:
  ethernets:
    enp1s0:
      dhcp4: true
    enp2s0:
      dhcp4: true
    enp3s0:
      dhcp4: false
      addresses:
        - 192.168.50.2/24
      routes:
        - to: 192.168.40.0/24
          via: 192.168.50.1
  version: 2
```

Esta ruta permite que la víctima pueda responder al atacante pasando también por Suricata.

---

## 4. Configuración de Suricata como router

Para que la máquina Suricata pueda reenviar tráfico entre las dos redes, es necesario activar el **IP forwarding**.

Se edita el archivo:

```bash
sudo nano /etc/sysctl.conf
```

Y se habilita esta línea:

```conf
net.ipv4.ip_forward=1
```

Después se aplican los cambios:

```bash
sudo sysctl -p
```

Salida esperada:

```text
net.ipv4.ip_forward = 1
```

Con esto, la máquina Suricata ya puede funcionar como router entre `192.168.40.0/24` y `192.168.50.0/24`.

---

## 5. Activación de nftables

En Ubuntu Server 24.04, nftables normalmente ya viene instalado. Aun así, se comprueba y se habilita el servicio:

```bash
sudo systemctl enable nftables
sudo systemctl status nftables
```

La salida debe mostrar el servicio como activo o habilitado:

```text
Active: active (exited)
```

Esto confirma que nftables está disponible para gestionar reglas de filtrado y, más adelante, enviar tráfico hacia NFQUEUE.

---

## 6. Ajustes de offloading con ethtool

En el servidor Suricata se instala `ethtool`:

```bash
sudo apt install -y ethtool
```

Después se desactivan algunas funciones de offloading en las interfaces internas:

```bash
sudo ethtool -K enp3s0 gro off lro off gso off tso off
sudo ethtool -K enp4s0 gro off lro off gso off tso off
```

Esto ayuda a evitar comportamientos extraños al analizar paquetes, ya que algunas optimizaciones de red pueden modificar la forma en la que los paquetes llegan a Suricata.

---

## 7. Verificación de conectividad inicial

Antes de instalar o configurar Suricata, se comprueba que las máquinas tienen conectividad entre redes.

### 7.1 Ping desde atacante a víctima

Desde la máquina atacante:

```bash
ping 192.168.50.2
```

Resultado esperado:

```text
64 bytes from 192.168.50.2
0% packet loss
```

### 7.2 Ping desde víctima a atacante

Desde la máquina víctima:

```bash
ping 192.168.40.2
```

Resultado esperado:

```text
64 bytes from 192.168.40.2
0% packet loss
```

Con estas pruebas se confirma que el enrutamiento funciona correctamente antes de añadir Suricata.

---

## 8. Instalación de Suricata en modo IDS

Se instala Suricata y la herramienta para actualizar reglas:

```bash
sudo apt install -y suricata suricata-update
```

Después se actualizan las reglas:

```bash
sudo suricata-update
```

Este primer modo de funcionamiento será IDS, es decir, Suricata detectará tráfico, pero todavía no lo bloqueará.

---

## 9. Configuración principal de Suricata

El archivo principal de configuración es:

```bash
sudo nano /etc/suricata/suricata.yaml
```

### 9.1 Configurar HOME_NET

En el apartado `vars`, se define `HOME_NET` con las dos redes internas del laboratorio:

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.40.0/24,192.168.50.0/24]"
    EXTERNAL_NET: "!$HOME_NET"
```

Esto es importante porque Suricata necesita saber qué redes se consideran internas para aplicar correctamente sus reglas.

---

### 9.2 Activar logs EVE JSON

Para poder enviar los eventos a Loki y visualizarlos después en Grafana, se activa la salida `eve-log` en formato JSON:

```yaml
outputs:
  - fast:
      enabled: yes
      filename: fast.log
      append: yes

  - eve-log:
      enabled: yes
      filetype: regular
      filename: /var/log/suricata/eve.json
```

El archivo `eve.json` será el que más adelante leerá Alloy.

---

### 9.3 Validar configuración

Antes de reiniciar el servicio, se valida la configuración:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

Resultado esperado:

```text
Configuration provided was successfully loaded. Exiting.
```

Si la validación es correcta, se reinicia Suricata:

```bash
sudo systemctl restart suricata
sudo systemctl status suricata
```

El servicio debe aparecer como:

```text
Active: active (running)
```

---

## 10. Cambio de IDS a IPS con NFQUEUE

Una vez comprobado que Suricata funciona como IDS, se cambia a modo IPS. Para ello se utiliza **NFQUEUE**, que permite que nftables envíe paquetes a Suricata y que Suricata decida si los acepta o los descarta.

### 10.1 Configuración de NFQUEUE en Suricata

En `suricata.yaml`, se añade la configuración de NFQUEUE:

```yaml
nfqueue:
  - id: 0
    mode: accept
    fail-open: yes
```

El identificador `0` representa la cola que usaremos desde nftables.

---

### 10.2 Problema con AF-PACKET y systemd

Durante la configuración apareció un problema: el servicio de Suricata seguía arrancando con `--af-packet` desde systemd, lo cual no encajaba con el uso de NFQUEUE.

Para solucionarlo se creó un override del servicio:

```bash
sudo mkdir -p /etc/systemd/system/suricata.service.d
```

Y después:

```bash
sudo tee /etc/systemd/system/suricata.service.d/override.conf >/dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/suricata -D -c /etc/suricata/suricata.yaml -q 0 --runmode autofp --pidfile /run/suricata.pid
EOF
```

Con esto, Suricata arranca usando la cola NFQUEUE 0.

---

### 10.3 Cargar el módulo nfnetlink_queue

Se carga el módulo necesario del kernel:

```bash
sudo modprobe nfnetlink_queue
```

Para aplicar el override:

```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed suricata
sudo systemctl restart suricata
sudo systemctl status suricata
```

Resultado esperado:

```text
Active: active (running)
ExecStart=/usr/bin/suricata -D -c /etc/suricata/suricata.yaml -q 0
```

Con esto, Suricata ya queda preparada para trabajar como IPS.

---

## 11. Configuración de nftables para NFQUEUE

Se edita el archivo principal de nftables:

```bash
sudo nano /etc/nftables.conf
```

Contenido configurado:

```nft
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0;
    policy accept;
  }

  chain forward {
    type filter hook forward priority 0;
    policy accept;

    # Tráfico del atacante hacia la víctima
    ip saddr 192.168.40.0/24 ip daddr 192.168.50.0/24 counter queue num 0

    # Tráfico de la víctima hacia el atacante
    ip saddr 192.168.50.0/24 ip daddr 192.168.40.0/24 counter queue num 0
  }
}
```

Se cargan las reglas:

```bash
sudo nft -f /etc/nftables.conf
```

Y se comprueban:

```bash
sudo nft list ruleset
```

Con esto, todo el tráfico entre ambas redes pasa por la cola 0, donde Suricata puede inspeccionarlo.

---

## 12. Verificación con NFQUEUE activo

Antes de crear reglas de bloqueo, se comprueba que todavía hay conectividad:

```bash
ping 192.168.50.2
```

Si hay respuesta, significa que nftables está enviando el tráfico a NFQUEUE, pero Suricata todavía no lo está bloqueando.

---

## 13. Reglas locales de Suricata

Se edita el archivo de reglas locales:

```bash
sudo nano /etc/suricata/rules/local.rules
```

Contenido añadido:

```conf
# ALERTAS
alert icmp any any -> any any (msg:"ICMP detectado"; sid:1000001; rev:1;)
alert tcp any any -> any 80 (msg:"Conexion HTTP detectada"; sid:1000002; rev:1;)

# BLOQUEOS
drop icmp any any -> any any (msg:"ICMP bloqueado"; sid:1000003; rev:1;)
drop tcp 192.168.40.0/24 any -> 192.168.50.0/24 22 (msg:"SSH bloqueado desde atacante"; sid:1000004; rev:1;)
```

Después se reinicia Suricata:

```bash
sudo systemctl restart suricata
```

---

## 14. Activar local.rules en suricata.yaml

Se comprueba si `local.rules` ya está incluido:

```bash
sudo grep -n "local.rules" /etc/suricata/suricata.yaml
```

Si no aparece, se añade:

```yaml
default-rule-path: /etc/suricata/rules

rule-files:
  - local.rules
```

Después se valida:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

Resultado esperado:

```text
4 rules successfully loaded, 0 rules failed
```

Y se reinicia el servicio:

```bash
sudo systemctl restart suricata
sudo systemctl status suricata
```

---

## 15. Pruebas de bloqueo y logs

### 15.1 Prueba ICMP

Desde la máquina atacante:

```bash
ping 192.168.50.2
```

Resultado esperado:

```text
100% packet loss
```

En Suricata se revisa el log:

```bash
sudo tail -f /var/log/suricata/fast.log
```

Salida esperada:

```text
[Drop] [1:1000003:1] ICMP bloqueado
[1:1000001:1] ICMP detectado
```

Esto confirma que Suricata detecta y bloquea tráfico ICMP.

---

### 15.2 Prueba SSH

Desde el atacante:

```bash
ssh usuario@192.168.50.2
```

Resultado esperado:

```text
Connection timed out
```

En el log:

```text
[Drop] [1:1000004:1] SSH bloqueado desde atacante
```

---

### 15.3 Prueba HTTP

Desde el atacante:

```bash
curl http://192.168.50.2
```

En el log debe aparecer:

```text
[1:1000002:1] Conexion HTTP detectada
```

Con estas pruebas se confirma que las reglas locales funcionan correctamente.

---

## 16. Instalación de Alloy, Loki y Grafana

Una vez Suricata funciona como IPS, se configura la monitorización para visualizar los logs.

### Herramientas utilizadas

- **Alloy:** recolector de logs.
- **Loki:** almacén de logs.
- **Grafana:** visualización mediante dashboards.

La idea es que Suricata genere eventos en `eve.json`, Alloy los lea, Loki los almacene y Grafana los muestre.

---

## 17. Instalación del repositorio de Grafana

Se instala `gpg`:

```bash
sudo apt install -y gpg
```

Se añade la clave GPG:

```bash
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /usr/share/keyrings/grafana.gpg > /dev/null
```

Se añade el repositorio:

```bash
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

Se actualiza APT:

```bash
sudo apt update
```

---

## 18. Instalación de Alloy

```bash
sudo apt install -y alloy
```

---

## 19. Instalación y arranque de Grafana

```bash
sudo apt install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

El servicio debe aparecer como activo.

Grafana queda disponible en:

```text
http://192.168.50.1:3000
```

Credenciales iniciales:

```text
Usuario: admin
Contraseña: admin
```

En el primer acceso se debe cambiar la contraseña por una segura.

> Por seguridad, no se debe documentar la contraseña final real.

---

## 20. Instalación y configuración de Loki

Se instala Loki:

```bash
sudo apt install -y loki
```

Se edita su configuración:

```bash
sudo nano /etc/loki/config.yml
```

Se comprueba que tenga configurado el puerto HTTP:

```yaml
server:
  http_listen_port: 3100
  grpc_listen_port: 9096
```

Se reinicia el servicio:

```bash
sudo systemctl restart loki
sudo systemctl status loki
```

Resultado esperado:

```text
Active: active (running)
```

---

## 21. Configuración de Alloy

Se edita la configuración de Alloy:

```bash
sudo nano /etc/alloy/config.alloy
```

Contenido configurado:

```alloy
loki.write "default" {
  endpoint {
    url = "http://127.0.0.1:3100/loki/api/v1/push"
  }
}

loki.source.file "suricata" {
  targets = [
    {
      __path__ = "/var/log/suricata/eve.json",
      job      = "suricata-logs",
      host     = "dosorio-suricata",
      source   = "evejson",
    },
  ]

  forward_to = [loki.write.default.receiver]
}
```

Se reinicia Alloy:

```bash
sudo systemctl restart alloy
sudo systemctl status alloy
```

---

## 22. Permisos para leer eve.json

Para que Alloy pueda leer el log de Suricata, se instalan las ACL:

```bash
sudo apt install -y acl
```

Se dan permisos al usuario `alloy`:

```bash
sudo setfacl -m u:alloy:r /var/log/suricata/eve.json
sudo setfacl -m u:alloy:rx /var/log/suricata
```

Con esto, Alloy puede leer el archivo `eve.json` y enviarlo a Loki.

---

## 23. Configuración de Loki como datasource en Grafana

Se accede a Grafana desde el navegador:

```text
http://192.168.50.1:3000
```

Después:

1. Entrar con el usuario administrador.
2. Ir a **Connections**.
3. Seleccionar **Data sources**.
4. Añadir un nuevo datasource.
5. Seleccionar **Loki**.
6. Configurar la URL:

```text
http://localhost:3100
```

7. Pulsar **Save & Test**.

Si la conexión es correcta, Grafana ya puede consultar los logs almacenados en Loki.

---

## 24. Creación del dashboard

En Grafana:

1. Ir a **Dashboards**.
2. Seleccionar **Create dashboard**.
3. Añadir una visualización.
4. Seleccionar Loki como fuente de datos.
5. Crear consultas para visualizar eventos de Suricata.

El dashboard creado se llamó:

```text
Suricata-Monitorización
```

El objetivo del dashboard es ver de forma más clara los eventos detectados y bloqueados, como ICMP, SSH o HTTP.

---

## 25. Mejora de visualización

Después de crear el primer dashboard, se ajustaron los paneles para que fueran más intuitivos.

Se añadieron visualizaciones orientadas a:

- paquetes bloqueados por minuto
- eventos por tipo
- alertas ICMP
- bloqueos SSH
- eventos HTTP
- actividad general de Suricata

Con esto, la monitorización pasó de depender solo de logs en terminal a tener una vista gráfica más clara en Grafana.

---

## 26. Problemas encontrados y soluciones aplicadas

| Problema | Causa | Solución |
|---|---|---|
| Suricata seguía arrancando en AF-PACKET | El servicio systemd forzaba `--af-packet` | Se creó un override de systemd |
| Suricata no arrancaba en modo NFQUEUE | No se indicaba la cola `-q 0` | Se añadió `-q 0` en `ExecStart` |
| NFQUEUE no funcionaba | Faltaba módulo del kernel | Se cargó `nfnetlink_queue` |
| Las reglas locales no se aplicaban | `local.rules` no estaba incluido en `suricata.yaml` | Se añadió en `rule-files` |
| Alloy no podía leer `eve.json` | Falta de permisos | Se aplicaron ACL con `setfacl` |
| Los logs eran difíciles de interpretar | Solo se revisaban en terminal | Se configuró Loki y Grafana |

---

## 27. Comprobaciones finales

Al finalizar, se comprobó:

- Conectividad inicial entre redes.
- Funcionamiento de Suricata como router.
- Suricata activo como IDS.
- Suricata funcionando como IPS con NFQUEUE.
- nftables enviando tráfico a la cola 0.
- Bloqueo de ICMP.
- Bloqueo de SSH.
- Detección de HTTP.
- Generación de logs en `fast.log` y `eve.json`.
- Alloy leyendo logs de Suricata.
- Loki recibiendo eventos.
- Grafana mostrando información en dashboards.

---

## 28. Resultado final

El laboratorio quedó funcionando correctamente con Suricata actuando como **NIDS/NIPS**, inspeccionando tráfico entre dos redes internas y bloqueando eventos definidos mediante reglas locales.

Además, se configuró una pila de observabilidad formada por **Alloy, Loki y Grafana**, lo que permitió visualizar los logs de Suricata de una forma más clara y útil.

Con esto, la práctica no solo permite detectar tráfico sospechoso, sino también bloquearlo y analizarlo desde un panel gráfico.

---

## 29. Conclusión técnica

Esta práctica permitió trabajar con un escenario realista de seguridad de red, donde una máquina intermedia actúa como router y sistema de defensa entre dos redes.

Lo más importante fue entender la diferencia entre IDS e IPS:

- En modo **IDS**, Suricata detecta y registra eventos.
- En modo **IPS**, Suricata puede tomar decisiones y bloquear tráfico usando NFQUEUE.

También fue importante configurar correctamente nftables, systemd y los permisos de logs, ya que pequeños detalles como un parámetro incorrecto en el servicio o un archivo de reglas no incluido pueden impedir que el sistema funcione correctamente.

La integración con Alloy, Loki y Grafana mejoró la parte de monitorización, haciendo que los eventos fueran más fáciles de consultar y analizar.
