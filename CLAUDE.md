# Food Ordering System — CLAUDE.md

## Project Overview

Proyecto de curso Udemy (Ali Gelenler) que implementa un sistema de pedidos de comida usando **microservicios**, **arquitectura event-driven** y **Domain-Driven Design (DDD)**. Es una referencia de arquitectura enterprise con patrones avanzados: Saga, Outbox, Hexagonal Architecture y CQRS-lite.

## Stack Tecnológico

| Categoria | Tecnología |
|-----------|-----------|
| Framework | Spring Boot 3.5.8 |
| Lenguaje | Java 17 |
| Mensajería | Apache Kafka (3 brokers) |
| Serialización | Apache Avro 1.12.0 + Confluent Schema Registry |
| Base de datos | PostgreSQL (schema separado por servicio) |
| ORM | Spring Data JPA / Hibernate 6 |
| Build | Maven (multi-module) |
| Testing | JUnit 5 + Mockito 5.12.0 |

## Módulos del Proyecto

```
food-ordering-system/
├── common/
│   ├── common-domain/        # AggregateRoot, BaseEntity, Value Objects (Money, IDs)
│   ├── common-application/   # Utilities compartidos de capa de aplicación
│   └── common-dataaccess/    # Componentes JPA compartidos
├── infrastructure/
│   ├── saga/                 # Interface SagaStep<T> (process/rollback)
│   ├── outbox/               # Interface OutboxScheduler
│   └── kafka/
│       ├── kafka-config-data/
│       ├── kafka-producer/
│       ├── kafka-consumer/
│       └── kafka-model/      # Avro .avsc schemas y clases generadas
├── order-service/            # Puerto 8181
├── payment-service/          # Puerto 8182
├── restaurant-service/       # Puerto 8183
└── customer-service/         # Puerto 8184
```

### Estructura interna de cada servicio (ejemplo: order-service)

```
order-service/
├── order-domain/
│   ├── order-domain-core/           # Aggregates, Domain Services, Domain Events
│   └── order-application-service/  # Use Cases, Ports (interfaces), Sagas, Outbox schedulers
├── order-application/               # REST Controllers
├── order-dataaccess/                # JPA entities, repositories, adapter implementations
├── order-messaging/                 # Kafka listeners y publishers
└── order-container/                 # @SpringBootApplication, application.yml
```

## Arquitectura

### Hexagonal (Ports & Adapters)

- **Ports de entrada:** `OrderApplicationService`, `PaymentResponseMessageListener`, `RestaurantApprovalResponseMessageListener`
- **Ports de salida:** `OrderRepository`, `PaymentOutboxRepository`, `PaymentRequestMessagePublisher`
- **Adapters:** implementaciones en `order-dataaccess` y `order-messaging`
- La capa de dominio no depende de ningún framework

### DDD

- **Aggregates:** `Order`, `Payment`, `OrderApproval`, `Customer`, `Restaurant`
- **Value Objects:** `Money`, `OrderId`, `CustomerId`, `RestaurantId`, `TrackingId`, `OrderStatus`, `PaymentStatus`
- **Domain Services:** `OrderDomainService`, `PaymentDomainService`, `RestaurantDomainService`
- **Domain Events:** `OrderCreatedEvent`, `OrderPaidEvent`, `OrderCancelledEvent`, `PaymentCompletedEvent`, `OrderApprovedEvent`

### Saga Pattern

Gestiona transacciones distribuidas con compensación:

- `OrderPaymentSaga` — paso 1: procesar/rollback pago
- `OrderApprovalSaga` — paso 2: procesar/rollback aprobación restaurante

**SagaStatus:** `STARTED → PROCESSING → SUCCEEDED` (happy path) / `COMPENSATING → COMPENSATED` (rollback)

### Outbox Pattern

Garantiza entrega confiable de eventos:

1. Se persiste evento en tabla outbox (junto con la transacción de negocio, atómicamente)
2. Scheduler (`@Scheduled`, cada 10s) lee registros con `OutboxStatus.STARTED`
3. Publica a Kafka
4. Actualiza status a `COMPLETED` (o `FAILED`)

**Tablas outbox:** `payment_outbox`, `restaurant_approval_outbox` (en schema `order`)

## Flujo de Comunicación entre Servicios

```
Cliente → [POST /orders] → Order Service
    ↓ (guarda Order + payment_outbox atómicamente)
PaymentOutboxScheduler → [payment-request topic] → Payment Service
    ↓ (procesa pago)
Payment Service → [payment-response topic] → Order Service
    ↓ (OrderPaymentSaga.process())
ApprovalOutboxScheduler → [restaurant-approval-request topic] → Restaurant Service
    ↓ (valida y aprueba)
Restaurant Service → [restaurant-approval-response topic] → Order Service
    ↓ (OrderApprovalSaga.process()) → Order APPROVED
```

### Kafka Topics

| Topic | Dirección | Avro Model |
|-------|-----------|-----------|
| `payment-request` | Order → Payment | `PaymentRequestAvroModel` |
| `payment-response` | Payment → Order | `PaymentResponseAvroModel` |
| `restaurant-approval-request` | Order → Restaurant | `RestaurantApprovalRequestAvroModel` |
| `restaurant-approval-response` | Restaurant → Order | `RestaurantApprovalResponseAvroModel` |
| `customer` | Customer Service → Order | `CustomerAvroModel` |

## Bases de Datos

PostgreSQL en `localhost:5432`, una base de datos con schemas separados:

| Servicio | Schema | Tablas principales |
|---------|--------|-------------------|
| order-service | `order` | `orders`, `order_items`, `order_address`, `payment_outbox`, `restaurant_approval_outbox` |
| payment-service | `payment` | `payments`, `credit_entries`, `credit_histories`, `payment_outbox` |
| restaurant-service | `restaurant` | `restaurants`, `products`, `order_approvals`, `restaurant_approval_outbox` |
| customer-service | `customer` | `customers` |

## Infraestructura Local Requerida

- **PostgreSQL:** `localhost:5432` (user: `postgres`, pass: `admin`)
- **Kafka:** 3 brokers en `localhost:19092`, `localhost:29092`, `localhost:39092`
- **Schema Registry:** `http://localhost:8081`

## Comandos de Build

```bash
# Compilar todo el proyecto
mvn clean install

# Compilar un servicio específico (ej. order-service)
cd order-service && mvn clean install

# Saltar tests
mvn clean install -DskipTests

# Compilar sin generar JAR ejecutable
mvn clean package -DskipTests
```

## Convenciones del Proyecto

- Paquete raíz: `com.food.ordering.system`
- Cada servicio tiene su propio `application.yml` en `<service>-container/src/main/resources/`
- Los schemas Avro `.avsc` están en `infrastructure/kafka/kafka-model/src/main/resources/avro/`
- Los scripts SQL de inicialización están en `<service>-container/src/main/resources/`
- Los mappers de datos JPA están en `<service>-dataaccess/`
- Los mappers de mensajería están en `<service>-messaging/`

## Ciclo de Vida de una Orden

```
PENDING → PAID → APPROVED  (camino feliz)
PENDING → CANCELLING → CANCELLED  (fallo en pago)
PAID    → CANCELLING → CANCELLED  (fallo en aprobación del restaurante)
```

## Notas Importantes

- El dominio puro (`*-domain-core`) **no** debe tener dependencias a Spring ni a frameworks
- El **Outbox pattern** es crítico para evitar pérdida de mensajes: nunca publicar a Kafka directamente desde el dominio
- Los **Sagas** coordinan transacciones distribuidas — siempre implementar el método `rollback()` para compensación
- Los **Value Objects** son inmutables y se comparan por valor, no por identidad
- El campo `version` en las tablas outbox previene actualizaciones concurrentes (optimistic locking)
- Los consumers Kafka operan en modo batch (`batch-listener: true`, hasta 500 registros)
