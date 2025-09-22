# Directorio con todo lo necesario para ser un cliente de un server y presentar de foram automática métricas en un grafana

Importante que el fichero .env está con las ip del servidor que esta como master y la ip local
         - cat .env
                MASTER_IP=<ip servidor que hace de master>
                WORKER_IP=<ip host actual>

La gestión se hace con el comando make, la configuración sin parámetros indica su funcionamiento.

                   Makefile para gestionar Docker Compose y Consul

                Uso: make [comando]

                Comandos disponibles:
                  help         Muestra esta ayuda
                  up           Inicia Docker Compose y registra los servicios en Consul
                  down         Detiene Docker Compose y elimina los registros en Consul
                  register     Registra cAdvisor, Node Exporter y Promtail en Consul
                  unregister   Elimina el registro de los servicios en Consul
                  get-ip       Muestra la IP del host utilizada para el registro en Consul


# Contenedores para su servicio

        Para su servicio se usa:
                - consul que será cliente del consul master
                - node-export métricas de host
                - cadvisor metricas de los docker
                - promtail para mandar los logs a el master

# Atención especial a las etiquetas para publicar en el consul

        En este ejemplo dentro del Makefile , esta indicado las "tag" que hay que usar

                - @curl -s -X PUT -H "Content-Type: application/json" --data \
                '{"ID": "node-exporter", "Name": "node-exporter", "Address": "$(HOST_IP)", "Port": 9100, "Tags": ["nginx", "metrics"]}' \
                http://localhost:8500/v1/agent/service/register

        Si se añade "metrics" el servicio será monitorizado por el prometheus de forma automática
        Si se añade "nginx" el servicio será  gestionado por el proxy inverso de forma automática.
