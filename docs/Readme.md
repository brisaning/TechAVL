Arquitectura Detallada para Sistema Web/Móvil con GPS en Tiempo Real

# 1. Diagrama de Componentes

| **Dispositivos** (GPS IoT)      | **Backend** (Procesamiento) | **Almacenamiento** | **Frontend/Móvil** |
|---------------------------------|-----------------------------|--------------------|---------------------|
| - MQTT/CoAP                     | - API Gateway               | - PostgreSQL       | - Web Dashboard     |
| - HTTP/2                        |   (Kong/Traefik)            |   (TimescaleDB)    |   (React + Mapbox)  |
| - Protocol Buffers              | - Microservicios:           | - Redis (Caché)    | - Mobile App        |
|                                 |   * Geolocation             | - Kafka (Logs)     |   (React Native)    |
|                                 |   * Alerts                  |                    |                     |
|                                 | - Flink/Spark (Streaming)   |                    |                     |

```mermaid
    flowchart LR
    subgraph Dispositivos_GPS_IoT["Dispositivos_GPS_IoT"]
            A["MQTT/CoAP"]
            B["HTTP/2"]
            C["Protocol Buffers"]
    end
    subgraph Backend_Procesamiento["Backend_Procesamiento"]
            D["API Gateway"]
            E["Microservicios: Geolocation"]
            F["Microservicios: Alerts"]
            G["Flink/Spark Streaming"]
    end
    subgraph Almacenamiento["Almacenamiento"]
            H["PostgreSQL"]
            I["Redis Cache"]
            J["Kafka Logs"]
    end
    subgraph Frontend_Movil["Frontend_Movil"]
            K["Web Dashboard"]
            L["Mobile App"]
    end
        A --> B
        B --> C
        C --> D
        D --> E & F & J & K & L
        E --> G & H
        F --> G & H
        G --> H & I
        I --> K & L
        J --> G
```

Este diagrama muestra cómo los diferentes componentes del sistema AVL se interconectan y colaboran:
* __Dispositivos (GPS IoT)__: Representa los dispositivos que envían datos, utilizando protocolos como MQTT/CoAP, HTTP/2 y Protocol Buffers para la comunicación eficiente.
* __Backend (Procesamiento)__: Esta sección central maneja la lógica de negocio y el procesamiento de datos.
	* La __API Gateway__ actúa como el punto de entrada principal.
	* Los __Microservicios__ (Geolocation, Alerts) se encargan de funciones específicas.
	* __Flink/Spark (Streaming)__ procesa datos en tiempo real.
* __Almacenamiento__: Aquí se gestionan los datos.
	* __PostgreSQL (con TimescaleDB)__ se usa para almacenar datos temporales y series de tiempo.
	* __Redis__ funciona como caché para un acceso rápido a los datos.
	* __Kafka__ se utiliza para gestionar logs y flujos de eventos.
* __Frontend/Móvil__: Permite la interacción del usuario.
	* El __Web Dashboard__ proporciona una interfaz de usuario basada en React y Mapbox para visualizar y gestionarla información.
	* La __Mobile App__ ofrece funcionalidades similares para dispositivos móviles, desarrollada con React Native.



## 1.1. Flujo de Datos

__Dispositivos GPS → MQTT Broker (EMQX/Mosquitto) → Kafka__ (cola de mensajes).

__Procesador en Tiempo Real (Flink)__ consume datos de Kafka → Filtra/Agrega → Almacena en PostgreSQL/Redis.

__API REST/GraphQL__ (Go/Elixir) sirve datos al __Frontend (React)__ y __App Móvil (React Native)__.

__WebSocket__ para actualizaciones en vivo en el mapa.

# 2. Estrategia para 50k → 500k Dispositivos

## 2.1 Escalabilidad Horizontal
* __MQTT Broker Clusterizado__ (EMQX):
	* Balanceo de carga con DNS Round Robin + Session Persistence.
	* Particionamiento por región geográfica (ej: topics gps/us-east, gps/eu-central).
* __Kafka__:
	* Aumento de partitions y consumers (Flink workers autoescalables).
	* Compresión de mensajes (Snappy/Zstandard).
* __API Gateway__:
	* Balanceo de carga con DNS Round Robin.
	* Rate limiting por dispositivo (100 req/s) usando Redis + Token Bucket.
* __Flink__:
	* Autoescalamiento de workers (Flink autoescalamiento).
	* Balanceo de carga con DNS Round Robin.
* __PostgreSQL__:
	* Autoescalamiento de workers (PostgreSQL autoescalamiento).
	* Compresión de datos (pg_bloat).
* __Redis__:
	* Autoescalamiento de workers (Redis autoescalamiento).
	* Compresión de datos (Redis compression).     

## 2.2 Optimización de Recursos
* __Edge Computing__:
	* Preprocesamiento en dispositivos (ej: enviar datos solo si hay movimiento > 50 metros).
* __Protocol Buffers__:
	* Reduce tamaño de payload en un 60% vs JSON.

# 3. Alta Disponibilidad (HA) y Recuperación ante Desastres (DR)

## 3.1 Alta Disponibilidad
* MQTT Broker:
	* Cluster en múltiples AZs con réplica sincrónica (EMQX Enterprise).
* Bases de Datos:
	* PostgreSQL: Streaming replication con 1 líder + 2 réplicas.
	* Redis: Cluster mode con sharding (16 particiones).
* Kubernetes:
	* Distribución de pods en ≥3 nodos (anti-affinity rules).

## 3.2 Disaster Recovery
* Backups Automatizados:
	* PostgreSQL: WAL-G + Snapshots en S3 (retención de 30 días).
	* Kafka: MirrorMaker a región secundaria (AWS us-east → us-west).
* Recuperación:
	* RTO < 15 min: Failover automático con Prometheus + Alertmanager.
	* RPO < 1 min: Replicación asíncrona con lag controlado.

# 4. Tecnologías Clave

| Capa               | Tecnología              | Razón de Elección                              |
|--------------------|-------------------------|------------------------------------------------|
| __Comunicación__   | MQTT + TLS 1.3          | Bajo Overheah, ideal para IoT                  |
| __Procesamiento__  | Apache Flink            | Estado consistente en ventanas de tiempo       |
| __Almacenamiento__ | TimescaleDB             | Escalabilidad para series temporales + PostGIS |
| __Frontend__       | React + Mapbox GL       | Renderizado rápido de mapas complejos          |
| __Movíl__          | React Nativa + Maplibre | Mismo código para iOS/Android                  |


# 5. Diagrama de Despliegue en AWS


**AWS Cloud**

| **Región us-east-1**       | **Región us-east-1** | **Región us-west-2** (DR) | **Región us-west-2** (DR)     |
|----------------------------|----------------------|---------------------------|-------------------------------|
| - VPC Publica:             | - VPC Privada:       | - VPC Privada:            | - S3 Cross-Region Replication |
|   * ALB                    |   * EKS Cluster      |   * RDS Replica           |                               |
|   * MQTT Brokers           |   * Kafka (MSK)      |   * Redis Cluster         |                               |
|   * NAT Gateway            |   * Flink            |                           |                               |



```mermaid
graph LR
    subgraph AWS_Cloud
        subgraph Region_us-east-1
            subgraph VPC_Publica_us_east_1
                A[ALB]
                B[MQTT Brokers]
                C[NAT Gateway]
            end

            subgraph VPC_Privada_us_east_1
                D[EKS Cluster]
                E[Kafka MSK]
                F[Flink]
            end

            A --> D
            B --> D
            C --> D
            D --> E
            D --> F
            E --> F
        end

        subgraph Region_us-west-2_DR
            subgraph VPC_Privada_us_west_2
                G[RDS Replica]
                H[Redis Cluster]
            end

            I[S3 Cross-Region Replication]
        end

        %% Conexiones entre regiones (simplificadas para el diagrama)
        F -- Data Sync --> G
        F -- Cache Sync --> H
        I -- Replicates To --> Region_us-east-1
    end
```