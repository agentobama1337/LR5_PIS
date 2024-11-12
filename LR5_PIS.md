# ТЗ на реализацию бэкенда страницы откликов и приглашений на стажировку

## 1. Входные параметры

| Параметр | Тип параметра | Обязательность | Описание |
|----------|---------------|----------------|-----------|
| userId | uuid | + | Идентификатор пользователя |
| companyId | uuid | - | Идентификатор компании |
| status | number | - | Статус откликов (0 - новые, 1 - принятые, -1 - отказ) |
| search | string | - | Поисковая строка для фильтрации откликов |
| page | number | - | Номер страницы для пагинации |
| limit | number | - | Количество элементов на странице |

## 2. Авторизация

### a. По бизнесу

- Доступ к списку откликов/приглашений предоставляется только авторизованным пользователям
- Пользователь может видеть только свои отклики/приглашения

### b. Техническая реализация
```sql
-- Проверка прав доступа к откликам/приглашениям пользователя
SELECT id FROM tbl_user_responses 
WHERE user_id = {userId}
AND company_id = {companyId}
```

## 3. Передача данных при инициализации

```json
{
  "applications": [
    {
      "company": "ИЛ АЙ-ТЕКО",  
      "position": "IT-рекрутер",
      "date": "24.10.2024",
      "status": 0,
      "location": "Москва",
      "companyLogo": "https://example.com/logos/il-iteco.png"
    },
    {
      "company": "Bellerage",
      "position": "Стажер-аналитик",
      "date": "24.10.2024",
      "status": 0,
      "location": "Удаленно",
      "companyLogo": "https://example.com/logos/bellerage.png"
    },
    {
      "company": "ООО НПП \"Доза\"",
      "position": "Видеомонтажер",
      "date": "18.09.2024",
      "status": 1,
      "location": "Зеленоград",
      "companyLogo": "https://example.com/logos/doza.png"
    }
  ],
  "pagination": {
      "currentPage": 1,
      "totalPages": 5,
      "totalItems": 13,
      "itemsPerPage": 10
  }
}
```

### Описание атрибутов

| Атрибут, уровень 1 | Уровень 2 | Тип | Название атрибута | Формирование на бэкенде | Обязательность |
|--------------------|-----------|-----|-------------------|-------------------------|----------------|
| applications | company | string | Название компании | SELECT company FROM tbl_companies WHERE id = {companyId} | + |
| applications | position | string | Должность | SELECT position FROM tbl_user_responses WHERE user_id = {userId} AND company_id = {companyId} | + |
| applications | date | TIMESTAMP | Дата подачи заявки | SELECT date FROM tbl_user_responses WHERE user_id = {userId} AND company_id = {companyId} | + |
| applications | status | number | Статус заявки | SELECT status FROM tbl_user_responses WHERE user_id = {userId} AND company_id = {companyId} | + |
| applications | location | string | Место работы | SELECT location FROM tbl_user_responses WHERE user_id = {userId} AND company_id = {companyId} | + |
| applications | companyLogo | string | Ссылка на логотип компании | SELECT logo FROM tbl_companies WHERE id = {companyId} | + |
| pagination | currentPage | number | Текущая страница | Входной параметр page | + |
| pagination | totalPages | number | Всего страниц | CEIL(COUNT(*) / {limit}) | + |

## 4. REST-запросы

### 4.1 Получение списка купленных курсов

| Название | Получение списка откликов/приглашений пользователя |
|----------|-------------------------------------|
| URL | /api/user/responses |
| Тип метода | GET |
| Проверка авторизации | Базовая авторизация |
| Действия на бэкенде | SELECT c.logo, c.name, r.position, r.date, r.status, c.locationFROM tbl_user_responses rJOIN tbl_companies c ON r.company_id = c.idWHERE r.user_id = {userId}AND r.status LIKE COALESCE({status}, r.status)AND (c.name LIKE '%{search}%' OR r.position LIKE '%{search}%')LIMIT {limit} OFFSET {(page - 1) * limit} |
| Query parameters | page: номер страницы<br>limit: элементов на странице<br>status: фильтрация по статусу (опционально)<br>search: поисковый запрос (опционально) |
| Responses | 200 OK: [...список откликов/приглашений]<br>401 Unauthorized: {"message": "Требуется авторизация"} |

### 4.2 Обновление статуса отклика/приглашения

| Название | Обновление статуса отклика/приглашения |
|----------|----------------------------------------|
| URL | /api/user/responses/{companyId}/status |
| Тип метода | PUT |
| Проверка авторизации | Базовая авторизация |
| Действия на бэкенде | UPDATE tbl_user_responses SET status = {status} WHERE user_id = {userId} AND company_id = {companyId} |
| Query parameters | {"status": number} |
| Responses | 200 OK: {"message": "Статус обновлен"}<br>400 Bad Request: {"message": "Некорректные данные"}<br>404 Not Found: {"message": "Отклик не найден"} |