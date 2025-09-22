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
