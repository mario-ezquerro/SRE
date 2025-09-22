.PHONY: help up down register unregister get-ip

# Variable para obtener la IP del host
HOST_IP := $(shell hostname -I | awk '{print $$1}')

# Mostrar ayuda (objetivo por defecto)
help:
	@echo "🛠️  Makefile para gestionar Docker Compose y Consul"
	@echo ""
	@echo "Uso: make [comando]"
	@echo ""
	@echo "Comandos disponibles:"
	@echo "  help         Muestra esta ayuda"
	@echo "  up           Inicia Docker Compose y registra los servicios en Consul"
	@echo "  down         Detiene Docker Compose y elimina los registros en Consul"
	@echo "  register     Registra cAdvisor, Node Exporter y Promtail en Consul"
	@echo "  unregister   Elimina el registro de los servicios en Consul"
	@echo "  get-ip       Muestra la IP del host utilizada para el registro en Consul"
	@echo ""

# Obtener la IP del host
get-ip:
	@echo "🔄 Obteniendo IP del host..."
	@echo "✅ HOST_IP=$(HOST_IP)"

# Levantar los servicios con Docker Compose
up: get-ip
	@echo "🚀 Iniciando Docker Compose..."
	docker-compose up -d
	@make register

# Registrar servicios en Consul
register: get-ip
	@echo "Esperando a que Consul esté listo..." 
	#sleep 15;
	@until curl -sf $(HOST_IP):8500/v1/agent/self | jq '.Config' > /dev/null; do \
	    echo "Consul no está listo. Reintentando..."; \
	    sleep 5; \
	done

	@echo "Consul está listo."

	@echo "📝 Registrando servicios en Consul..."

	@curl -s -X PUT -H "Content-Type: application/json" --data \
		'{"ID": "cadvisor", "Name": "cadvisor", "Address": "$(HOST_IP)", "Port": 8000, "Tags": ["nginx", "monitoring", "metrics"]}' \
		http://localhost:8500/v1/agent/service/register

	@curl -s -X PUT -H "Content-Type: application/json" --data \
		'{"ID": "node-exporter", "Name": "node-exporter", "Address": "$(HOST_IP)", "Port": 9100, "Tags": ["nginx", "metrics"]}' \
		http://localhost:8500/v1/agent/service/register

	@curl -s -X PUT -H "Content-Type: application/json" --data \
		'{"ID": "promtail", "Name": "promtail", "Address": "$(HOST_IP)", "Port": 9080, "Tags": ["logs"]}' \
		http://localhost:8500/v1/agent/service/register

	@echo "✅ Servicios registrados en Consul."

# Detener los servicios y eliminar el registro de Consul
down:
	@echo "🛑 Deteniendo Docker Compose..."
	@make unregister
	docker-compose down
	@echo "🧹 Limpiando recursos..."
#	docker system prune -f

# Desregistrar servicios en Consul
unregister:
	@echo "❌ Eliminando registro de servicios en Consul..."
	@curl -s -X PUT http://localhost:8500/v1/agent/service/deregister/cadvisor
	@curl -s -X PUT http://localhost:8500/v1/agent/service/deregister/node-exporter
	@curl -s -X PUT http://localhost:8500/v1/agent/service/deregister/promtail
	@echo "✅ Servicios desregistrados de Consul."

