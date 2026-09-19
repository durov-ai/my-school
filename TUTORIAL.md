# Полный туториал: запуск сайта школы + Telegram-бота с нуля

Этот гайд проведёт тебя через весь процесс — от чистого VPS до полностью
рабочего сайта, на который бот может добавлять учителей, отличников и фото
галереи. Пошагово, ничего не пропускай.

**Оценка времени:** 30–40 минут при первом разе.

---

## Что тебе понадобится заранее

1. **VPS-сервер** с Ubuntu 22.04 или новее, доступ по SSH (root или sudo-пользователь).
2. **Домен** (необязательно, но желательно) — например `school-example.uz`, направленный на IP твоего VPS через A-запись. Без домена сайт тоже будет работать по IP, но без HTTPS.
3. **Telegram-аккаунт** — для создания бота и получения своего Telegram ID.
4. Файлы проекта — распакуй архив `school-project/` (backend, bot, data, site), который прислал Claude, на свой компьютер.

---

## Шаг 1. Создать Telegram-бота

1. Открой Telegram, найди **@BotFather**.
2. Отправь `/newbot`.
3. Придумай имя бота (например `School Admin Bot`) и юзернейм, оканчивающийся на `bot` (например `school1_admin_bot`).
4. BotFather пришлёт тебе **токен** вида `123456789:AAExampleTokenString`. Скопируй и сохрани его — он понадобится ниже.

## Шаг 2. Узнать свой Telegram ID

1. Найди в Telegram бота **@userinfobot**.
2. Отправь ему любое сообщение.
3. Он пришлёт твой числовой ID, например `123456789`. Сохрани его — это будет единственный ID, у которого есть доступ к боту-администратору.

Если админов будет несколько — попроси каждого написать @userinfobot и собери все ID через запятую.

---

## Шаг 3. Подключиться к VPS и подготовить сервер

Подключись по SSH:

```bash
ssh root@ТВОЙ_IP_АДРЕС
```

Обнови систему и поставь необходимое:

```bash
apt update && apt upgrade -y
apt install -y python3 python3-venv python3-pip nginx git ufw
```

Открой нужные порты в firewall:

```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable
```

---

## Шаг 4. Загрузить файлы проекта на сервер

Проще всего — через `scp` с твоего компьютера (не с самого VPS). На своём компьютере, в папке, где лежит `school-project`:

```bash
scp -r school-project root@ТВОЙ_IP_АДРЕС:/opt/school-project
```

Или, если тебе удобнее через git — залей `school-project` в приватный репозиторий на GitHub и на сервере сделай:

```bash
cd /opt
git clone https://github.com/твой-юзернейм/school-project.git
```

После загрузки у тебя на сервере должна быть структура:

```
/opt/school-project/
├── backend/
│   ├── main.py
│   └── requirements.txt
├── bot/
│   ├── bot.py
│   ├── config.py
│   └── requirements.txt
├── data/
│   ├── teachers.json
│   ├── pride.json
│   └── gallery.json
└── site/
    └── index.html
```

---

## Шаг 5. Настроить и запустить Backend (FastAPI)

Зайди на сервер (если ещё не там) и перейди в папку backend:

```bash
cd /opt/school-project/backend
```

Создай виртуальное окружение и установи зависимости:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Придумай секретный токен** для связи бота с сервером — любую длинную случайную строку. Можно сгенерировать так:

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

Скопируй результат — это будет `BOT_API_TOKEN`. Используй его и в backend, и в боте (значения должны совпадать!).

Запусти сервер вручную для проверки:

```bash
export BOT_API_TOKEN="сюда_вставь_сгенерированную_строку"
uvicorn main:app --host 0.0.0.0 --port 8000
```

Если всё правильно — увидишь что-то вроде `Uvicorn running on http://0.0.0.0:8000`.

Проверь в браузере (замени на свой IP): `http://ТВОЙ_IP:8000` — должен открыться сайт школы (пока с пустыми разделами учителей/отличников/галереи, если данные ещё не подгрузились — это нормально, если ты не подождал секунду).

Останови сервер (`Ctrl+C`) — сейчас настроим его как постоянно работающий сервис.

### Сделать backend постоянным сервисом (systemd)

Создай файл сервиса:

```bash
nano /etc/systemd/system/school-backend.service
```

Вставь (замени `СЮДА_ТВОЙ_ТОКЕН` на реальный токен):

```ini
[Unit]
Description=School Site Backend
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/school-project/backend
Environment="BOT_API_TOKEN=СЮДА_ТВОЙ_ТОКЕН"
ExecStart=/opt/school-project/backend/venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Сохрани (`Ctrl+O`, `Enter`, `Ctrl+X`) и запусти:

```bash
systemctl daemon-reload
systemctl enable school-backend
systemctl start school-backend
systemctl status school-backend
```

Ты должен увидеть `active (running)` зелёным цветом. Если ошибка — смотри логи:

```bash
journalctl -u school-backend -f
```

---

## Шаг 6. Настроить и запустить Telegram-бота

Перейди в папку бота:

```bash
cd /opt/school-project/bot
```

Создай отдельное виртуальное окружение и установи зависимости:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Сделать бота постоянным сервисом

Создай файл сервиса:

```bash
nano /etc/systemd/system/school-bot.service
```

Вставь (подставь свои реальные значения — токен бота из Шага 1, тот же `BOT_API_TOKEN` из Шага 5, и свой Telegram ID из Шага 2):

```ini
[Unit]
Description=School Site Telegram Bot
After=network.target school-backend.service

[Service]
Type=simple
WorkingDirectory=/opt/school-project/bot
Environment="BOT_TOKEN=ТОКЕН_ОТ_BOTFATHER"
Environment="BOT_API_TOKEN=СЮДА_ТОТ_ЖЕ_ТОКЕН_ЧТО_И_В_BACKEND"
Environment="BACKEND_URL=http://127.0.0.1:8000"
Environment="ADMIN_IDS=123456789"
ExecStart=/opt/school-project/bot/venv/bin/python bot.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Если админов несколько — впиши их ID через запятую без пробелов: `ADMIN_IDS=123456789,987654321`.

Сохрани и запусти:

```bash
systemctl daemon-reload
systemctl enable school-bot
systemctl start school-bot
systemctl status school-bot
```

Снова должно быть `active (running)`.

### Проверка бота

Открой Telegram, найди своего бота по юзернейму (тот, что ты придумал в Шаге 1), напиши `/start`.

Если ты в списке `ADMIN_IDS` — увидишь меню с кнопками "Добавить учителя / Добавить отличника / Добавить фото". Если нет — бот вежливо откажет и покажет твой ID (на случай, если ты его неправильно скопировал).

Попробуй добавить тестового учителя, пройдя весь диалог. В конце бот напишет "✅ O'qituvchi muvaffaqiyatli qo'shildi!" — значит всё работает.

---

## Шаг 7. Настроить Nginx (чтобы сайт открывался по обычному адресу, без `:8000`)

Сейчас сайт доступен только по `http://ТВОЙ_IP:8000`. Настроим Nginx, чтобы он открывался по обычному домену/IP на 80 порту.

```bash
nano /etc/nginx/sites-available/school-site
```

Вставь (если есть домен — впиши его вместо IP в `server_name`):

```nginx
server {
    listen 80;
    server_name ТВОЙ_ДОМЕН_ИЛИ_IP;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Активируй конфиг:

```bash
ln -s /etc/nginx/sites-available/school-site /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

`nginx -t` должен показать `syntax is ok` и `test is successful`. Теперь сайт открывается по адресу `http://ТВОЙ_ДОМЕН_ИЛИ_IP` без указания порта.

### (Рекомендуется) Настроить HTTPS через Let's Encrypt

Если у тебя есть домен (не просто IP):

```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d ТВОЙ_ДОМЕН
```

Следуй инструкциям на экране (введи email, согласись с условиями). Certbot сам настроит HTTPS и автопродление сертификата.

---

## Шаг 8. Финальная проверка всего вместе

1. Открой сайт в браузере — `http://ТВОЙ_ДОМЕН_ИЛИ_IP`.
2. Пройди гейт выбора языка → должна открыться главная страница.
3. Перейди в раздел "O'qituvchilar" (Учителя) — должны отобразиться 8 стартовых учителей.
4. Перейди в "Faxrimiz" (Гордость школы) — должны отобразиться 6 отличников.
5. Перейди в "Galereya" — должны отобразиться 8 стартовых фото.
6. Открой Telegram-бота, добавь нового тестового учителя через диалог.
7. Обнови страницу сайта (F5) — новый учитель должен появиться в списке.

Если всё это сработало — система полностью развёрнута и готова к работе.

---

## Как это использовать дальше

- Чтобы добавить нового учителя/отличника/фото — просто открой бота в Telegram и следуй диалогу. Изменения появятся на сайте сразу после обновления страницы (F5), никакой пересборки или перезапуска не требуется.
- Данные хранятся в `/opt/school-project/data/*.json` — если когда-нибудь понадобится отредактировать что-то вручную или сделать резервную копию, это именно те файлы.
- **Рекомендация:** настрой регулярный бэкап папки `data/` (например, простой cron-задачей, копирующей файлы в другое место раз в день) — на случай случайного удаления записи через бота.

---

## Частые проблемы и решения

**Бот не отвечает на `/start`**
→ Проверь `systemctl status school-bot`. Если не запущен — смотри `journalctl -u school-bot -f` на предмет ошибок (чаще всего — неверный `BOT_TOKEN`).

**Бот пишет "Ulanishda xatolik" при добавлении данных**
→ Backend не отвечает. Проверь `systemctl status school-backend`. Также убедись, что `BOT_API_TOKEN` в обоих сервис-файлах (`school-backend.service` и `school-bot.service`) совпадает **дословно**.

**Сайт открывается, но разделы "Учителя"/"Гордость"/"Галерея" пустые с ошибкой**
→ Открой консоль браузера (F12 → Console) и посмотри на ошибку. Скорее всего backend недоступен — проверь `systemctl status school-backend` и попробуй открыть напрямую `http://ТВОЙ_ДОМЕН/api/teachers` в браузере — должен вернуться JSON-список.

**После правки .service файла изменения не применяются**
→ Не забывай `systemctl daemon-reload` перед `systemctl restart <имя-сервиса>`.

**Хочу добавить ещё одного администратора бота**
→ Отредактируй `ADMIN_IDS` в `/etc/systemd/system/school-bot.service`, затем:
```bash
systemctl daemon-reload
systemctl restart school-bot
```

**Хочу удалить неправильно добавленную запись**
→ Пока в боте нет команды удаления через диалог (это можно добавить позже при необходимости). Временное решение — зайти на сервер и вручную отредактировать соответствующий JSON-файл в `/opt/school-project/data/`, либо использовать `DELETE /api/teachers/{id}` (и аналогичные для pride/gallery) через `curl` с заголовком `X-Bot-Token`:
```bash
curl -X DELETE http://127.0.0.1:8000/api/teachers/ID_ЗАПИСИ \
  -H "X-Bot-Token: ТВОЙ_BOT_API_TOKEN"
```
ID записи можно посмотреть, открыв `/api/teachers` в браузере — он в JSON у каждой записи.

---

Готово! Теперь у тебя полноценная система: сайт + backend + Telegram-бот для управления контентом, всё работает автономно на твоём сервере.
