# Ajusta la configuracion
	chmod 666 /var/run/docker.sock

Configurar Grafana
Agregar Prometheus como fuente de datos

## En Grafana > Configuration > Data Sources.
	Type: Prometheus.
	URL: http://prometheus:9090.
	Guardar.

## Agregar Loki como fuente de datos
	Type: Loki.
	URL: http://loki:3100.
	Guardar.

## Importar Dashboards
	En Grafana > Dashboards > Import.
	Usa estos IDs para importar:
	cAdvisor Dashboard: 193  -> actualmetne no presenta datos "analizar"
	Loki Logs Dashboard: 13639
	Node Exporte Full 1860
	Prometheus Alerts Dashboard: 9578



---

# Grafico de las conexiones

- Consulta realizada correctamente

Claro, vamos a desglosar la configuración de tu `docker-compose.yml` y crear una representación gráfica de las conexiones entre los servicios.

**Descripción General**

Tu archivo `docker-compose.yml` define un conjunto de servicios para la monitorización y registro de un sistema, probablemente un clúster o una infraestructura compleja.  Los servicios clave incluyen:

- **Prometheus:** Para la recopilación y el almacenamiento de métricas.
- **Grafana:** Para la visualización de las métricas.
- **Loki:** Para la agregación y el almacenamiento de logs.
- **Promtail:** Para la recolección de logs y el envío a Loki.
- **Alertmanager:** Para la gestión de alertas basadas en las métricas de Prometheus.
- **cAdvisor:** Para la recopilación de métricas de contenedores.
- **node-exporter:** Para la recopilación de métricas del sistema host.
- **Consul:** Para el descubrimiento de servicios y la configuración.
- **Nginx:** Como proxy inverso.
- **consul-template:** Para la generación dinámica de la configuración de Nginx.

**Diagrama de Conexiones**

A continuación, presento un diagrama simplificado que ilustra las conexiones principales entre estos servicios:

```
+-----------------+     +-----------------+     +-----------------+
| node-exporter   |---->|  Prometheus     |<----|    Grafana      |
+-----------------+     +-----------------+     +-----------------+
      ^   |                 ^   |                 ^
      |   +-----------------+   |                 |
      |   |  cAdvisor       |   |                 |
      |   +-----------------+   |                 |
      |                         |                 |
+-----------------+     +-----------------+     +-----------------+
|    Promtail     |---->|      Loki       |<----|    Grafana      |
+-----------------+     +-----------------+     +-----------------+
      ^                         ^
      |                         |
+-----------------+     +-----------------+
|     Logs        |     | consul-template |
|  (var/log)      |     +-----------------+
+-----------------+           ^    |
                              |    |
+-----------------+     +-----------------+
|    Nginx        |<----|     Consul      |
+-----------------+     +-----------------+
      ^
      |
+-----------------+
|  Alertmanager   |
+-----------------+
```

**Explicación Detallada de las Conexiones**

1. **Métricas:**
   - `node-exporter` y `cAdvisor` recopilan métricas de la máquina host y los contenedores, respectivamente, y las exponen a través de HTTP.
   - `Prometheus`  hace "scraping" (peticiones periódicas) de estas métricas desde `node-exporter` y `cAdvisor`.  Prometheus también puede recopilar métricas de otros servicios.
   - `Grafana` consulta a `Prometheus` para visualizar las métricas en dashboards.
2. **Logs:**
   - `Promtail` lee los logs de los archivos (en este caso, `/var/log` del host) y los envía a `Loki`.
   - `Loki` almacena los logs y proporciona una API para consultarlos.
   - `Grafana` consulta a `Loki` para mostrar los logs en los dashboards.
3. **Descubrimiento de Servicios y Configuración Dinámica:**
   - `Consul` actúa como un registro de servicios, permitiendo a los servicios descubrirse entre sí.  También puede almacenar datos de configuración.
   - `consul-template` consulta a `Consul` para obtener información y la utiliza para generar dinámicamente archivos de configuración (en este caso, la configuración de Nginx).
   - `Nginx` utiliza la configuración generada por `consul-template`. Esto es útil para balanceo de carga, ya que Nginx puede actualizar su configuración automáticamente cuando los servicios cambian.
4. **Alertas:**
   - `Prometheus` evalúa reglas de alerta basadas en las métricas recopiladas.
   - `Alertmanager` recibe las alertas de Prometheus y las gestiona (por ejemplo, deduplicación, agrupación, enrutamiento).

**Puntos Clave Adicionales**

- **Redes:** Todos los servicios están conectados a la red `observability-net`, lo que permite la comunicación entre ellos.
- **Puertos:** Algunos servicios (como Prometheus, Grafana, Loki) exponen puertos para acceder a sus interfaces web o APIs.
- **Volúmenes:** Se utilizan volúmenes para persistir datos (por ejemplo, los datos de Consul, la configuración de Grafana) y para compartir archivos de configuración (por ejemplo, entre `consul-template` y Nginx).
- **Dependencias:** La sección `depends_on` en `docker-compose.yml` asegura que los servicios se inicien en el orden correcto (por ejemplo, Grafana depende de Prometheus y Loki).
- **Recursos:** Se definen límites y reservas de recursos (CPU y memoria) para cada servicio, asegurando una asignación controlada de recursos.
- **Healthchecks:** Se definen healthchecks para que Docker pueda verificar el estado de los servicios y reiniciarlos si fallan.

Espero que esta explicación y el diagrama te sean útiles. Si tienes alguna otra pregunta, no dudes en consultarme.

Fuentes y contenido relacionado







+---------------------------------------------------------------------------------+
| Usuarios / Clientes Externos                                                    |
+----------------------------------+----------------------------------------------+
                                   | (HTTPS/HTTP)
                                   v
+----------------------------------+----------------------------------------------+
| F5 BIG-IP Load Balancer                                                         |
|  - Virtual Servers (VIPs)                                                       |
|  - Pools                                                                        |
|  - Pool Members (Gestionados Automáticamente) <----------+                      |
|  - Health Monitors (Opcional, Consul es primario)       |                      |
+----------------------------------+----------------------------------------------+
                                   | (Tráfico balanceado a nodos/contenedores sanos)
                                   v
+----------------------------------+----------------------------------------------+
| Red del Clúster                                                                 |
+---------------------------------------------------------------------------------+
   |          |          |                      ^                  ^
   |          |          |                      | (API Call: AS3)  | (Consul API Read)
   v          v          v                      |                  |
+--------+ +--------+ +--------+      +---------------------+   +-----------------+
| Host 1 | | Host 2 | | Host N |      | Automatización F5   |   | Consul Servers  |
|--------| |--------| |--------|      | (CTS / Script)      |   | (Cluster)       |
| Docker | | Docker | | Docker |      +---------------------+   +-----------------+
|--------| |--------| |--------|                                        ^ | (RPC/Gossip)
| Nomad  | | Nomad  | | Nomad  |-----------------+                      | v
| Client | | Client | | Client |                 | (Job Scheduling)  +-----------------+
|--------| |--------| |--------|                 +-------------------> | Nomad Servers   |
| Consul | | Consul | | Consul |-----------------+ (Service Reg/HC)    | (Cluster)       |
| Client | | Client | | Client |-------------------------------------> +-----------------+
|--------| |--------| |--------|                      ^ | (Consul Data)
| NodeExp| | NodeExp| | NodeExp|                      | |
|--------| |--------| |--------|      +---------------------+
| cAdvisr| | cAdvisr| | cAdvisr|      | Consul Template     |---+ (Genera Config)
|--------| |--------| |--------|      +---------------------+   |
| Promtail| |Promtail| |Promtail|                              |
|--------| |--------| |--------|                              v
| App Ctr| | App Ctr| | App Ctr| <-----------------------+  +-----------------+
| (Vol.) | | (Vol.) | | (Vol.) | (Acceso a datos)        |  | Prometheus      |<-+ (Scrape Targets)
+--------+ +--------+ +--------+                         |  |                 |  | (Alerts)
    |          |          |                              |  +-----------------+  |
    |          |          | (Mount)                      |      | (Metrics)   |  v
    v          v          v                              |      v             +----------+
+--------------------------+---------------------------+  |  +-----------------+ | Alertmgr |
| Almacenamiento Compartido (NFS, GlusterFS, CephFS...) |  |  | Grafana         | +----------+
+------------------------------------------------------+  |  |                 |   | (Notifications)
                                                          |  +-----------------+   v (Email, Slack...)
                                                          |      ^ (Logs)
                                                          |      |
                                                          |  +-----------------+
                                                          +--| Loki            |
                                                             +-----------------+
