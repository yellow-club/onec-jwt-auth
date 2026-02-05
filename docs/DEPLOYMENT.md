# Деплой и публикации (2× default.vrd)

В этом проекте intentionally используются **две публикации** одной и той же ИБ:

1) **Публикация AUTH** — выдача JWT по PAT/Basic
2) **Публикация API** — бизнесовые HTTP-сервисы, закрытые JWT на периметре

Почему так: у публикации API включён `<accessTokenAuthentication>`, а для выдачи токена это не нужно (и мешает).

> JWT-аутентификация платформы доступна только для опубликованных ИБ.

---

## 0) Перед началом

### Версия платформы
- Минимально поддерживаемая: **8.3.27.1936**


### Ключи (RS256)

Для RS256 нужна **пара** ключей в формате PEM:

- **`jwt_private.pem`** — закрытый ключ для подписи JWT *внутри 1С*  
  Хранится в константе `СекретныйКлючПодписиJWT`. Рекомендуемый путь загрузки: обработка **`Тестирование` → «Добавить секретный ключ»**.
- **`jwt_public.pem`** — публичный ключ для проверки подписи JWT *на периметре*  
  Его нужно указать в `default.vrd` публикации **API** (атрибут `keyInformation`).

Пример генерации через OpenSSL:

```bash
# private key (RSA 2048)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out jwt_private.pem

# public key из private
openssl pkey -in jwt_private.pem -pubout -out jwt_public.pem
```

> В `keyInformation` для HS* алгоритмов хранится симметричный ключ, а для RS256 — **публичный** ключ.

---

## 1) Публикация AUTH (выдача JWT)

Цель: разрешить `POST /hs/auth/token` по PAT/Basic.

- Публикуете ИБ на веб‑сервере как обычно
- В `default.vrd` **НЕ добавляете** `<accessTokenAuthentication>` для HTTP-сервисов

Рекомендации:
- ограничьте доступ к публикации (IP allowlist, VPN, внутренний контур)
- включите HTTPS

---

## 2) Публикация API (боевые эндпойнты, Bearer JWT)

Цель: закрыть HTTP-сервисы JWT аутентификацией на периметре.

### Пример фрагмента `default.vrd`

Согласно админ‑гайду, `<accessTokenAuthentication>` может быть подчинён `<httpServices>` — тогда JWT будет применяться только к HTTP‑сервисам.

```xml
<httpServices enable="true">
  <!-- ... ваши сервисы ... -->

  <accessTokenAuthentication>
    <!-- aud, который ожидает платформа -->
    <accessTokenRecepientName>api_service</accessTokenRecepientName>

    <issuers>
      <issuer
        name="limitedAccess"
        authenticationClaimName="sub"
        authenticationUserPropertyName="name"
        keyInformation="-----BEGIN PUBLIC KEY-----...-----END PUBLIC KEY-----"
      />
    </issuers>
  </accessTokenAuthentication>
</httpServices>
```

Что означает этот конфиг:
- `accessTokenRecepientName` сравнивается с `aud` в payload.
- `issuer/@name` сравнивается с `iss` в payload.
- `authenticationClaimName` (по умолчанию `sub`) — какое поле токена использовать для поиска пользователя.
- `authenticationUserPropertyName="name"` — искать 1С‑пользователя по его `Name`.

> Важно: в примере `keyInformation` указан одной строкой. На практике удобнее хранить PEM без переносов строк или с XML‑экранированием переносов.

---

## 3) Сопоставление с конфигурацией

В конфигурации значения по умолчанию такие:
- `iss` = `limitedAccess`
- `aud` содержит `api_service`
- `sub` = `jwt_service` (технический пользователь публикации)

Если вы меняете эти значения — делайте это **согласованно**:
- `ПолитикаJWT` (в конфигурации)
- `default.vrd` (в публикации API)

---

## 4) Две публикации — два набора VRD

Рекомендуем хранить в репозитории шаблоны:
- `deploy/vrd/auth.default.vrd.example`
- `deploy/vrd/api.default.vrd.example`

И дальше поддерживать их как “живую документацию”.

