 ***HomeLab Simulacion de red de empresa***

***1. HARDWARE DISPONIBLE***
--------------------------------------------------------------------------------
- Servidor: DELL PowerEdge (compatible con Proxmox)
- CPU: 2x Intel Xeon E5-2680v3 (24 núcleos / 48 hilos)
- RAM: 128 GB DDR4
- Controladora: DELL PERC H730 1GB (soporta RAID y modo HBA/JBOD)
- Red: 4x 1 GbE Broadcom 5720
- Discos:
    * SSD Kingston 512 GB (sistema y VMs de alto rendimiento)
    * 2x HDD SAS 1.2 TB 10K (datos, mirror)
    * HDD SATA 1 TB (backups, disco portátil no recomendado 24/7)
- Gestión remota: iDRAC8 Enterprise
- Router: OpenWrt (sin switch gestionable adicional)

***2. Orden de almacenamiento***
--------------------------------------------------------------------------------
Opción recomendada: Modo HBA en controladora + ZFS

Pools:
  - rpool  -> SSD Kingston 512 GB
      * Proxmox (sistema)
      * VMs/LXCs que requieren velocidad (Docker host, Wazuh indexer, Ansible)
  - datos  -> Mirror RAID1 con 2x SAS 1.2 TB
      * Volúmenes grandes (AMP, logs Wazuh, datos de monitoreo)
  - backup -> HDD SATA 1 TB
      * Destino para Proxmox Backup Server (PBS) y restic

Nota: Me encuentro dudado si usar el controlador Raid así que esto podría cambiar

***3. ASIGNACIÓN DE RECURSOS POR SERVICIO***
--------------------------------------------------------------------------------

| Servicio             | Tipo           | vCPU | RAM   | Disco       | Notas                                |
|----------------------|----------------|:----:|:-----:|-------------|--------------------------------------|
| Docker host          | VM Debian 12   | 4    | 8 GB  | 100 GB SSD  | Grafana, Prometheus, DDNS            |
| Wazuh                | VM Debian 12   | 8    | 16 GB | 200 GB SAS  | Indexer + server + dashboard         |
| AMP CubeCoders       | LXC Debian 12  | 8    | 16 GB | 200 GB SAS  | Servidores de juegos                 |
| ntopng               | LXC/VM         | 2    | 4 GB  | 60 GB SAS   | Recibe NetFlow                       |
| Ansible              | VM/LXC         | 2    | 4 GB  | 20 GB SSD   | Gestión y automatización             |
| Proxmox Backup Server| VM             | 4    | 8 GB  | 500 GB HDD  | Backups                              |
| Reserva              | -              | -    | ~70 GB| -           | Para futuros servicios               |

**Total estimado:** 28 vCPU, 56 GB RAM, ~1.1 TB (sin backups).  
**Nota:** Quedan aproximadamente 70 GB de RAM libres para expansión.


***4. SERVICIOS Y SU UBICACIÓN***
--------------------------------------------------------------------------------
- DHCP y DNS        : OpenWrt (dnsmasq) 
- DDNS Cloudflare   : OpenWrt (ddns-scripts) o Docker (oznu/cloudflare-ddns)
- Wazuh             : VM dedicada, stack completo
- AMP CubeCoders    : LXC dedicado, instalación directa (sin Docker)
- Monitoreo         : Prometheus + Grafana en Docker host
                      node_exporter en todos los Linux
                      OpenWrt con SNMP + snmp_exporter
                      ntopng en LXC con NetFlow desde OpenWrt
- VPN               : WireGuard en OpenWrt (road-warrior y site-to-site)
- Ansible           : VM dedicada, repositorio Git en GitHub
- Backups           : PBS en VM + restic para configuraciones

***5. EXPOSICIÓN A INTERNET (SOLO AMP)***
--------------------------------------------------------------------------------
Panel web de AMP:
  - Usar Cloudflare Tunnel (cloudflared) con acceso protegido por Cloudflare Access.
  - No se abriran puertos del panel en el router.

Puertos de juegos:
  - Abrir solo los puertos TCP/UDP necesarios en OpenWrt y redirigir al LXC de AMP.

Nota: Todo lo demás (Proxmox, Wazuh, Grafana, etc.) solo accesible por VPN.

***6. BACKUPS***
--------------------------------------------------------------------------------
1. Proxmox Backup Server (PBS) como VM usando el HDD de 1 TB.
   - Backups programados de todas las VMs/LXCs.
2. Backups de archivos de configuración con restic/borg desde Ansible:
   - Configuraciones de OpenWrt, Docker compose, playbooks, etc.
   - Destino local en HDD backup y opcionalmente en Cloudflare R2.
3. Copia fuera de casa:
   - Usar bucket S3 compatible (Cloudflare R2) con restic para copias cifradas.
   - O disco USB externo si hay puerto disponible.

***7. DISEÑO DE VPN CON WIREGUARD EN OPENWRT***
--------------------------------------------------------------------------------
- Zona de firewall separada: "wg"
- Road-warrior: acceso desde pc/Telefono.
- Site-to-site: túnel entre OpenWrt y una VM en el servidor.
- Puerto UDP 51820 expuesto en OpenWrt ("único puerto abierto a internet").
- Reglas: solo tráfico necesario desde wg hacia LAN, denegar el resto.

***8. ORDEN DE IMPLEMENTACIÓN***
--------------------------------------------------------------------------------
1.  Instalar Proxmox en SSD (ZFS si se usa HBA).
2.  Configurar red en Proxmox (bridges/VLANs).
3.  Crear VMs/LXCs base Debian 12 y configurar SSH con llaves.
4.  Configurar OpenWrt: DHCP, DNS, DDNS, VLANs, firewall, WireGuard.
5.  Levantar Docker host: Grafana, Prometheus, node_exporter, DDNS.
6.  Instalar Wazuh en VM dedicada.
7.  Instalar AMP en LXC y exponer solo lo necesario.
8.  Desplegar ntopng con NetFlow desde OpenWrt.
9.  Configurar PBS y programar backups.
10. Automatizar con Ansible y subir playbooks a GitHub.

***9. Futuras mejoras ***
--------------------------------------------------------------------------------
- Adquirir un switch gestionable (8 puertos) para facilitar VLANs, bonding y
  port mirror para ntopng.
- Sustituir el HDD de 1 TB portátil por un disco NAS/SSD para backups 24/7.
- Usar iDRAC8 para monitoreo de hardware (temperaturas, ventiladores, alertas).
- Usar ansible-vault para secretos (API keys, contraseñas).
