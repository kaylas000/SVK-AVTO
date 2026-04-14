# СВК Авто — сайт автосервиса

Полный репозиторий сайта автосервиса «СВК Авто» (ООО «СВК Авто Плюс») с AI-автоматизацией.

## Структура репозитория

```
├── _config.yml           # Jekyll конфигурация
├── _layouts/             # HTML шаблоны
│   ├── default.html
│   ├── page.html
│   ├── post.html
│   └── service.html
├── _includes/            # Переиспользуемые блоки
│   ├── header.html
│   ├── footer.html
│   ├── seo.html
│   ├── schema.html
│   └── chat-widget.html
├── _services/            # Услуги (коллекция Jekyll)
├── _locations/           # Локации (коллекция Jekyll)
├── _posts/               # Статьи блога (AI-генерация)
├── assets/               # Статика
│   ├── css/main.css
│   ├── js/main.js
│   └── preview/          # Превью картинки статей
├── pages/                # Статические страницы
├── worker/               # Cloudflare Worker
│   ├── src/
│   │   ├── index.ts      # Роутер
│   │   ├── chat.ts       # AI-консультант
│   │   ├── publisher.ts  # Публикация статей
│   │   ├── planner.ts    # Планировщик тем
│   │   ├── callback.ts   # Форма обратного звонка
│   │   ├── health.ts     # Health check
│   │   ├── turnstile.ts  # Валидация Turnstile
│   │   ├── quota.ts      # Квоты чата
│   │   ├── geo.ts        # Геофильтр
│   │   ├── ai-image.ts   # Генерация изображений
│   │   └── indexnow.ts   # IndexNow API
│   └── wrangler.toml
├── schema.sql            # D1 миграции
├── topics.sql            # Стартовый пул тем
├── robots.txt
├── sitemap.xml
├── llms.txt              # Карта для LLM-агентов
└── CNAME                 # Кастомный домен
```

## Быстрый старт

### 1. GitHub Pages

#### Настройка репозитория

1. Создайте репозиторий на GitHub
2. Загрузите все файлы
3. Включите GitHub Pages в настройках:
   - Source: Deploy from a branch
   - Branch: main / root
   - Custom domain: `svkautoplus.ru`

#### DNS настройки

Добавьте в DNS вашего домена:

```
; A-записи для apex домена
svkautoplus.ru.     A     185.199.108.153
svkautoplus.ru.     A     185.199.109.153
svkautoplus.ru.     A     185.199.110.153
svkautoplus.ru.     A     185.199.111.153

; CNAME для www
www.svkautoplus.ru. CNAME username.github.io.
```

⚠️ **Важно:** Не используйте wildcard DNS (`*.domain`) — GitHub не рекомендует.

#### HTTPS

В настройках GitHub Pages включите «Enforce HTTPS».

### 2. Cloudflare Worker

#### Установка Wrangler

```bash
npm install -g wrangler
```

#### Аутентификация

```bash
wrangler login
```

#### Создание D1 базы данных

```bash
wrangler d1 create svkauto-db
# Скопируйте database_id в wrangler.toml
```

#### Применение миграций

```bash
cd worker
wrangler d1 execute svkauto-db --file=../schema.sql
wrangler d1 execute svkauto-db --file=../topics.sql
```

#### Настройка секретов

```bash
# Обязательные секреты
wrangler secret put GROQ_API_KEY              # API ключ Groq
wrangler secret put GITHUB_TOKEN              # Fine-grained token с правами contents:write
wrangler secret put CF_ACCOUNT_ID             # Cloudflare Account ID
wrangler secret put CHAT_AUTH_TOKEN           # Токен для /run-now и /plan-now
wrangler secret put TURNSTILE_SECRET_KEY      # Turnstile Secret Key
wrangler secret put INDEXNOW_KEY              # Ключ для IndexNow

# Переменные (уже в wrangler.toml, можно изменить)
# SITE_BASE_URL, GITHUB_OWNER, GITHUB_REPO, GITHUB_BRANCH
```

#### Деплой Worker

```bash
cd worker
wrangler deploy
```

### 3. Turnstile (защита от ботов)

1. В Cloudflare Dashboard перейдите в Turnstile
2. Создайте новый виджет:
   - Site Key: для фронтенда (вставьте в `chat-widget.html`)
   - Secret Key: сохраните в секреты Worker

### 4. Cloudflare Bulk Redirects (301 редиректы)

Для старых URL из Joomla настройте в Cloudflare:

| Source URL | Target URL | Status |
|------------|------------|--------|
| `/about-us.html` | `/` | 301 |
| `/svk-uslugi.html` | `/services/` | 301 |
| `/tseny.html` | `/prices/` | 301 |
| `/stoimost-to.html` | `/prices/` | 301 |
| `/service-action.html` | `/promo/` | 301 |
| `/contact-us.html` | `/contacts/` | 301 |
| `/about-us/pravila-okazaniya-uslug.html` | `/rules/` | 301 |
| `/about-us/diskontnie-programmi.html` | `/prices/` | 301 |
| `/about-us/*` | `/` | 301 |
| `/svk-uslugi/*` | `/services/` | 301 |

## Тестирование

### Локальный запуск Worker

```bash
cd worker
wrangler dev
```

### Тест cron без деплоя

```bash
wrangler dev --test-scheduled
```

Затем в другом терминале:
```bash
curl "http://localhost:8787/__scheduled"
```

### Ручной запуск публикации

```bash
curl -X POST https://your-worker.your-subdomain.workers.dev/api/run-now \
  -H "Authorization: Bearer YOUR_CHAT_AUTH_TOKEN"
```

### Ручной запуск планировщика

```bash
curl -X POST https://your-worker.your-subdomain.workers.dev/api/plan-now \
  -H "Authorization: Bearer YOUR_CHAT_AUTH_TOKEN"
```

### Проверка статуса

```bash
curl https://your-worker.your-subdomain.workers.dev/api/health
```

## AI-Блог

### Расписание публикаций

- **3 статьи в неделю:** понедельник, среда, пятница
- **Cron:** ежедневно в 06:05 UTC (09:05 по Москве)
- **Планировщик:** воскресенье в 05:00 UTC

### Алгоритм выбора темы

1. Выбираются кандидаты из таблицы `topics`
2. Применяются cooldown-правила:
   - Нельзя повторять (system + angle) < 60 дней
   - Нельзя повторять system < 21 дней
   - Нельзя ставить подряд одну category
   - Нельзя в одну неделю 2 темы с одним system
3. Для топ-5 кандидатов генерируются идеи через Groq
4. Проверка simhash (hamming distance >= 10)
5. Первый прошедший — в календарь

### Структура статьи

```yaml
---
layout: post
title: "..."
description: "..."
date: YYYY-MM-DD
category: "..."
tags: [...]
image: /assets/preview/YYYY/MM/slug.png
---

# Контент статьи (Markdown)
```

## AI-Онлайн-консультант

### Уровни защиты

1. **Turnstile** — обязательная валидация перед первым сообщением
2. **Rate Limiting** — burst-защита на уровне Cloudflare
3. **Квота 5 вопросов** — точный учёт в D1, сброс ежедневно
4. **Геофильтр** — только Москва и Московская область
5. **CORS + Origin check** — только svkautoplus.ru

### Интенты

- `booking` — запись, адреса, телефоны
- `to` — техническое обслуживание
- `symptoms` — диагностика, симптомы
- `prices` — цены, прайс
- `guarantee` — гарантия, правила

## SEO

### Структурированные данные

- `WebSite` — на всех страницах
- `Organization` — с контактами и департаментами
- `AutoRepair` (LocalBusiness) — для каждой станции
- `BlogPosting` — для статей
- `BreadcrumbList` — хлебные крошки

### Мета-теги

- Уникальный `<title>` для каждого типа страницы
- `<meta description>` до 155 символов
- OpenGraph (og:title, og:description, og:image)
- Twitter Cards
- Canonical URL

### Файлы

- `robots.txt` — разрешено всё, кроме `/api/`
- `sitemap.xml` — статические страницы
- `llms.txt` — карта для LLM-агентов

## Разработка

### Добавление новой услуги

1. Создайте файл в `_services/service-name.md`
2. Используйте front matter с параметрами:
   - `title`, `description`, `service_name`
   - `when_needed` (список)
   - `process` (список)
   - `pricing` (markdown)
   - `warranty` (строка)
   - `faq` (массив объектов с question/answer)

### Добавление темы для блога

```sql
INSERT INTO topics (topic_id, category, system, angle, audience, priority)
VALUES ('unique-id', 'Категория', 'Система', 'Угол', 'аудитория', 5);
```

### Локальная разработка Jekyll

```bash
# Установка зависимостей
bundle install

# Локальный сервер
bundle exec jekyll serve

# Сборка
bundle exec jekyll build
```

## Переменные окружения

### GitHub Actions (опционально)

Для автоматического деплоя Worker:

```yaml
# .github/workflows/deploy.yml
name: Deploy Worker
on:
  push:
    branches: [main]
    paths: ['worker/**']
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          workingDirectory: worker
```

## Лицензия

© 2024–2025 ООО «СВК Авто Плюс». Все права защищены.

## Поддержка

При возникновении проблем:

1. Проверьте логи Worker: `wrangler tail`
2. Проверьте статус: `/api/health`
3. Проверьте таблицы D1: `wrangler d1 execute svkauto-db --command="SELECT * FROM calendar_slots"`
