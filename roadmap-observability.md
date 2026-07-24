# Roadmap — Observabilidad (Metrics / Logs / Traces)

Objetivo: instrumentar los 4 microservicios sobre el stack existente del homelab
(**Prometheus pull + Grafana + Loki/Promtail** en el namespace `monitoring`; **Tempo** aún no desplegado)
para tener métricas JVM + custom, logs estructurados útiles en Loki y, más adelante, trazas distribuidas.

## Stack de destino

| Pilar | Herramienta | Cómo llega a Grafana |
|-------|-------------|----------------------|
| Métricas | Micrometer + `micrometer-registry-prometheus` → `/actuator/prometheus` | Prometheus scrapea (pull) |
| Logs | Structured logging nativo de Spring Boot 3.4+ (JSON ECS) a stdout | Promtail → Loki |
| Trazas | `micrometer-tracing-bridge-otel` + OTLP exporter | OTLP → Tempo *(pendiente de desplegar)* |

## Dependencias por fase

| Fase | Dependencia | Dónde |
|------|-------------|-------|
| 1 | `spring-boot-starter-actuator` | los 4 `-container` |
| 1 | `micrometer-registry-prometheus` | los 4 `-container` |
| 1 | `spring-boot-starter-web` | **solo** `payment-container` + `restaurant-container` |
| 2 | `io.micrometer:micrometer-core` | módulos con custom metrics (ej. `order-application`) |
| 3 | `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp` | los 4 `-container` |

> ⚠️ **payment-service y restaurant-service no tienen servidor web** (`spring-boot-starter` a secas, son solo-Kafka).
> Sin `spring-boot-starter-web` no exponen endpoint HTTP y Prometheus no los puede scrapear.
> `order-service` y `customer-service` ya tienen web vía sus módulos `-application`.

---

## Fase 0 — Base de métricas + logs (order-service) ✅ HECHO

Cambios aplicados y verificados (compilan/empaquetan con JDK 17):

| Archivo | Cambio |
|---------|--------|
| `order-container/pom.xml` | + `spring-boot-starter-actuator`, + `micrometer-registry-prometheus` |
| `order-application/pom.xml` | + `io.micrometer:micrometer-core` |
| `order-container/.../application.yml` | bloque `management` (expone `prometheus`, health probes), `spring.application.name`, `show-sql:false`, Hibernate → `WARN` |
| `OrderController.java` | `Counter orders.created` (incrementa por orden) + log con `trackingId` en MDC |

Resultado: métricas JVM gratis (`jvm_memory_used_bytes`, GC, threads, CPU) + `orders_created_total`, y logs listos para estructurar.

### Config de referencia (application.yml)
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
  metrics:
    tags:
      application: ${spring.application.name}
  prometheus:
    metrics:
      export:
        enabled: true

logging:
  level:
    com.food.ordering.system: INFO
    org.hibernate.SQL: WARN
    org.hibernate.orm.jdbc.bind: WARN
```

### Logs JSON solo en k8s (dev local queda en texto plano)
Setear como env var en el deployment (encaja con la convención de overrides por env):
```yaml
env:
  - name: LOGGING_STRUCTURED_FORMAT_CONSOLE
    value: ecs
```

---

## Fase 1 — Descubrimiento en Prometheus (PENDIENTE)

Falta que Prometheus descubra el endpoint `/actuator/prometheus`. Elegir según el setup:

- **kube-prometheus-stack (Operator):** crear un `ServiceMonitor` que apunte al Service de order-service, puerto 8181, path `/actuator/prometheus`.
- **Prometheus "plano":** anotaciones en el Service/pod:
  ```yaml
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8181"
    prometheus.io/path: "/actuator/prometheus"
  ```

> TODO: confirmar cuál Prometheus corre en el cluster y materializar el manifiesto.

### Grafana
- Importar dashboard **4701** (*JVM (Micrometer)*) para métricas JVM.
- (Opcional) dashboard custom para `orders_created_total` y métricas de negocio.

---

## Fase 2 — Replicar en los otros 3 servicios (PENDIENTE)

1. **customer-service** — igual que order-service (ya tiene web).
2. **payment-service** — idem **+ agregar `spring-boot-starter-web`** (solo-Kafka).
3. **restaurant-service** — idem **+ agregar `spring-boot-starter-web`** (solo-Kafka).

Para cada uno: actuator + prometheus + `LOGGING_STRUCTURED_FORMAT_CONSOLE=ecs` + reducir ruido Hibernate + custom metrics relevantes (ej. pagos procesados, aprobaciones, rollbacks de SAGA).

---

## Fase 3 — Trazas distribuidas → Tempo (PENDIENTE)

1. Desplegar **Tempo** en el cluster (namespace `monitoring`).
2. Agregar a los 4 container: `micrometer-tracing-bridge-otel` + `opentelemetry-exporter-otlp`.
3. Configurar el endpoint OTLP y el sampling.
4. Correlación logs↔trazas: el structured logging ya incluye `trace_id`/`span_id` como campos → linkear desde Loki a Tempo en Grafana.

---

## Verificación (order-service)

```bash
# Build local (¡usar JDK 17, ver nota abajo!)
JAVA_HOME=~/.sdkman/candidates/java/17.0.9-amzn mvn -pl order-service/order-container -am -DskipTests package

# Deploy
git push origin master              # GH Actions build ARM64
kubectl rollout restart deployment order-service -n food-ordering

# Métricas
kubectl port-forward -n food-ordering deployment/order-service 8181:8181
curl -s localhost:8181/actuator/prometheus | grep -E "jvm_memory_used_bytes|orders_created_total"

# Logs en Loki (Grafana → Explore)
# {namespace="food-ordering", app="order-service"} | json | trackingId != ""
```

---

## Notas / Gotchas

- 🔴 **Builds locales:** el `java` default de la máquina es **JDK 25**, que rompe Lombok en silencio
  (`cannot find symbol: log`), porque desde JDK 23 el annotation processing implícito del classpath está
  deshabilitado y Lombok está como `provided`, no en `annotationProcessorPaths`. **CI usa JDK 17 y funciona.**
  Compilar local con `JAVA_HOME=~/.sdkman/candidates/java/17.0.9-amzn mvn ...`.
  *Fix permanente opcional:* agregar lombok a `annotationProcessorPaths` del maven-compiler-plugin.
- **"logback":** el structured logging nativo de Spring Boot **es logback por debajo** (Spring monta el
  encoder JSON sobre logback). Solo hace falta un `logback-spring.xml` explícito si se quiere control fino
  (appenders separados, formato propio).
- No mezclar la config de observabilidad (universal) con los overrides por-entorno (datasource/kafka),
  que siguen yendo por env vars en los deployments de k8s.