# Миграция REST API на GraphQL для сервиса client-info

## 1. Анализ текущей REST архитектуры

### Исходный REST контракт

Сервис `client-info` предоставляет следующие REST эндпоинты:

| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | `/clients/{id}` | Получить информацию о клиенте |
| GET | `/clients/{id}/documents` | Получить список документов клиента |
| GET | `/clients/{id}/relatives` | Получить список родственников клиента |

### Выявленные проблемы REST API

#### 1.1 Over-fetching (избыточная загрузка данных)

Карточка клиента содержит до **500 атрибутов**, но в разных сценариях используются разные подмножества:

| Сценарий | Необходимые поля | REST возвращает | Избыточность |
|----------|------------------|-----------------|--------------|
| Отображение в списке | id, name, phone | 500 полей | 99.6% |
| Оформление полиса | id, name, birthDate, passport | 500 полей | 98% |
| Личный кабинет | id, name, email, phone, address | 500 полей | 95% |

**Последствия:**
- Увеличение времени загрузки страниц
- Лишний сетевой трафик
- Повышенная нагрузка на сервер и базу данных

#### 1.2 Under-fetching (недостаточная загрузка данных)

Для получения полной информации о клиенте требуется **3 отдельных запроса**:

```
1. GET /clients/123          → клиент
2. GET /clients/123/documents → документы
3. GET /clients/123/relatives → родственники
```

**Последствия:**
- Увеличение времени отклика (сумма RTT всех запросов)
- Высокая нагрузка на сервер (3 запроса вместо 1)
- Сложность управления состоянием на клиенте
- Проблема N+1 запросов при отображении списка клиентов

#### 1.3 Проблема версионирования API

При добавлении новых полей или изменении структуры приходится:
- Создавать новые версии API (`/v1/`, `/v2/`)
- Поддерживать несколько версий одновременно
- Мигрировать потребителей на новую версию

---

## 2. Решение на GraphQL

### 2.1 Преимущества GraphQL для сервиса client-info

| Проблема REST | Решение GraphQL |
|---------------|-----------------|
| Over-fetching | Клиент выбирает только нужные поля |
| Under-fetching | Один запрос для получения всех данных |
| Версионирование | Эволюция схемы без версий |
| N+1 запросы | DataLoader для эффективной загрузки |

### 2.2 Примеры запросов GraphQL

#### Пример 1: Отображение в списке (минимальные данные)

```graphql
query GetClientsList {
  client(id: "123") {
    id
    name
    phone
  }
}
```

**Результат:**
```json
{
  "data": {
    "client": {
      "id": "123",
      "name": "Иванов Иван",
      "phone": "+7 (999) 123-45-67"
    }
  }
}
```

**Сравнение с REST:**
- REST: 500 полей (~15 КБ)
- GraphQL: 3 поля (~100 байт)
- **Экономия трафика: 99.3%**

#### Пример 2: Оформление полиса (определённый набор полей)

```graphql
query GetClientForPolicy($id: ID!) {
  client(id: $id) {
    id
    name
    birthDate
    passport {
      series
      number
      issueDate
      region
    }
    address {
      city
      street
      house
    }
  }
}
```

**Результат:**
```json
{
  "data": {
    "client": {
      "id": "123",
      "name": "Иванов Иван",
      "birthDate": "1985-03-15",
      "passport": {
        "series": "4500",
        "number": "123456",
        "issueDate": "2010-05-20",
        "region": "Москва"
      },
      "address": {
        "city": "Москва",
        "street": "ул. Ленина",
        "house": "10"
      }
    }
  }
}
```

#### Пример 3: Полная информация (решение проблемы under-fetching)

```graphql
query GetClientFull($id: ID!) {
  client(id: $id) {
    id
    name
    phone
    email
    documents(first: 5) {
      edges {
        node {
          id
          type
          number
          status
        }
      }
    }
    relatives(first: 10) {
      edges {
        node {
          id
          relationType
          name
          age
        }
      }
    }
  }
}
```

**Сравнение с REST:**
- REST: 3 запроса, общее время = RTT × 3 + обработка
- GraphQL: 1 запрос, время = RTT × 1 + обработка
- **Ускорение: до 60%** (зависит от сетевой задержки)

---

## 3. Реализация паттернов оптимизации

### 3.1 DataLoader для защиты от N+1 запросов

Проблема N+1 возникает при получении списка клиентов с их документами:

```graphql
query GetClients {
  clients(first: 100) {
    edges {
      node {
        id
        name
        documents(first: 10) {  # N запросов к БД
          edges {
            node { id type number }
          }
        }
      }
    }
  }
}
```

**Решение через DataLoader:**

```javascript
// Псевдокод реализация DataLoader
class DocumentLoader {
  async load(clientId) {
    // Один запрос к БД для всех клиентов
    const documents = await db.documents.find({ clientId: { $in: clientIds } });
    return documents;
  }
}

// В resolver'е
const documents = await context.loaders.document.load(client.id);
```

**Результат:**
- Без DataLoader: 101 запрос к БД (1 + 100)
- С DataLoader: 2 запроса к БД (1 для клиентов + 1 для всех документов)
- **Ускорение: 50×**

### 3.2 Пагинация для больших коллекций

Использована пагинация типа **Relay Connection** с курсорами:

```graphql
query GetDocuments($clientId: ID!, $first: Int, $after: String) {
  clientDocuments(clientId: $clientId, first: $first, after: $after) {
    edges {
      node {
        id
        type
        number
        status
      }
      cursor
    }
    pageInfo {
      hasNextPage
      hasPreviousPage
      startCursor
      endCursor
    }
    totalCount
  }
}
```

**Преимущества:**
- Эффективная загрузка больших списков
- Поддержка бесконечного скроллинга
- Стабильность при изменении данных

### 3.3 Типобезопасность через Enums

Использование перечислений для валидации на уровне схемы:

```graphql
enum DocumentType {
  PASSPORT
  DRIVER_LICENSE
  SNILS
  INN
  MILITARY_ID
  BIRTH_CERTIFICATE
  OTHER
}

enum DocumentStatus {
  ACTIVE
  EXPIRED
  REVOKED
  PENDING_VERIFICATION
}
```

---

## 4. План миграции

### Этап 1: Подготовка (1-2 недели)
- [ ] Внедрить GraphQL-сервер (Apollo Server, graphql-yoga)
- [ ] Создать схему GraphQL (файл `client-info-graphql-schema.graphql`)
- [ ] Написать resolver'ы для существующих REST эндпоинтов
- [ ] Настроить DataLoader для оптимизации запросов

### Этап 2: Параллельная работа (2-3 недели)
- [ ] Подключить GraphQL к новым функциональностям
- [ ] Продолжить работу REST API для существующих клиентов
- [ ] Мониторинг метрик производительности

### Этап 3: Постепенная миграция (4-6 недель)
- [ ] Мигрировать веб-приложение на GraphQL
- [ ] Мигрировать сервис core-app на GraphQL
- [ ] Документировать breaking changes (если будут)

### Этап 4: Декомиссия REST (1-2 недели)
- [ ] Отключить REST эндпоинты
- [ ] Удалить старый код
- [ ] Обновить документацию

---

## 5. Метрики успеха

| Метрика | До (REST) | После (GraphQL) | Цель |
|---------|-----------|-----------------|------|
| Время загрузки страницы | 2.5 сек | 1.0 сек | -60% |
| Количество запросов на страницу | 3 | 1 | -67% |
| Объём передаваемых данных | ~15 КБ | ~500 байт | -97% |
| RPS сервиса client-info | 250 | 50 | -80% |
| Время разработки новых фич | 5 дней | 2 дня | -60% |

---

## 6. Заключение

Миграция на GraphQL решает ключевые проблемы REST API сервиса `client-info`:

1. **Over-fetching** — клиент выбирает только нужные поля
2. **Under-fetching** — один запрос для всех данных
3. **Версионирование** — эволюция схемы без breaking changes
4. **N+1 запросы** — DataLoader для эффективной загрузки

Дополнительные преимущества:
- Упрощённая разработка фронтенда
- Автоматическая документация через GraphQL Schema
- Улучшенная типобезопасность
- Возможность кэширования на уровне полей

---

## Приложения

### A. Полный список сущностей в схеме

| Тип | Поля | Описание |
|-----|------|----------|
| Client | id, name, age, phone, email, birthDate, passport, address, documents, relatives, createdAt, updatedAt | Основная сущность клиента |
| PassportData | series, number, departmentCode, issueDate, issuedBy, region | Паспортные данные |
| Address | postalCode, city, street, house, apartment | Адрес регистрации |
| Document | id, type, number, issueDate, expiryDate, status, series | Документ клиента |
| Relative | id, relationType, name, age, birthDate, gender | Родственник клиента |
