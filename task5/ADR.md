# ADR Task 5: Эволюция MVP в SaaS-платформу

**Автор:** Башир Филипп  
**Дата:** 09.03.2026  
**Базовое решение:** Edge-First архитектура (см. Task 4)

---

## Введение

Задача: трансформировать MVP в **мультитенантную SaaS-платформу** для других агрохолдингов.

### Что было выбрано в Task 4
- Edge-First решение
- Видеоаналитика на фермах (latency <5 сек)
- Автономность при офлайне
- RabbitMQ для событий

---

## Требования к SaaS-платформе

1. **Изоляция данных** между клиентами
2. **Система подписок** - тарифные планы (Basic/Pro/Enterprise)
3. **Public API** - для интеграции с системами клиентов
4. **Панель самообслуживания** - управление аккаунтом
5. **Горизонтальное масштабирование**
6. **Usage metering** - сбор метрик для биллинга
7. **Developer Portal** - документация API

---

## 1. Мультитенантная архитектура

### 1.1 Вариант 1: Shared Schema

**Описание:** Одна PostgreSQL БД, каждый клиент в своей схеме.

**Плюсы:**
- Низкая стоимость
- Простота управления
- Легкий backup

**Минусы:**
- Риск утечки данных
- Сложность масштабирования
- "Noisy neighbor" проблема
---

### 1.2 Вариант 2: Database per Tenant ⭐

**Описание:** Отдельная PostgreSQL БД для каждого клиента.

**Плюсы:**
- Сильная изоляция (физическое разделение)
- Независимое масштабирование
- Простая кастомизация
- Проще backup отдельного клиента

**Минусы:**
- Больше операционных затрат
- Сложнее агрегации по всем клиентам
- Больше connection pools

---

### 1.3 Вариант 3: Cluster per Tenant

**Описание:** Полностью выделенный кластер для крупного клиента.

**Плюсы:**
- Максимальная изоляция
- Полный контроль над ресурсами
- Кастомизация под любые требования

**Минусы:**
- Очень дорого
- Сложность управления
- Долгий provisioning


---

### 1.4 Сравнение вариантов

| Критерий | Shared Schema | DB per Tenant | Cluster per Tenant |
|----------|--------------|---------------|-------------------|
| **Изоляция** | Слабая | Сильная | Максимальная |
| **Стоимость** | Низкая | Средняя | Высокая |
| **Сложность** | Низкая | Средняя | Высокая |
| **Масштабируемость** | Ограниченная | Хорошая | Отличная |
| **Provisioning** | <1 мин | 5-10 мин | 2-4 часа |

### 1.5 Рекомендация: Гибридный подход

- **Basic tier** → Shared Schema
- **Pro tier** → Database per Tenant ⭐ (рекомендуется)
- **Enterprise** → Cluster per Tenant

---

## 2. Биллинг и монетизация

### 2.1 Тарифные планы

#### Basic - N ₽/мес
- До 3 ферм
- До 10 камер
- 5 пользователей
- API: 1,000 req/день

#### Pro - N ₽/мес
- До 20 ферм
- Unlimited камеры
- 100 пользователей
- API: 10,000 req/час
- Webhooks

#### Enterprise - от N ₽/мес
- Unlimited фермы
- Custom ML-модели
- Dedicated cluster
- API: unlimited

### 2.2 Архитектура биллинга

**Новые сервисы:**
1. **Subscription Service** - управление подписками
2. **Usage Metering Service** - сбор метрик
3. **Billing Service** - расчет счетов
4. **Payment Gateway** - обработка платежей


## 3. API для клиентов

### 3.1 Public API Gateway

**Authentication:**
```
Authorization: Bearer <api_key>
API Key format: {tenant_id}_{random}
```

### 3.2 API Endpoints

```
# Incidents
GET    /api/v1/incidents
GET    /api/v1/incidents/:id
POST   /api/v1/incidents/:id/confirm

# Farms
GET    /api/v1/farms
GET    /api/v1/farms/:id/status

# Metrics
GET    /api/v1/metrics/daily
GET    /api/v1/metrics/export
```

### 3.3 Developer Portal

**Компоненты:**
- API Documentation (OpenAPI)
- Interactive API Explorer
- SDK (Python, JS, Java)
- Code Examples
- API Key Management

---

## 4. Финальная архитектура To-Be

### 4.1 C1 Диаграмма

**Файл:** [C1_saas.puml](C1_saas.puml)  
**Изображение:** [C1_SaaS_Platform_Context.png](C1_SaaS_Platform_Context.png)

**Показывает:**
- Клиенты (агрохолдинги)
- SaaS Platform
- Фермы с Edge-агентами
- Внешние системы (платежи, ERP)

### 4.2 C2 Финальная архитектура

**Файл:** [C2_saas.puml](C2_saas.puml)  
**Изображение:** [C2_SaaS_Platform_Final.png](C2_SaaS_Platform_Final.png)

**Слои:**

#### Edge Layer (фермы)
- Edge Agent (tenant context)
- Video Analytics (ONNX)
- IoT Adapter
- MQTT Broker

#### Tenant Management
- Multi-tenant API Gateway
- Tenant Registry
- Onboarding Service

#### Core Services
- Incident Service (tenant-aware)
- Alert Service
- Metrics Service
- IAM Service

#### Billing Layer
- Subscription Service
- Usage Metering (ClickHouse)
- Billing Service
- Payment Gateway

#### Integration Layer
- Public API Gateway
- Webhook Manager
- Developer Portal

### 4.3 C2 Пример интеграции

**Файл:** [C2_integrate.puml](C2_integrate.puml)  
**Изображение:** [C2_Customer_Integration_Example.png](C2_Customer_Integration_Example.png)

**Показывает:**
- Клиент "АгроХолдинг У" (tenant: agro-u-001)
- 3 фермы клиента
- Внутренние системы (ERP, BI)
- Integration Service
- Webhook Receiver
- Типы интеграций (Pull, Push, WebSocket)

---

## 5. Миграция MVP → SaaS

### 5.1 План миграции

```
MVP → MVP+Multi-tenancy → MVP+Billing → Full SaaS
     (3 мес)               (2 мес)       (2 мес)
```

### 5.2 Фаза 1: Multi-tenancy (3 месяца)

**Задачи:**
1. Добавить `tenant_id` во все таблицы
2. Реализовать Tenant Registry
3. Multi-tenant API Gateway (Kong)
4. Database per Tenant провisionинг
5. Обновить сервисы (tenant-aware)
6. Tenant context в RabbitMQ


### 5.3 Фаза 2: Billing (2 месяца)

**Задачи:**
1. Subscription Service
2. Usage Metering + ClickHouse
3. Billing Service
4. Payment Gateway
5. Тарифные планы

### 5.4 Фаза 3: Public API (2 месяца)

**Задачи:**
1. Public API Gateway
2. OpenAPI спецификация
3. Webhook Manager
4. Developer Portal
5. SDK для Python/JS


## 6. Изоляция данных

### На уровне приложения
- Tenant ID в каждом запросе
- Валидация tenant
- Запрет cross-tenant запросов

### На уровне БД
- Database per Tenant (физическая изоляция)
- Row-level security для Shared Schema
- Отдельные credentials

### На уровне инфраструктуры
- Network isolation
- K8s namespaces (Enterprise)
- TLS encryption

---

## 7. Мониторинг и операции

### 7.1 Метрики

```
tenant_active_farms{tenant_id="agro-rus-001"} 3
tenant_api_requests{tenant_id="agro-rus-001"} 8532
tenant_incidents{tenant_id="agro-rus-001"} 12
```

### 7.2 Scaling

**Capacity planning:**
- Старт: 10 tenants, 50 farms
- Год 1: 100 tenants, 500 farms
- Год 3: 500 tenants, 5000 farms

**Scaling:**
- DB nodes: 3 → 20
- App servers: 10 → 50
- RabbitMQ: 3 → 9 nodes

---

## 8. Риски

### Риск 1: Ошибка в tenant isolation
**Митигация:** Code review, тесты, Database per Tenant

### Риск 2: Сложность управления БД
**Митигация:** Автоматизация (Terraform), Managed PostgreSQL

### Риск 3: Noisy neighbor
**Митигация:** Shared Schema только для Basic, resource limits

### Риск 4: Проблемы с платежами
**Митигация:** Retry logic, 2-3 платежные системы

---

## 9. Заключение

### Выбранное решение

**Multi-tenancy:**
- Basic → Shared Schema
- Pro → Database per Tenant ⭐
- Enterprise → Cluster per Tenant

**Ключевые компоненты:**
1. Multi-tenant API Gateway (Kong)
2. Database per Tenant
3. Usage Metering (ClickHouse)
4. Billing Service
5. Public API Gateway
6. Developer Portal

**Сохраняется Edge-First:**
- Видеоаналитика на фермах (<5 сек)
- Автономность
- Локальные оповещения

### Диаграммы

1. **C1** - Контекст SaaS-платформы
2. **C2 Финальная** - Полная архитектура с billing и API
3. **C2 Интеграция** - Пример клиента с интеграциями

### Следующие шаги

1. Утвердить Database per Tenant
2. Фаза 1: Multi-tenancy (3 мес)
3. Фаза 2: Billing (2 мес)
4. Фаза 3: Public API (2 мес)
5. Beta-тестирование
6. Launch

---

**Автор:** Башир Филипп  
**Дата:** 09.03.2026
