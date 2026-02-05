# API

Ниже — публичные HTTP контракты (то, что видит клиент).
Внутренние структуры и сценарии описаны в `docs/ARCHITECTURE.md`.

> Во всех ответах есть `X-Correlation-Id`. Клиент может прислать его сам.

---

## POST /hs/auth/token

**Назначение:** получить JWT по PAT (основной путь) или Basic (резерв).

### Заголовки
- `Authorization: PAT pat.<uuid>.<secretHex>`
  - `uuid` — UUID токена PAT
  - `secretHex` — 64 hex символа
- или: `Authorization: Basic <base64(login:password)>`

### Ответ 200
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9....",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

---

## GET /hs/chats/contacts

**Назначение:** пример “боевого” эндпойнта. Дальше вы расширяете по аналогии.

### Заголовки
- `Authorization: Bearer <jwt>`

### Ответ 200 (пример)
```json
{
  "items": [
    { "id": "....", "name": "Alice" },
    { "id": "....", "name": "Bob" }
  ]
}
```

---

## Формат ошибок

Любая ошибка возвращается как:
```json
{
  "ошибка": {
    "код": "API.Доступ.AUTHN.NO_AUTH",
    "сообщение": "Не авторизован.",
    "корреляция": "..."
  }
}
```

Коды HTTP:
- `400` — некорректные входные данные/формат
- `401` — не авторизован (AUTHN)
- `403` — доступ запрещён (AUTHZ)
- `500` — внутренняя ошибка / конфигурация / инфраструктура

