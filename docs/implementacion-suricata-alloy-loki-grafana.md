# Implementación NIDS/NIPS con Suricata + Alloy, Loki y Grafana

En esta práctica se monta un laboratorio de seguridad con **Suricata** funcionando primero como IDS y después como IPS. Además, se añaden **Alloy, Loki y Grafana** para recoger y visualizar los logs de Suricata de una forma más clara.

La idea principal es simular una red con una máquina atacante, una máquina víctima y una máquina intermedia con Suricata. Todo el tráfico entre atacante y víctima pasa por Suricata, que puede detectar y bloquear conexiones según las reglas configuradas.

---

## 1. Arquitectura del laboratorio

Para esta práctica se han usado tres máquinas virtuales en IsardVDI. Todas usan **Ubuntu 24.04.3 LTS**.

| Máquina | Rol | IP principal |
|---|---|---|
| Atacante | Genera tráfico de prueba | `192.168.40.2/24` |
| Suricata | Router entre redes + IDS/IPS | `192.168.40.1/24` y `192.168.50.1/24` |
| Víctima | Recibe el tráfico de prueba | `192.168.50.2/24` |

La máquina Suricata tiene dos interfaces internas, una en cada red, por lo que puede actuar como router entre ambas subredes.

```text
Red atacante: 192.168.40.0/24
Red víctima:  192.168.50.0/24
```

---

## 2. Máquina atacante

La máquina atacante se usa para generar tráfico hacia la víctima y comprobar si Suricata detecta o bloquea correctamente.

Nombre usado en el laboratorio:

```text
dosorio-atacante
```

Interfaces:

- Default
- Wireguard VPN
- Personal1: `192.168.40.2/24`

Configuración netplan:

```yaml
network:
  ethernets:
    enp1s0:
      dhcp4: true
    enp2s0:
      dhcp4: true
    enp3s0:
      dhcp4: false
      addresses: [192.168.40.2/24]
      routes:
        - to: 192.168.50.0/24
          via: 192.168.40.1
  version: 2
```

Recursos asignados:

- RAM: 5 GB
- CPU: 3

---

## 3. Máquina Suricata

Esta máquina actúa como punto intermedio entre las dos redes. Primero enruta tráfico entre ambas subredes y después analiza ese tráfico con Suricata.

Nombre usado en el laboratorio:

```text
dosorio-suricata
```

Interfaces:

- Default
- Wireguard VPN
- Personal1: `192.168.40.1/24`
- Personal2: `192.168.50.1/24`

Configuración netplan:

```yaml
network:
  ethernets:
    enp1s0:
      dhcp4: true
    enp2s0:
      dhcp4: true
    enp3s0:
      dhcp4: false
      addresses: [192.168.40.1/24]
    enp4s0:
      dhcp4: false
      addresses: [192.168.50.1/24]
  version: 2
```

Recursos asignados:

- RAM: 6 GB
- CPU: 3

---

## 4. Máquina víctima

La máquina víctima sirve para recibir tráfico desde el atacante. De esta manera podemos comprobar si Suricata permite, detecta o bloquea las conexiones.

Nombre usado en el laboratorio:

```text
dosorio-victima
```

Interfaces:

- Default
- Wireguard VPN
- Personal2: `192.168.50.2/24`

Configuración netplan:

```yaml
network:
  ethernets:
    enp1s0:
      dhcp4: true
    enp2s0:
      dhcp4: true
    enp3s0:
      dhcp4: false
      addresses: [192.168.50.2/24]
      routes:
        - to: 192.168.40.0/24
          via: 192.168.50.1
  version: 2
```

Recursos asignados:

- RAM: 5 GB
- CPU: 2

---

## 5. Activación de IP forwarding

Para que la máquina Suricata pueda actuar como router entre ambas subredes, hay que activar el reenvío de paquetes IPv4.

Se edita el archivo:

```bash
sudo nano /etc/sysctl.conf
```

Y se descomenta o añade la siguiente línea:

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

Con esto, la máquina Suricata ya puede reenviar tráfico entre la red atacante y la red víctima.

---

## 6. Activación de nftables

En el servidor Suricata se activa `nftables`, que se usará más adelante para enviar tráfico a NFQUEUE.

```bash
sudo systemctl enable nftables
```

Comprobación del servicio:

```bash
systemctl status nftables
```

Salida esperada:

```text
Active: active (exited)
```

Esto confirma que nftables está instalado y habilitado.

---

## 7. Desactivación de offloading con ethtool

En Suricata es recomendable desactivar algunas funciones de offloading en las interfaces que inspeccionan tráfico. Esto evita problemas con checksums, segmentación y posibles comportamientos extraños al analizar paquetes.

Primero se instala `ethtool`:

```bash
sudo apt install -y ethtool
```

Después se desactiva offloading en las interfaces internas:

```bash
sudo ethtool -K enp3s0 gro off lro off gso off tso off
sudo ethtool -K enp4s0 gro off lro off gso off tso off
```

Con esto se reduce la posibilidad de falsos positivos o falsos negativos provocados por cómo el sistema procesa los paquetes antes de que Suricata los vea.

---

## 8. Verificación de conectividad inicial

Antes de instalar y configurar Suricata, se comprueba que las máquinas pueden comunicarse entre sí.

### 8.1 Ping desde atacante a víctima

Desde la máquina atacante:

```bash
ping 192.168.50.2
```

Resultado esperado:

```text
64 bytes from 192.168.50.2: icmp_seq=1 ttl=63 time=15.7 ms
64 bytes from 192.168.50.2: icmp_seq=2 ttl=63 time=4.18 ms
```

### 8.2 Ping desde víctima a atacante

Desde la máquina víctima:

```bash
ping 192.168.40.2
```

Resultado esperado:

```text
64 bytes from 192.168.40.2: icmp_seq=1 ttl=63 time=3.47 ms
```

Con estas pruebas se confirma que el enrutamiento entre subredes funciona correctamente.

---

## 9. Instalación de Suricata

En la máquina Suricata se instala el paquete principal y la herramienta de actualización de reglas.

```bash
sudo apt install suricata suricata-update
```

Después se actualizan las reglas:

```bash
sudo suricata-update
```

Esto descarga reglas actualizadas para Suricata.

---

## 10. Configuración inicial de Suricata en modo IDS

El archivo principal de configuración es:

```bash
sudo nano /etc/suricata/suricata.yaml
```

### 10.1 Definir HOME_NET

Dentro de `vars`, se define `HOME_NET` con las dos redes internas:

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.40.0/24,192.168.50.0/24]"
    EXTERNAL_NET: "!$HOME_NET"
```

Esto permite que Suricata identifique correctamente el tráfico interno del laboratorio.

### 10.2 Activar logs EVE JSON

En la sección `outputs`, se activa `eve-log` para generar eventos en formato JSON:

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

El archivo `eve.json` será el que después leerá Alloy para enviar los eventos a Loki.

### 10.3 Validar configuración

Se comprueba que el archivo de configuración no tiene errores:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

Salida correcta:

```text
Configuration provided was successfully loaded. Exiting.
```

### 10.4 Reiniciar y comprobar servicio

```bash
sudo systemctl restart suricata
systemctl status suricata
```

Salida esperada:

```text
Active: active (running)
```

En este punto, Suricata ya funciona como IDS.

---

## 11. Cambio de IDS a IPS con NFQUEUE

Una vez comprobado que Suricata funciona como IDS, se configura para trabajar como IPS. Para ello se usa NFQUEUE, que permite que nftables envíe paquetes a Suricata para que este decida si se aceptan o se bloquean.

### 11.1 Configuración de NFQUEUE

En `/etc/suricata/suricata.yaml`, se añade la configuración de NFQUEUE:

```yaml
nfqueue:
  - id: 0
    mode: accept
    fail-open: yes
```

También se evita que Suricata siga usando modos de captura incompatibles con esta configuración, como AF-PACKET, si están forzados en el servicio.

### 11.2 Validación de configuración

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

Salida esperada:

```text
Configuration provided was successfully loaded. Exiting.
```

---

## 12. Problema con el servicio Suricata y solución

Al intentar reiniciar Suricata en modo IPS, el servicio falló porque systemd seguía arrancando Suricata con parámetros de AF-PACKET. Eso impedía usar NFQUEUE correctamente.

### 12.1 Crear override de systemd

Primero se creó un override para modificar el comando de arranque:

```bash
sudo tee /etc/systemd/system/suricata.service.d/override.conf >/dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/bin/suricata -D -c /etc/suricata/suricata.yaml --pidfile /run/suricata.pid
