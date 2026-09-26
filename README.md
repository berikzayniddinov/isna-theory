# ISNA Theory

Заметки по темам для senior Java-инженера. Разбито по папкам-темам, внутри файлы отсортированы по номерам.

## Структура

### [spring-boot/](./spring-boot/)
Spring Framework, Spring Boot, IoC/DI, MVC, embedded servers, конфигурация, аннотации.
- 01 — Spring Boot: build/runtime/K8s/Consul обзор
- 05 — Spring Framework: IoC + DI
- 06 — Spring Boot: детально
- 07 — Embedded servers: Tomcat/Undertow
- 08 — Application startup chain
- 57 — Spring Boot config: детально
- 60 — Spring аннотации: детально
- 61 — Spring MVC controllers: internals

### [java-jvm/](./java-jvm/)
Java язык, JVM внутренности, GC, JIT, threads, virtual threads.
- 02 — JVM / JDK / JRE / bytecode
- 16-18 — Java 11 → 21: syntax / JVM / API migration
- 19 — Java 21 virtual threads (базовое)
- 43 — Java basics: примитивы, память
- 83 — Java threads vs virtual threads deep
- 84 — JVM / OS / RAM / disk / native
- 110 — Virtual threads Java 21 deep
- 111 — GC: G1, ZGC, Shenandoah deep
- 112 — JIT compilation + JVM tuning deep
- 115 — Потоки: состояния, volatile, synchronized, deadlock

### [build-deploy/](./build-deploy/)
Gradle, JAR, Docker, Kubernetes, Helm, CI/CD, deployment паттерны.
- 03 — Gradle
- 04 — JAR / fat JAR
- 09 — Docker
- 10 — Kubernetes
- 58 — Helm / Helmsman
- 77 — Kubernetes deep для микросервисов
- 79 — CI/CD deploy patterns (GitOps, blue-green, canary)
- 80 — Kubernetes internals
- 81 — Build vs deployment deep
- 82 — Git push to pod: full pipeline
- 101 — Project build/run/deploy diagram

### [database/](./database/)
PostgreSQL внутренности, indexes, VACUUM, replication, partitioning, backup, security, NoSQL, Elasticsearch.
- 28 — PostgreSQL internals (базовое)
- 45 — Elasticsearch
- 86 — DBMS architecture (100M rows deep)
- 87 — Database internals: pages, I/O
- 88 — Postgres locks + EXPLAIN deep
- 89 — VACUUM, statistics, slow queries
- 90 — Postgres replication + HA
- 91 — Partitioning + sharding (Postgres-specific)
- 92 — Backup / recovery / safe migrations
- 94 — JSON, FTS, materialized views, extensions
- 95 — NoSQL сравнение + cloud databases
- 96 — Advanced SQL (CTE, window, pagination)
- 98 — PostgreSQL security deep
- 99 — Postgres monitoring in production
- 102 — Database full schema diagram
- 103 — Index scan vs seq scan deep
- 105 — Autovacuum tuning для больших таблиц
- 106 — ALTER TABLE lock queue + thundering herd
- 108 — Partitioning theory (механика planner deep)
- 109 — Indexes: когда создавать, когда нельзя

### [jpa-transactions/](./jpa-transactions/)
JPA, Hibernate, Spring Data, транзакции (ACID/propagation/internals), connection pool, fetch strategies.
- 12 — JPA basics
- 13 — Hibernate internals
- 14 — Spring Data JPA
- 15 — JPA performance
- 29 — PostgreSQL + Spring + HikariCP
- 32 — Транзакции: ACID / isolation / propagation
- 33 — @Transactional internals
- 34 — @Transactional advanced
- 35 — @Transactional + JPA + persistence context
- 68 — JPA stream / pageable / cursor / JDBC
- 71 — Fetch types + serializable
- 104 — N+1 problem + fetch strategies deep
- 107 — Connection pool + I/O wait deep

### [messaging/](./messaging/)
RabbitMQ, Kafka, integration bus, sync vs async.
- 20 — RabbitMQ AMQP basics
- 21 — RabbitMQ delivery guarantees
- 22 — Spring AMQP
- 23 — RabbitMQ prod patterns
- 39 — Kafka basics
- 40 — Kafka producer / consumer / offsets
- 41 — Spring Kafka
- 42 — Kafka prod patterns
- 65 — Integration bus / ESB
- 66 — Sync vs async

### [security/](./security/)
Spring Security, OAuth2/OIDC, Keycloak, ЭЦП (Kalkan).
- 24 — Spring Security basics
- 25 — OAuth2 / OIDC theory
- 26 — Keycloak
- 27 — Spring Security + OAuth2 + Keycloak
- 75 — Kalkan ЭЦП

### [microservices/](./microservices/)
Микросервисные паттерны: SAGA, Outbox, CQRS, Event Sourcing, resilience, decomposition, Strangler Fig.
- 48 — Monolith vs microservices
- 49 — Microservices decomposition
- 50 — Saga pattern
- 51 — Outbox / Inbox / идемпотентность
- 52 — Microservices resilience (circuit breaker, retry, bulkhead)
- 93 — Distributed transactions (Saga + Outbox)
- 97 — Data modeling для микросервисов
- 100 — CQRS + Event Sourcing deep
- 114 — Strangler Fig pattern (миграция монолита)

### [networking-api/](./networking-api/)
REST, SOAP, gateway, nginx, load balancer, TCP, XML/WSDL.
- 21 — Polling + sync patterns
- 31 — Load balancer
- 46 — Nginx
- 62 — REST API
- 63 — SOAP API
- 64 — REST vs SOAP
- 67 — API Gateway
- 69 — TCP batching / flushing
- 70 — XSD / WSDL

### [testing/](./testing/)
Unit, integration, e2e тесты.
- 53 — Testing: unit
- 54 — Testing: integration + slice
- 55 — Testing: e2e / smoke / Selenium
- 56 — Testing best practices

### [observability/](./observability/)
Логирование, метрики, prometheus, ELK, prod diagnostics.
- 38 — Logging
- 44 — Prometheus + Grafana metrics
- 76 — Logging stack e2e
- 78 — Prod diagnostics workflow

### [cache/](./cache/)
Redis, Hazelcast.
- 72 — Redis
- 73 — Hazelcast
- 74 — Redis vs Hazelcast

### [discovery-config/](./discovery-config/)
Consul, Spring Cloud, service discovery, externalized config.
- 11 — Consul detailed
- 36 — Spring Cloud
- 59 — Consul deep dive

### [patterns-architecture/](./patterns-architecture/)
Design patterns (Facade, Proxy), абстракция, SOLID, DDD.
- 85 — Design patterns: Facade + Proxy deep (Spring AOP, @Transactional через proxy)
- 113 — Абстракция в программировании (SOLID, паттерны, DDD)

### [misc/](./misc/)
Разное: highload, nodes, infra glossary.
- 30 — Highload / expensive operations
- 37 — Nodes detailed
- 47 — Infra glossary

## Стиль файлов

Большинство «deep»-файлов (файлы после ~80) написаны в **прозном стиле** — модель мира, «почему» через прод-сценарии, конкретные recipes для диагностики. Образец глубины — файл 88 (Postgres locks + EXPLAIN).

Ранние файлы (01-70) местами написаны в более компактном enumeration-стиле — постепенно приводятся к тому же стандарту.
