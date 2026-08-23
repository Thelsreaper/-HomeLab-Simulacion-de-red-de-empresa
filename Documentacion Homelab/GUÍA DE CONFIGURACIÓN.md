*GUÍA DE CONFIGURACIÓN POR SERVICIO*

**1. PROXMOX VE (Hipervisor)**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Interfaz web solo por HTTPS (certificado Let's Encrypt o autofirmado).
- Restringir acceso a la IP de gestión/VPN.
- Habilitar 2FA (TOTP) para usuarios administradores.
- No exponer puerto 8006 a Internet; acceder solo por VPN.
- Mantener sistema actualizado (apt update && apt full-upgrade).
- Usar cuentas separadas para admin y automatización (Ansible) con permisos mínimos.

***RENDIMIENTO:***
- ZFS con compresión lz4 o zstd.
- Habilitar KSM para deduplicar memoria entre VMs.
- Swappiness del host <= 10.
- Reservar al menos 2 vCPU y 4 GB RAM para el host.
- Usar virtio para discos y red en las VMs.

***REDUNDANCIA:***
- Backups automáticos con Proxmox Backup Server (PBS) o vzdump.
- Snapshots antes de cambios importantes.
- Monitorizar salud de discos con smartctl y alertas.
- (Preparado para replicación ZFS si se añade segundo nodo).


**2. DOCKER HOST (Grafana, Prometheus, DDNS, etc.)**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Contenedores con usuario no root.
- Redes internas Docker; evitar exponer puertos innecesarios.
- TLS en servicios web (Grafana, Prometheus).
- Acceso a Grafana/Prometheus solo por VPN o IP de administración.
- Imágenes actualizadas (docker compose pull && up -d).
- Secretos con Docker secrets o variables de entorno cifradas.
- Logging centralizado para auditoría.

***RENDIMIENTO:***
- Asignar 8 GB RAM y 4 vCPU al host.
- Limitar recursos por contenedor (mem_limit, cpus).
- Volúmenes persistentes en ZFS (SSD).
- Prometheus con retention.time adecuado.
- node_exporter en el host.

***REDUNDANCIA:***
- Guardar docker-compose.yml y .env en Git.
- Backups periódicos de volúmenes con restic.
- Reinicio automático (restart: unless-stopped).
- Snapshots del volumen de Prometheus.


**3. WAZUH**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Cambiar contraseñas por defecto (indexer, API, dashboard).
- Usar certificados TLS entre agentes, indexer y dashboard.
- Restringir acceso web solo VPN/IP gestión.
- Firewall: solo agentes autorizados a puertos 1514/1515.
- Autenticación fuerte y roles en el dashboard.

***RENDIMIENTO:***
- VM con 8 vCPU y 16 GB RAM.
- Índices en discos rápidos (SAS mirror o SSD).
- Heap de OpenSearch al 50% de la RAM.
- Retención de logs según capacidad (ej. 30 días).
- node_exporter en la VM.

***REDUNDANCIA:***
- Snapshots automáticos de índices (snapshot repository).
- Backups de configuración (/var/ossec/etc) y certificados.
- Preparado para cluster en el futuro.

**4. AMP CUBECODERS (LXC)**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Usuario no root para ejecutar AMP (usuario 'amp').
- Panel web protegido con Cloudflare Access o solo VPN.
- Restringir puertos de juegos en OpenWrt a IPs conocidas.
- Mantener AMP y servidores de juegos actualizados.
- Aislar LXC en VLAN de servicios sin acceso a gestión.

***RENDIMIENTO:***
- 8 vCPU y 16 GB RAM, ajustable.
- Archivos de juegos en discos rápidos (SAS mirror).
- CPU pinning en Proxmox si hay latencia.
- Desactivar servicios innecesarios en el LXC.

***REDUNDANCIA:***
- Backups del directorio /home/amp/.ampdata con restic.
- Snapshots del LXC antes de cambios.
- Documentar puertos y configuraciones en Git.

**5. MONITOREO (Prometheus + Grafana + ntopng)**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Autenticación en Grafana (OAuth o usuarios locales).
- HTTPS en Grafana.
- Prometheus accesible solo desde Grafana y red de gestión.
- ntopng con autenticación y HTTPS; no exponer a Internet.
- SNMP en OpenWrt con SNMPv3 (auth + cifrado), no v1/v2c.

***RENDIMIENTO:***
- Prometheus con 2-4 GB RAM y almacenamiento en SSD.
- Intervalos de scrape de 15-30s en node_exporter.
- ntopng con límite de captura (usar NetFlow/sFlow si es posible).
- Grafana con SQLite es suficiente.

***REDUNDANCIA:***
- Backups de configuración de Prometheus (reglas, alertas) y dashboards de Grafana.
- Snapshots del volumen de datos de Prometheus.
- Alertas configuradas (correo, Telegram, etc.).

**6. ANSIBLE**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Usuario dedicado 'ansible' con llaves SSH sin contraseña.
- Llave privada con passphrase y uso de ssh-agent.
- Repositorio Git privado en GitHub.
- Secretos cifrados con ansible-vault.
- Acceso a la VM solo por VPN.

***RENDIMIENTO:***
- VM con 1-2 vCPU y 2-4 GB RAM.
- Configurar forks en ansible.cfg para no saturar nodos.
- Usar --check antes de aplicar.

***REDUNDANCIA:***
- Repositorio Git como copia principal.
- git push después de cada cambio.
- Copia local de secretos cifrados en lugar seguro.

**7. PROXMOX BACKUP SERVER (PBS)**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Acceso web solo HTTPS y VPN.
- Usuario y contraseña fuerte; 2FA si es posible.
- Restringir puerto 8007 a red de gestión.
- Cifrado de backups (opcional).

***RENDIMIENTO:***
- 4 vCPU y 8 GB RAM.
- Almacenamiento en HDD de 1 TB (o disco NAS).
- Backups programados en horarios de baja actividad.
- Deduplicación activa.

***REDUNDANCIA:***
- Copias de la configuración y datastore del PBS a otro medio.
- Replicación remota con rclone (Cloudflare R2) si es posible.
- Verificación periódica de integridad de backups.

**8. WIREGUARD EN OPENWRT**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Claves Curve25519 generadas aleatoriamente.
- No compartir claves privadas; guardarlas en lugar seguro.
- Exponer solo puerto UDP 51820 en el firewall.
- Reglas restrictivas: desde zona wg solo permitir servicios concretos.
- Kill-switch en clientes para evitar fugas.

***RENDIMIENTO:***
- WireGuard es eficiente; sin ajustes especiales.
- MTU 1420 si hay fragmentación con muchos clientes.
- Monitorear tráfico con SNMP/ntopng.

***REDUNDANCIA:***
- Configuraciones de clientes y servidor en Ansible y Git.
- Backups de /etc/config/network, /etc/config/firewall y claves.
- Posible túnel de respaldo OpenVPN (opcional).

**9. OPENWRT (Router/Firewall)**
--------------------------------------------------------------------------------
***SEGURIDAD:***
- Cambiar contraseña root y desactivar SSH por contraseña (usar llaves).
- Firmware actualizado.
- Desactivar servicios innecesarios (Telnet, HTTP).
- Firewall con zonas y política por defecto denegar.
- DNSSEC y DNS sobre TLS/HTTPS si es posible.
- Acceso web restringido a VLAN de gestión.

***RENDIMIENTO:***
- Usar nftables (por defecto en OpenWrt moderno).
- Habilitar flow offloading si el hardware lo soporta.
- Limitar logging del firewall.

***REDUNDANCIA:***
- Backups automáticos de configuración con Ansible (fetch /etc/config).
- Copia de configuración en Git.
- Posible router de repuesto en frío.

**10.SEGURIDAD EN GENERALES**
--------------------------------------------------------------------------------
- Principio de mínimo privilegio en todos los servicios.
- Segmentar red con VLANs (gestión, servicios, usuarios, backups).
- VPN para todo acceso administrativo.
- 2FA donde sea posible (Proxmox, Grafana, Cloudflare Access).
- Inventario actualizado de puertos y servicios expuestos.
- Auditorías periódicas con nmap.
- Alertas de seguridad (intentos de acceso, cambios no autorizados).

***Extra para mejorar la redundancia***
--------------------------------------------------------------------------------

- Si se añade segundo servidor en el futuro: HA en Proxmox, replicación ZFS
  y balanceo de servicios.
