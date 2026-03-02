# Лабораторная работа: Паттерн "Адаптер" (Adapter)
**Тема:** Интеграция устаревшего сервиса уведомлений в современную систему оповещений.
**Язык:** C#
**Тип приложения:** Console Application

## 1. Цель работы
Научиться применять структурный паттерн **Adapter** для интеграции несовместимых интерфейсов и моделей данных. Студенты должны понять, что Адаптер берет на себя ответственность за трансляцию данных, включая преобразование типов идентификаторов, не требуя изменений в legacy-коде.

## 2. Легенда (Сценарий)
Вы разрабатываете ядро новой системы мониторинга (`ServerMonitor`). При авариях система должна отправлять уведомления.
В компании используется библиотека `LegacyNotifyLib`, написанная 20 лет назад. 
**Ключевая проблема:** Старая система была создана до внедрения современных стандартов идентификации. Она оперирует только целочисленными кодами пользователей (`int`) и не знает ничего о `Guid`, которые используются в новой системе. 
Вносить изменения в `LegacyNotifyLib` запрещено (библиотека поставляется в виде закрытой DLL, в рамках задания вы эмулируете её классы).

Ваша задача: создать Адаптер, который позволит новой системе отправлять уведомления через старый шлюз, взяв на себя всю логику преобразования идентификаторов и форматов данных.

## 3. Требования к реализации
1.  **Изоляция Legacy:** Классы старой системы не должны содержать ссылок на типы новой системы (никаких `Guid` в legacy-классах).
2.  **Ответственность Адаптера:** Преобразование `Guid` -> `int` должно происходить внутри Адаптера (эмуляция таблицы маппинга).
3.  **Преобразование данных:** Адаптер должен преобразовывать Enum в byte, DateTime в строку и объединять поля сообщения.
4.  **Клиентский код:** Сервис мониторинга не должен знать о существовании legacy-классов.

## 4. Схема классов и интерфейсов

### 4.1. Целевой интерфейс (Target)
**Интерфейс:** `INotificationChannel`
*   **Назначение:** Единый стандарт для новой системы.
*   **Метод:** `void Send(NotificationContext context)`

### 4.2. Модель данных новой системы (New System)
**Класс:** `NotificationContext`
*   **Свойства:**
    *   `Guid UserId` (Современный идентификатор)
    *   `string Title` (Заголовок)
    *   `string Body` (Текст)
    *   `Priority Priority` (Enum: `Low`, `Normal`, `Critical`)
    *   `DateTime CreatedAt`

**Перечисление:** `Priority`
*   `Low = 0`, `Normal = 1`, `Critical = 2`

### 4.3. Модели данных устаревшей системы (Legacy Models)
*Эти классы эмулируют закрытую библиотеку. Менять их сигнатуры запрещено.*

**Класс:** `LegacySmsPacket`
*   **Назначение:** Контейнер сообщения старого формата.
*   **Свойства:**
    *   `int RecipientCode` (Старая система понимает только целочисленные коды)
    *   `string Payload` (Единое поле: заголовок и текст через разделитель `|`)
    *   `byte PriorityFlag` (Приоритет как байт, возможные значения: 10, 20, 30)
    *   `string TimestampString` (Дата в формате `dd.MM.yyyy HH:mm`)

**Класс:** `LegacyUserRegistry`
*   **Назначение:** Внутренний реестр старой системы. 
*   **Метод:** `string GetPhoneNumber(int legacyCode)`
    *   *Логика:* Возвращает номер телефона по внутреннему коду (заглушка).
*   **Метод:** `bool ValidateCode(int legacyCode)`
    *   *Логика:* Проверяет, существует ли код в старой базе.

### 4.4. Адаптируемый класс (Adaptee)
**Класс:** `LegacySmsGateway`
*   **Назначение:** Старый шлюз отправки.
*   **Метод:** `void SubmitPacket(LegacySmsPacket packet)`
    *   *Важно:* Принимает только `LegacySmsPacket`.
    *   *Действие:* Выводит в консоль: `[LEGACY] Отправка коду [RecipientCode]. Текст: [Payload]`.

### 4.5. Адаптер (Adapter)
**Класс:** `SmsGatewayAdapter`
*   **Реализация:** `INotificationChannel`
*   **Зависимости:**
    *   `LegacySmsGateway _gateway`
    *   `LegacyUserRegistry _registry`
*   **Внутреннее состояние:**
    *   Должен содержать механизм маппинга `Guid` -> `int`. Для целей лабораторной работы реализуйте это через приватный `Dictionary<Guid, int>` или метод `GetLegacyCode(Guid id)`, возвращающий хардкоженные значения для тестов.
*   **Логика метода `Send(NotificationContext context)`:**
    1.  Получить `int legacyCode` из `Guid context.UserId`, используя внутренний маппинг адаптера.
    2.  Вызвать `_registry.ValidateCode(legacyCode)`. Если код неверен — выбросить исключение или logged warning.
    3.  Преобразовать `Priority` в `byte` (Low->10, Normal->20, Critical->30).
    4.  Сформировать `Payload` = `${context.Title}|${context.Body}`.
    5.  Сформировать `TimestampString` из `DateTime`.
    6.  Создать `LegacySmsPacket` и передать в `_gateway.SubmitPacket`.

### 4.6. Клиент (Client)
**Класс:** `MonitoringService`
*   **Зависимости:** `INotificationChannel`
*   **Метод:** `void TriggerAlert(NotificationContext context)`
*   **Логика:** Вызывает `channel.Send(context)`. Не знает про `Guid` vs `int`.

## 5. Детали преобразования данных (Маппинг)

| Поле новой системы | Поле старой системы | Где происходит преобразование |
| :--- | :--- | :--- |
| `Guid UserId` | `int RecipientCode` | **Внутри Адаптера** (через словарь/маппинг) |
| `Priority` (Enum) | `byte PriorityFlag` | Внутри Адаптера (switch/case) |
| `Title` + `Body` | `string Payload` | Внутри Адаптера (конкатенация) |
| `DateTime` | `string` | Внутри Адаптера (`.ToString()`) |

## 6. Общие требования
1.  **Чистота Legacy:** В файлах классов `LegacySmsPacket`, `LegacySmsGateway`, `LegacyUserRegistry` **нет** изменений.
2.  **Логика Адаптера:** Адаптер корректно преобразует `Guid` в `int` перед созданием пакета.
3.  **Валидация:** Адаптер использует `LegacyUserRegistry` для проверки кода перед отправкой.
4.  **Соблюдение интерфейса:** `MonitoringService` работает строго через `INotificationChannel`.

## 7. Пример ожидаемого вывода в консоль
```text
[MONITOR] Тревога! Пользователь (GUID): 550e8400-e29b-41d4-a716-446655440000
[ADAPTER] Преобразование GUID в Legacy Code...
[ADAPTER] Legacy Code получен: 1024. Проверка в реестре...
[ADAPTER] Код валиден. Преобразование приоритета 'Critical' -> 30.
[LEGACY] Отправка коду [1024]. Текст: [Server Down|DB Connection Lost]
[MONITOR] Уведомление отправлено.
```

# Заглушки для устаревшей системы (Legacy System Stubs)

> **Важно для студентов:** Эти классы эмулируют стороннюю закрытую библиотеку. **Менять их код запрещено.** Вы должны работать с ними только через публичные методы. Наилучший вариант - вынести этот код в отдельную библиотеку.

---

## 1. LegacySmsPacket.cs
```csharp
namespace LegacyNotifyLib
{
    /// <summary>
    /// Контейнер сообщения устаревшего формата.
    /// Не может быть изменён — эмулирует закрытую библиотеку.
    /// </summary>
    public class LegacySmsPacket
    {
        /// <summary>
        /// Целочисленный код получателя (старая система не знает Guid)
        /// </summary>
        public int RecipientCode { get; set; }

        /// <summary>
        /// Объединённое сообщение: заголовок и тело через разделитель |
        /// </summary>
        public string Payload { get; set; }

        /// <summary>
        /// Флаг приоритета в виде байта (10, 20, 30)
        /// </summary>
        public byte PriorityFlag { get; set; }

        /// <summary>
        /// Дата в строковом формате dd.MM.yyyy HH:mm
        /// </summary>
        public string TimestampString { get; set; }

        public override string ToString()
        {
            return $"[Code: {RecipientCode}] [Priority: {PriorityFlag}] [Time: {TimestampString}]";
        }
    }
}
```

---

## 2. LegacyUserRegistry.cs
```csharp
namespace LegacyNotifyLib
{
    /// <summary>
    /// Внутренний реестр пользователей старой системы.
    /// Работает только с целочисленными кодами — не знает о Guid.
    /// Не может быть изменён — эмулирует закрытую библиотеку.
    /// </summary>
    public class LegacyUserRegistry
    {
        // Внутренняя "база данных" старых кодов
        private readonly HashSet<int> _validCodes = new HashSet<int> 
        { 
            1024, 2048, 4096, 8192, 16384 
        };

        /// <summary>
        /// Проверяет, существует ли код в старой базе пользователей
        /// </summary>
        /// <param name="legacyCode">Целочисленный код пользователя</param>
        /// <returns>True если код валиден</returns>
        public bool ValidateCode(int legacyCode)
        {
            Console.WriteLine($"[LEGACY REGISTRY] Проверка кода {legacyCode}...");
            
            if (_validCodes.Contains(legacyCode))
            {
                Console.WriteLine($"[LEGACY REGISTRY] Код {legacyCode} найден в базе.");
                return true;
            }
            
            Console.WriteLine($"[LEGACY REGISTRY] Код {legacyCode} НЕ найден в базе!");
            return false;
        }

        /// <summary>
        /// Возвращает номер телефона по внутреннему коду
        /// </summary>
        /// <param name="legacyCode">Целочисленный код пользователя</param>
        /// <returns>Номер телефона в формате +7XXXXXXXXXX</returns>
        public string GetPhoneNumber(int legacyCode)
        {
            Console.WriteLine($"[LEGACY REGISTRY] Запрос телефона для кода {legacyCode}...");
            
            // Эмуляция разных номеров для разных кодов
            return legacyCode switch
            {
                1024 => "+79001112233",
                2048 => "+79004445566",
                4096 => "+79007778899",
                _ => "+79000000000"
            };
        }
    }
}
```

---

## 3. LegacySmsGateway.cs
```csharp
namespace LegacyNotifyLib
{
    /// <summary>
    /// Устаревший шлюз отправки SMS-уведомлений.
    /// Принимает только LegacySmsPacket — другие форматы не поддерживаются.
    /// Не может быть изменён — эмулирует закрытую библиотеку.
    /// </summary>
    public class LegacySmsGateway
    {
        private readonly string _gatewayUrl = "https://old-sms-gateway.internal";
        private readonly string _apiKey = "legacy-key-xxxx-xxxx";

        /// <summary>
        /// Отправляет пакет уведомлений через старый шлюз
        /// </summary>
        /// <param name="packet">Пакет в старом формате (другие форматы не принимаются)</param>
        /// <exception cref="ArgumentNullException">Если пакет null</exception>
        /// <exception cref="ArgumentException">Если Payload пустой</exception>
        public void SubmitPacket(LegacySmsPacket packet)
        {
            if (packet == null)
            {
                throw new ArgumentNullException(nameof(packet), "Пакет не может быть null");
            }

            if (string.IsNullOrWhiteSpace(packet.Payload))
            {
                throw new ArgumentException("Payload не может быть пустым", nameof(packet));
            }

            Console.WriteLine("===========================================");
            Console.WriteLine("[LEGACY GATEWAY] Инициализация соединения...");
            Console.WriteLine($"[LEGACY GATEWAY] URL: {_gatewayUrl}");
            Console.WriteLine($"[LEGACY GATEWAY] API Key: {_apiKey}");
            Console.WriteLine("-------------------------------------------");
            Console.WriteLine($"[LEGACY GATEWAY] Отправка пакета:");
            Console.WriteLine($"  • Получатель (Code): {packet.RecipientCode}");
            Console.WriteLine($"  • Приоритет (Flag): {packet.PriorityFlag}");
            Console.WriteLine($"  • Время: {packet.TimestampString}");
            Console.WriteLine($"  • Сообщение: {packet.Payload}");
            Console.WriteLine("-------------------------------------------");
            
            // Эмуляция задержки сети
            System.Threading.Thread.Sleep(300);
            
            Console.WriteLine("[LEGACY GATEWAY] ✓ Пакет успешно отправлен!");
            Console.WriteLine("===========================================");
        }
    }
}
```

---

## 4. Priority.cs (из старой системы)
```csharp
namespace LegacyNotifyLib
{
    /// <summary>
    /// Устаревший формат приоритета (для справки адаптеру)
    /// </summary>
    public static class LegacyPriorityCodes
    {
        public const byte Low = 10;
        public const byte Normal = 20;
        public const byte Critical = 30;
    }
}
```
