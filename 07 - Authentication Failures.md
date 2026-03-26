# Ошибки идентификации и аутентификации (Authentication Failures)

### Цель работы

Выявить ошибки конфигурации веб-сервера и приложения, которые могут привести к раскрытию чувствительной информации или созданию дополнительных векторов атак.

## Ход выполнения

### 1) Разведка

Для поиска скрытых файлов и директорий на сайте `https://duck-store.escape.tech` был использован инструмент **ffuf**. Использовал <!-- проверенный --> словари, содержащие наиболее вероятные пути, характерные для приложений на этом фреймворке. 
<!--
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
-->

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

### 3) Эксплуатация

Получив из открытого эндпоинта `/api/v1/users/` список UUID всех пользователей, была проведена эксплуатация IDOR-уязвимости, описанной в документации. Для этого использовался эндпоинт `/api/v1/users/{user_uuid}`, который, согласно комментариям разработчиков, не проверяет права доступа и возвращает полную конфиденциальную информацию.

**Пример получения данных пользователя `admin`:**

<img width="905" height="102" alt="изображение" src="https://github.com/user-attachments/assets/0c9e9d7f-e2f8-4261-89a7-bdefda71673e" />

Анализ файла `openapi.json` также показал, что эндпоинт `/api/v1/users/me/profile` с методом PUT позволяет изменять профиль текущего пользователя. В схеме `UserUpdate` присутствует поле `role`, что предполагает возможность изменения роли. Был выполнен следующий запрос:

```bash
curl -X PUT https://duck-store.escape.tech/api/v1/users/me/profile \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiOWJmOTAxMzctNjY3My00YTFhLTg0YjMtMzY1NzMyMjlmZjI1IiwidXNlcm5hbWUiOiJib2IiLCJyb2xlIjoidXNlciIsImV4cCI6MTc3Mzg1MTc2OX0.aFG7EAceepdpv3EjLbh4MIrtAOH1Sr4GDal1MfGMyMk" \
  -H "Content-Type: application/json" \
  -d '{"role": "admin"}'
```

- где `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiOWJmOTAxMzctNjY3My00YTFhLTg0YjMtMzY1NzMyMjlmZjI1IiwidXNlcm5hbWUiOiJib2IiLCJyb2xlIjoidXNlciIsImV4cCI6MTc3Mzg1MTc2OX0.aFG7EAceepdpv3EjLbh4MIrtAOH1Sr4GDal1MfGMyMk` - текущий JWT-токен.

Сервер успешно выполнил запрос изменил данные пользователя:

<img width="908" height="174" alt="изображение" src="https://github.com/user-attachments/assets/c5a4aefe-ad84-4632-b48d-2d2493de0634" />

По итогу был получен доступ к Админ-панели:

<img width="1142" height="659" alt="изображение" src="https://github.com/user-attachments/assets/54802de0-cab3-4aa1-b982-7678570f1db4" />

## Выводы о защищенности

В результате анализа выявлена критическая ошибка конфигурации — общедоступный файл спецификации API `/openapi.json`. Данный файл раскрывает полную структуру приложения и, что наиболее опасно, содержит подробные комментарии разработчиков о наличии уязвимостей.

Атака классифицируется как:
- **CWE-215: Information Exposure through Debug Information** - раскрытие отладочной информации
- **CWE-200: Exposure of Sensitive Information** - раскрытие чувствительной информации
- **CWE-269: Improper Privilege Management** - неправильное управление привилегиями
