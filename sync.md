# Техническая архитектура интеграционной сети (Паттерн: Умный Буфер)

## 1. Концептуальная схема (Компоненты и топология)

```mermaid
graph TD
    %% Описание узлов
    subgraph "Платный Enterprise-контур (Лицензии)"
        BaseLic[База 1С: С лицензией]
        ESB[Datareon ESB Bus]
    end

    subgraph "Центральное интеграционное ядро"
        Hub[УМНАЯ БУФЕРНАЯ БАЗА 1С / HUB]
    end

    subgraph "Безлицензионный контур (Бесплатно)"
        BaseNoLic1[Безлицензионная 1С: Альфа]
        BaseNoLic2[Безлицензионная 1С: Бета]
    end

    %% Описание связей
    BaseLic <-->|Родной адаптер 1С| ESB
    ESB <-->|1 Общий адаптер| Hub
    
    %% Потоки без лицензий
    Hub <-->|Real-time HTTP POST| BaseNoLic1
    Hub <-->|Real-time HTTP POST| BaseNoLic2
    BaseNoLic1 <-->|Прямой HTTP обход ESB| Hub
    BaseNoLic2 <-->|Прямой HTTP обход ESB| Hub
```

---

## 2. Диаграмма последовательности (Sequence Diagram)

### Сценарий А: Передача из Лицензионной базы в Безлицензионную (Сквозь шину)

```mermaid
sequenceDiagram
    autonumber
    participant BaseL as 1С (С лицензией)
    participant ESB as Datareon ESB
    participant Hub as Умный Буфер (Hub)
    participant BaseNL as 1С (Без лицензии)

    BaseL->>ESB: 1. Регистрация изменений (Родной адаптер)
    ESB->>Hub: 2. Передача пакета данных (Используется 1 лицензия)
    Note over Hub: Анализ метаданных пакета:<br/>Вычисление URL получателя
    Hub->>+BaseNL: 3. Активный HTTP POST (Push-доставка Payload)
    BaseNL-->>-Hub: 4. HTTP Статус 200 OK (Пакет принят)
    Note over Hub: Фиксация успешной доставки<br/>в Очереди сообщений
```

### Сценарий Б: Обмен между Безлицензионными базами напрямую (В обход Datareon)

```mermaid
sequenceDiagram
    autonumber
    participant BaseA as 1С Альфа (Без лицензии)
    participant Hub as Умный Буфер (Hub)
    participant BaseB as 1С Бета (Без лицензии)

    BaseA->>Hub: 1. HTTP POST: Универсальный JSON-конверт
    Note over Hub: Запись пакета в СУБД<br/>Регистр "Очередь сообщений"
    Hub-->>BaseA: 2. HTTP Статус 202 Accepted (Буфер принял)
    
    Note over Hub: Срабатывает регламентное задание<br/>обработки очереди
    
    Hub->>+BaseB: 3. Активный HTTP POST (Перенаправление Payload)
    BaseB->>BaseB: Внутренний кастомный парсинг JSON
    BaseB-->>-Hub: 4. HTTP Статус 200 OK (Данные применены)
    Note over Hub: Очистка или архивация<br/>очереди в буфере
```

---

## 3. Физическая структура универсального JSON-конверта

Общение всех безлицензионных баз с Умным Буфером стандартизируется под единую текстовую структуру:

```json
{
  "Header": {
    "MessageID": "f81d4fae-7dec-11d0-a765-00a0c91e6bf6",
    "Sender": "Base_Alpha",
    "Receiver": "Base_Beta",
    "MessageType": "Document_РеализацияТоваровУслуг",
    "Timestamp": "2026-08-26T12:00:00Z"
  },
  "Payload": {
    "Идентификатор1С": "УникальныйИДДокументаВБазеИсточнике",
    "Номер": "ТЛ00-001234",
    "Дата": "2026-08-26",
    "ОрганизацияИНН": "7712345678",
    "СуммаДокумента": 152000.50
  }
}
```
