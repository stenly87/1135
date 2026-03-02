# Лабораторная работа: Паттерн «Фабричный метод» (Factory Method)

## 1. Цель работы
*   Изучить и реализовать паттерн **Factory Method**.
*   Научиться работать с сериализацией JSON и файловым вводом/выводом.
*   Освоить базовые принципы работы с внешними библиотеками для генерации файлов (PDF, XLSX).

## 2. Задание
Разработать консольное приложение **«Экспортатор успеваемости»**.  
Приложение считывает список студентов из JSON-файла и позволяет пользователю выбрать формат экспорта отчёта (PDF, TXT или XLSX). Создание конкретного класса экспорта должно осуществляться через **Фабричный метод**.
В методе `Main` не должно быть прямых созданий объектов через `new PdfDocument()`. Только через Фабрику (`new PdfFactory().CreateDocument()`).

## 3. Входные данные (JSON)
Создайте файл `students.json` со следующей структурой данных:

```json
{
  "group": "1135",
  "semester": "Осень 2023",
  "students": [
    {
      "id": 1,
      "name": "Иванов Иван",
      "grade": 5
    },
    {
      "id": 2,
      "name": "Петров Петр",
      "grade": 4
    },
    {
      "id": 3,
      "name": "Сидорова Анна",
      "grade": 5
    },
    {
      "id": 4,
      "name": "Кузнецов Алексей",
      "grade": 3
    }
  ]
}
```

## 4. Схема взаимодействия классов
Необходимо реализовать следующую иерархию:

1.  **Product (Продукт):** Интерфейс `IExportDocument`.
    *   Метод: `Save(string path, List<Student> data)`.
2.  **Concrete Products (Конкретные продукты):**
    *   `PdfDocument` (реализует `IExportDocument`).
    *   `TxtDocument` (реализует `IExportDocument`).
    *   `XlsxDocument` (реализует `IExportDocument`).
3.  **Creator (Создатель):** Абстрактный класс `DocumentFactory`.
    *   Метод: `CreateDocument()` (возвращает `IExportDocument`).
4.  **Concrete Creators (Конкретные создатели):**
    *   `PdfFactory` (наследует `DocumentFactory`, возвращает `PdfDocument`).
    *   `TxtFactory` (наследует `DocumentFactory`, возвращает `TxtDocument`).
    *   `XlsxFactory` (наследует `DocumentFactory`, возвращает `XlsxDocument`).
5.  **Client (Клиент):** Класс `Program`.
    *   Считывает JSON.
    *   Спрашивает у пользователя тип файла.
    *   Инициализирует нужную Фабрику.
    *   Вызывает `CreateDocument()` и затем `Save()`.
  
В методе Main должно получиться что-то вроде следующего кода:
```
DocumentFactory factory;
...
factory = new PdfFactory();
...
IExportDocument document = factory.CreateDocument();
...
document.Save(path, data);
```

## 5. Примеры реализации 

Внутри классов-продуктов (`PdfDocument`, `XlsxDocument`) вам потребуется использовать сторонние библиотеки. Ниже приведены примеры того, как инициировать создание файла с их помощью.

### Пример для PDF (библиотека QuestPDF)
*Установить пакет:* `QuestPDF`
https://www.nuget.org/packages/QuestPDF
```csharp
// Внутри метода Save класса PdfDocument
Document.Create(container =>
{
    container.Page(page =>
    {
        page.Content().Column(column =>
        {
            foreach (var student in data)
            {
                column.Item().Text($"{student.Name} - {student.Grade}");
            }
        });
    });
})
.GeneratePdf(path);
```

### Пример для XLSX (библиотека ClosedXML)
*Установить пакет:* `ClosedXML`
https://www.nuget.org/packages/ClosedXML/
```csharp
// Внутри метода Save класса XlsxDocument
using (var workbook = new XLWorkbook())
{
    var worksheet = workbook.Worksheets.Add("Отчёт");
    int row = 1;
    
    foreach (var student in data)
    {
        worksheet.Cell(row, 1).Value = student.Name;
        worksheet.Cell(row, 2).Value = student.Grade;
        row++;
    }
    
    workbook.SaveAs(path);
}
```

## 6. Требования к реализации
1.  **Паттерн:** 
2.  **Linux-совместимость:**
    *   Не использовать хардкод путей (например, `C:\Reports`). Использовать `Path.Combine` и относительные пути.
    *   Учитывать регистр имен файлов (Linux case-sensitive).
3.  **Код:**
    *   Использовать `async/await` для работы с файлами.
    *   Классы должны находиться в отдельных файлах.
  
## 7. Улучшение приложения
    *   Создайте фабрики для чтения файлов разных форматов: json, xml, xlsx
      
