Arquitectura Detallada para Sistema Web/Móvil con GPS en Tiempo Real
1. Diagrama de Componentes

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


Flujo de Datos

    Dispositivos GPS → MQTT Broker (EMQX/Mosquitto) → Kafka (cola de mensajes).

    Procesador en Tiempo Real (Flink) consume datos de Kafka → Filtra/Agrega → Almacena en PostgreSQL/Redis.

    API REST/GraphQL (Go/Elixir) sirve datos al Frontend (React) y App Móvil (React Native).

    WebSocket para actualizaciones en vivo en el mapa.