# Неправильная конфигурация (Security Misconfiguration)

### Цель работы

Выявить ошибки конфигурации веб-сервера и приложения, которые могут привести к раскрытию чувствительной информации или созданию дополнительных векторов атак.

## Ход выполнения

### 1) Разведка

Для поиска скрытых файлов и директорий на сайте `https://duck-store.escape.tech` был использован инструмент **ffuf**. Использовал проверенный словарь, содержащий наиболее вероятные пути, характерные для приложений на этом фреймворке:

```
api
api/v1
api/v1/docs
api/v1/openapi.json
api/v1/auth
api/v1/users
api/v1/products
api/v1/admin
api/v1/cart
api/v1/orders
api/v1/testimonials
api/v1/uploads
docs
redoc
openapi.json
swagger.json
swagger-ui
api-docs
```

Для фильтрации ответов использован параметр `-fs 4621`, так как стандартная страница 404 имеет фиксированный размер. Это позволяет отсеять ложноположительные срабатывания и увидеть только реально существующие пути.

**Команда для перебора:**
```bash
ffuf -u https://duck-store.escape.tech/FUZZ -w api_paths.txt -c -fs 4621
```

В результате сканирования был обнаружены пути: 

<img width="726" height="134" alt="изображение" src="https://github.com/user-attachments/assets/aa763b41-3558-442f-9666-244e11f55c55" />

### 2) Атака

При переходе по адресу `https://duck-store.escape.tech/openapi.json` сервер вернул JSON-файл, содержащий полную спецификацию API:

<img width="1143" height="419" alt="изображение" src="https://github.com/user-attachments/assets/caf355d2-6408-4b6e-86af-6e0d3c856463" />

Анализ файла показал, что он содержит не только описание всех эндпоинтов, но и, что наиболее критично, **комментарии разработчиков с подробным описанием уязвимостей**, присутствующих в приложении.
**Примеры обнаруженных уязвимостей из документации:**

| Эндпоинт | Уязвимость | Описание из документации |
|----------|------------|--------------------------|
| `/api/v1/users/{user_uuid}` | **IDOR** | "VULNERABLE: IDOR - This endpoint exposes ALL user information including email, account credit, role. No authorization check" |
| `/api/v1/admin/users` | **Broken Access Control** | "VULNERABLE: Broken Access Control - Any authenticated user can access this admin endpoint" |
| `/api/v1/orders/coupons` | **Information Disclosure** | "VULNERABLE: Lists ALL coupon codes including internal/secret ones. No authentication required" |
| `/api/v1/cart/add` | **Business Logic Error** | "VULNERABLE: No validation on quantity - negative values are accepted, resulting in negative totals" |
| `/api/v1/uploads/fetch-url` | **SSRF** | "VULNERABLE: SSRF - Fetches any URL without validation, allowing access to internal services" |
| `/api/v1/products/filter/by-color` | **SQL Injection** | "VULNERABLE: SQL Injection - sort parameter directly interpolated into SQL query" |

При переходе по адресу `https://duck-store.escape.tech/api/v1/users/` сервер возвращает информацию пользователей, которая включает поле **id** и **username**:

<img width="905" height="144" alt="изображение" src="https://github.com/user-attachments/assets/5ea55a9e-9b6f-4172-b024-825edf201377" />



