# Asssistly Bot (Telegram AI Assistant)

Продакшн-бот для Telegram, который:
- готовит черновики ответов на входящие сообщения;
- перефразирует любой текст под выбранный стиль;
- поддерживает подписку/тарифы и пробный период;
- умеет работать с групповыми подсказками без сообщений в группу;
- запускается в `polling` или `webhook` режиме.

## Что умеет бот

### 1) Подготовить ответ
Пользователь отправляет сообщение собеседника -> бот генерирует готовый ответ в выбранном режиме:
- `Обычный`
- `Коротко`
- `Вежливо`
- `По делу`
- `Отказать`
- `Занят`

### 2) Перефразировать текст
Пользователь отправляет свой текст -> бот переписывает его в выбранном стиле (без ответа по смыслу):
- `💼 Деловой` (`business`)
- `🤝 Вежливый` (`polite`)
- `🙂 Дружелюбный` (`friendly`)
- `🧢 Неформальный` (`casual`)
- `⚡ Кратко` (`short`)
- `🏛 Официальный` (`formal`)
- `⚖️ Нейтральный` (`neutral`)

### 3) Подписка и trial
- Пробная подписка: `3 дня`.
- Тарифы:
  - Неделя — `150 ₽`
  - Месяц — `450 ₽`
  - Полгода — `1990 ₽`
  - Год — `3184 ₽`
- Оплата через YooKassa.

### 4) Работа с группами
- Бот **не пишет сообщения в группы**.
- Бот отслеживает сообщения в подключенных группах и присылает подсказки владельцу в личку.
- Для каждой группы можно включать/выключать подсказки из личного меню бота.

### 5) Онбординг и help
- Онбординг показывается на первом `/start`.
- В настройках есть повторный запуск онбординга.
- `/help` содержит команды и блок экстренной диагностики для пользователей.

## Технологии

- Node.js (ESM)
- Express
- Telegram Bot API (через `axios`)
- OpenAI API
- Prisma (частично, с fallback на file-state)
- YooKassa API

## Структура проекта

```text
.
├── index.js                     # основной сервер и логика бота
├── package.json
├── .env
├── data/
│   └── bot-state.json           # file-state fallback (пользователи/сессии)
├── prisma/
│   └── schema.prisma            # схема БД (SQLite)
└── docs/
    ├── BOT_ARCHITECTURE.md
    └── paraphraseStatement.examples.md
```

## Команды бота

- `/start` — главное меню
- `/status` — статус подписки и текущего режима
- `/profile` — синоним `/status`
- `/help` — справка
- `/onboarding` — пройти онбординг заново
- `/admin` — админ-панель (только для `ADMIN_IDS`)

## Конфиг `.env`

Пример:

```env
ADMIN_IDS=1441173568

TELEGRAM_BOT_TOKEN=...
TELEGRAM_WEBHOOK_SECRET=super-secret
TELEGRAM_MODE=polling
PUBLIC_URL=https://your-domain-or-tunnel
BOT_USERNAME=Assistly_bot

OPENAI_API_KEY=...
OPENAI_MODEL=gpt-4o-mini

DATABASE_URL="file:./dev.db"
PORT=4010

YOOKASSA_SHOP_ID=...
YOOKASSA_SECRET_KEY=...
YOOKASSA_WEBHOOK_SECRET=super-secret
```

### Важные переменные

- `TELEGRAM_MODE`
  - `polling` — бот сам делает `getUpdates`.
  - `webhook` — бот принимает апдейты на `POST /webhook/<TELEGRAM_WEBHOOK_SECRET>`.
- `PUBLIC_URL` — нужен для автоподнятия webhook.
- `OPENAI_API_KEY` — обязателен для полноценной AI-генерации (иначе fallback).
- `ADMIN_IDS` — список Telegram ID админов через запятую.

## Локальный запуск

```bash
npm install
npm start
```

Сервер поднимется на `PORT` (по умолчанию `4010`).

Health-check:

```bash
curl -sS http://localhost:4010/health
```

## Запуск в polling / webhook

### Polling

Используйте:

```env
TELEGRAM_MODE=polling
```

При старте бот вызывает `deleteWebhook` и работает через `getUpdates`.

### Webhook

Используйте:

```env
TELEGRAM_MODE=webhook
PUBLIC_URL=https://your-domain
```

На старте бот попытается сделать:
- `setWebhook` на `${PUBLIC_URL}/webhook/${TELEGRAM_WEBHOOK_SECRET}`
- `getWebhookInfo` для проверки.

## Деплой 24/7 (сервер + PM2)

```bash
cd /root/bot
git pull
npm ci
pm2 start index.js --name tgbot
pm2 save
pm2 startup
```

Проверка:

```bash
pm2 status
pm2 logs tgbot --lines 80
curl -sS http://localhost:4010/health
```

## Как обновлять бота

На сервере:

```bash
cd /root/bot
git pull
npm ci
pm2 restart tgbot --update-env
pm2 logs tgbot --lines 80
```

## Проверка OpenAI

```bash
set -a; source .env; set +a
curl -sS https://api.openai.com/v1/models -H "Authorization: Bearer $OPENAI_API_KEY"
```

Ожидаем `"data":[...]`.  
Если `invalid_api_key` — ключ неверный/отозван.

## Частые проблемы

### 1) `409 Conflict: terminated by other getUpdates request`
Причина: запущены 2 инстанса бота одновременно.

Решение:
- Оставить только один процесс (или только серверный PM2, или только локальный запуск).
- Для polling можно сбросить webhook:

```bash
curl -sS -X POST "https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/deleteWebhook"
```

### 2) `invalid_api_key`
Причина: неверный/отозванный OpenAI ключ.

Решение:
- создать новый ключ в OpenAI;
- обновить `OPENAI_API_KEY` в `.env`;
- перезапустить процесс с `--update-env`.

### 3) `403 Country, region, or territory not supported`
Причина: региональное ограничение OpenAI.

Решение:
- запускать бот в поддерживаемом регионе/через соответствующую инфраструктуру.

### 4) `Cannot find module .../dist/index.js`
Причина: запуск не из той папки или не тот `package.json`.

Решение:
- перейти в корень проекта (где `scripts.start = node index.js`);
- запускать `npm start` только оттуда.

### 5) `Connection closed by <ip> port 22` (SSH)
Причина: проблемы SSH на сервере (sshd/фаервол/бан).

Проверьте в web-консоли провайдера:

```bash
systemctl enable ssh --now
systemctl restart ssh
ufw allow 22/tcp
ufw status
```

## Безопасность

- Не коммитьте рабочие ключи в git.
- Регулярно ротируйте `OPENAI_API_KEY`, `TELEGRAM_BOT_TOKEN`, `YOOKASSA_SECRET_KEY`.
- После утечки ключа — немедленно revoke и выпуск нового.
- Ограничьте права API-ключей минимально необходимыми.

## Примечания по архитектуре

- Основная логика в `index.js`.
- Сессии пользователей — in-memory.
- Часть данных хранится в `data/bot-state.json` как fallback.
- При проблемах Prisma код переключается на file-state fallback, чтобы бот продолжал работать.

## Дополнительная документация

- Архитектура: [`docs/BOT_ARCHITECTURE.md`](docs/BOT_ARCHITECTURE.md)
- Примеры перефразирования: [`docs/paraphraseStatement.examples.md`](docs/paraphraseStatement.examples.md)

---

Если нужна, можно вынести `index.js` на модули (`bot/handlers`, `services/ai`, `services/payments`, `infra/telegram`) и добавить CI-валидацию перед деплоем.
