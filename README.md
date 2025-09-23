# SRE-Platform-Cloud
SRE for the rest of humans [Logs, opentelemetry, docker, proxy, etc]

Site reliability engineering (SRE) is an approach that applies software engineering principles to infrastructure and operations. Its core objective is to build and maintain highly scalable and reliable software systems. SRE is often viewed as a specific implementation of DevOps, a methodology that integrates software development and IT operations. Essentially, SRE is the "how-to" for putting DevOps' collaborative ideals into practice.


Here is the technical translation of the provided text, formatted for an IT environment.

-----


101-base: For single-host deployment, primarily used for technology evaluation and testing purposes.

101-cluster: For multi-host deployment, intended for production environments.

Ajuste de Variables
In both deployment types, it is mandatory to adjust the IP addresses and FQDN (Fully Qualified Domain Names) in the environment variables to match your specific setup.

----

### **Adjusting the Configuration**

```
chmod 666 /var/run/docker.sock
```

### **Configuring Grafana**

#### **Adding Prometheus as a Data Source**

1.  Navigate to **Grafana \> Configuration \> Data Sources**.
2.  Select **Type: Prometheus**.
3.  Set **URL: `http://prometheus:9090`**.
4.  Save the configuration.

#### **Adding Loki as a Data Source**

1.  Select **Type: Loki**.
2.  Set **URL: `http://loki:3100`**.
3.  Save the configuration.

#### **Importing Dashboards**

1.  Navigate to **Grafana \> Dashboards \> Import**.
2.  Use the following IDs to import the dashboards:
      * **cAdvisor Dashboard:** `193` (Currently shows no data—requires analysis)
      * **Loki Logs Dashboard:** `13639`
      * **Node Exporter Full:** `1860`
      * **Prometheus Alerts Dashboard:** `9578`

-----

### **Connection Diagram**

The configuration of your `docker-compose.yml` defines a comprehensive observability stack. The following diagram and detailed explanation break down the primary connections between these services.

**Overview**

This `docker-compose.yml` file sets up a monitoring and logging system, likely for a cluster or complex infrastructure. The key services include:

  * **Prometheus:** For metrics collection and storage.
  * **Grafana:** For metrics and log visualization.
  * **Loki:** For log aggregation and storage.
  * **Promtail:** For log collection and forwarding to Loki.
  * **Alertmanager:** For managing alerts based on Prometheus metrics.
  * **cAdvisor:** For collecting container metrics.
  * **node-exporter:** For collecting host system metrics.
  * **Consul:** For service discovery and configuration management.
  * **Nginx:** As a reverse proxy.
  * **consul-template:** For dynamic Nginx configuration generation.

**Connection Diagram**

This simplified diagram illustrates the primary connections among these services:

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

**Detailed Explanation of Connections**

1.  **Metrics:**

      * `node-exporter` and `cAdvisor` collect metrics from the host machine and containers, respectively, and expose them via HTTP.
      * `Prometheus` performs periodic scraping of these metrics from `node-exporter` and `cAdvisor`. It can also scrape metrics from other services.
      * `Grafana` queries `Prometheus` to visualize the metrics on dashboards.

2.  **Logs:**

      * `Promtail` reads logs from files (in this case, `/var/log` on the host) and ships them to `Loki`.
      * `Loki` stores the logs and provides an API for querying them.
      * `Grafana` queries `Loki` to display logs on dashboards.

3.  **Service Discovery and Dynamic Configuration:**

      * `Consul` acts as a service registry, enabling services to discover each other. It can also store configuration data.
      * `consul-template` queries `Consul` for service information and uses it to dynamically generate configuration files (in this case, for Nginx).
      * `Nginx` uses the configuration generated by `consul-template`. This is useful for load balancing, as Nginx can automatically update its configuration when services change.

4.  **Alerts:**

      * `Prometheus` evaluates alert rules based on the collected metrics.
      * `Alertmanager` receives alerts from Prometheus and manages them (e.g., deduplication, grouping, routing).

**Additional Key Points**

  * **Networking:** All services are connected to the `observability-net` network, enabling communication between them.
  * **Ports:** Some services (e.g., Prometheus, Grafana, Loki) expose ports to provide access to their web interfaces or APIs.
  * **Volumes:** Volumes are used to persist data (e.g., Consul data, Grafana configuration) and to share configuration files (e.g., between `consul-template` and Nginx).
  * **Dependencies:** The `depends_on` section in `docker-compose.yml` ensures that services start in the correct order (e.g., Grafana depends on Prometheus and Loki).
  * **Resources:** Resource limits and reservations (CPU and memory) are defined for each service, ensuring controlled resource allocation.
  * **Health Checks:** Health checks are defined to allow Docker to verify the status of services and restart them if they fail.

-----

```
+---------------------------------------------------------------------------------+
| External Users / Clients                                                        |
+----------------------------------+----------------------------------------------+
                                   | (HTTPS/HTTP)
                                   v
+----------------------------------+----------------------------------------------+
| F5 BIG-IP Load Balancer                                                         |
|  - Virtual Servers (VIPs)                                                       |
|  - Pools                                                                        |
|  - Pool Members (Automatically Managed) <----------+                            |
|  - Health Monitors (Optional, Consul is Primary)    |                            |
+----------------------------------+----------------------------------------------+
                                   | (Balanced traffic to healthy nodes/containers)
                                   v
+----------------------------------+----------------------------------------------+
| Cluster Network                                                                 |
+---------------------------------------------------------------------------------+
   |          |          |                      ^                  ^
   |          |          |                      | (API Call: AS3)  | (Consul API Read)
   v          v          v                      |                  |
+--------+ +--------+ +--------+      +---------------------+   +-----------------+
| Host 1 | | Host 2 | | Host N |      | F5 Automation       |   | Consul Servers  |
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
| cAdvisr| | cAdvisr| | cAdvisr|      | Consul Template     |---+ (Generates Config)
|--------| |--------| |--------|      +---------------------+   |
| Promtail| |Promtail| |Promtail|                               |
|--------| |--------| |--------|                                v
| App Ctr| | App Ctr| | App Ctr| <-----------------------+   +-----------------+
| (Vol.) | | (Vol.) | | (Vol.) | (Data Access)            |  | Prometheus      |<-+ (Scrape Targets)
+--------+ +--------+ +--------+                          |  |                 |       | (Alerts)
    |          |          |                               |  +-----------------+       |
    |          |          | (Mount)                       |         | (Metrics)   |    v
    v          v          v                               |         v               +----------+
+--------------------------+---------------------------+  |  +-----------------+    | Alertmgr |
| Shared Storage (NFS, GlusterFS, CephFS...)           |  |  | Grafana         |    +----------+
+------------------------------------------------------+  |  |                 |         | (Notifications)
                                                          |  +-----------------+         v (Email, Slack...)
                                                          |      ^ (Logs)
                                                          |      |
                                                          |  +-----------------+
                                                          +--| Loki            |
                                                             +-----------------+
```