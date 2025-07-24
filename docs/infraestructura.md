# Infraestructura Cloud para Sistema AVL Escalable

## 1. Proveedor de Nube

Elección: AWS (por madurez en servicios IoT y escalabilidad geográfica)
Alternativas: Google Cloud (BigQuery para analytics) o Azure (IoT Hub).

| Servicio AWS Clave       | Uso                                | Costo Estimado (50k dispositivos/mes)      |
|--------------------------|------------------------------------|--------------------------------------------|
| **EC2 Auto Scaling**     | Microservicios (Go, Flink)         | $1,200 (instancias spot + on-demand)       |
| **Amazon MSK (Kafka)**   | Cola de mensajes GPS               | $800 (3 brokers + almacenamiento)          |
| **RDS PostgreSQL**       | Datos históricos + PostGIS         | $1,500 (1 primary + 2 réplicas)            |
| **ElastiCache (Redis)**  | Caché de ubicaciones recientes     | $400 (cache.m6g.large)                     |
| **S3 + Glacier**         | Backups y datos fríos              | $200 (100GB)                               |
| **IoT Core (Opcional)**  | Gestión centralizada de dispositivos | $0.08 por millón de mensajes              |

__Total Estimado: $4,100/mes__ (puede reducirse un 40% con Spot Instances y Reserved Capacity).

## 2. Balanceo de Carga y Autoescalado

### 2.1 Estrategias

| Componente               | Táctica                                   | Herramienta                          |
|--------------------------|-------------------------------------------|--------------------------------------|
| **API Gateway**          | Balanceo Round Robin + SSL Termination    | ALB + Nginx Ingress (K8s)            |
| **MQTT Broker (EMQX)**   | DNS Round Robin + Session Persistence     | AWS NLB                              |
| **Microservicios**       | Autoescalado basado en CPU/mensajes por segundo | K8s HPA + Prometheus Metrics    |
| **Kafka**                | Particionamiento + aumento de consumers   | AWS MSK Auto Scaling                 |

### 2.2 Configuración de HPA en Kubernetes

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-gateway
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: External
    external:
      metric:
        name: kafka_messages_per_second
        selector:
          matchLabels:
            topic: gps_data
      target:
        type: AverageValue
        averageValue: 10k
```
## 3. Bases de Datos y Almacenamiento

| Tipo            | Tecnología               | Escalabilidad                          | Uso                                      |
|-----------------|--------------------------|----------------------------------------|------------------------------------------|
| **SQL (Principal)** | PostgreSQL + TimescaleDB | Particionamiento por tiempo/región     | Datos históricos + PostGIS               |
| **NoSQL (Caché)**  | Redis Cluster           | Sharding automático (16 particiones)   | Ubicaciones recientes (<1h)              |
| **Temporal**       | Kafka (7 días retención) | Aumento de partitions/consumers       | Buffer para procesamiento en tiempo real |

#### Ejemplo de Particionamiento en TimescaleDB:

```sql
CREATE TABLE gps_data (
  time TIMESTAMPTZ NOT NULL,
  device_id INT,
  region VARCHAR(10),
  location GEOGRAPHY(POINT)
) PARTITION BY LIST (region);  -- Ej: 'us-east', 'eu-central'
```

## 4. Monitoreo, Logging y Seguridad
### 4.1 Herramientas Clave

| Categoría     | Stack                        | Beneficio                                      |
|---------------|-----------------------------|-----------------------------------------------|
| **Monitoreo** | Prometheus + Grafana        | Alertas en tiempo real (ej: latencia >500ms)  |
| **Logging**   | ELK (Fluentd + Elasticsearch) | Búsqueda rápida en logs distribuidos         |
| **Seguridad** | AWS WAF + GuardDuty         | Protección contra DDoS y ataques de fuerza bruta |
| **Tracing**   | Jaeger                      | Diagnóstico de cuellos de botella            |

## 5. Escalabilidad Horizontal/Vertical

| Componente      | Escalabilidad Horizontal          | Escalabilidad Vertical                |
|----------------|-----------------------------------|---------------------------------------|
| **API Gateway** | Aumento de pods (K8s HPA)        | Upgrade a instancias m6i.2xlarge      |
| **Flink (Streaming)** | Más task managers          | Heap size ajustable (-Xmx8G)          |
| **PostgreSQL**  | Réplicas de lectura               | Upgrade a r6g.4xlarge                 |
| **Redis**       | Añadir nodos al cluster           | Uso de instancias memory-optimized    |

## 6. Costo-Efectividad

### 6.1 Estrategias de Ahorro

* Spot Instances: Para workers de Flink/Kafka (70% menos costoso).

* Reserved Instances: Para bases de datos (hasta 40% descuento).

* Cold Storage: Datos históricos >1 año en S3 Glacier Deep Archive ($0.00099/GB-mes).

### 6.2 Comparativa de Costos

| Escenario        | AWS (On-Demand)   | AWS (Optimizado)   |
|------------------|------------------|------------------|
| **50k dispositivos** | $4,100/mes     | $2,500/mes       |
| **500k dispositivos** | $18,000/mes   | $10,000/mes      |

## 7. Mitigación de Cuellos de Botella

| Cuello de Botella            | Solución                                  | Técnica                                      |
|------------------------------|------------------------------------------|---------------------------------------------|
| **Conexiones MQTT masivas**  | Broker clusterizado (EMQX) + Load Balancer | Session persistence + QoS nivel 1           |
| **Picos de escritura en DB** | Buffer con Kafka + batch inserts en Flink | Lógica de buffering (ej: 1k inserts/5s)    |
| **Consultas lentas en mapas**| Caché en Redis + materialized views      | Precomputar zonas frecuentes                |
| **Latencia en APIs**         | CDN (CloudFront) para frontend estático  | Cache HTTP de 5 minutos                     |


[__Anterior__ Apis](/docs/manejo-de-api.md)

[__Siguente__ Seguridad](/docs/seguridad.md)