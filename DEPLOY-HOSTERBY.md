# Деплой NEO PRO SISTEM на hoster.by

## Важно: какой хостинг нужен

Это **не** сайт на PHP/HTML. Это **Node.js SSR-приложение** (TanStack Start + Vite + Nitro).

| Тип тарифа hoster.by | Подойдёт? |
| --- | --- |
| Обычный виртуальный хостинг только PHP/MySQL | **Нет** |
| Виртуальный хостинг с **Node.js-селектором** (cPanel → «Настройка Node.js приложений») | **Условно** (мало RAM, PGlite может терять данные) |
| **VPS / Cloud VPS** с root, Ubuntu, от **2 ГБ RAM** | **Да, рекомендуется** |
| Vercel / Railway / Render + Postgres | Да (альтернатива) |

**Минимум для продакшена:**
- Node.js **20** или **22** LTS  
- RAM **от 2 ГБ** (сборка и PGlite тяжёлые; лучше 4 ГБ)  
- Постоянный диск (не «эфемерный» контейнер без volume)  
- HTTPS (Let's Encrypt)

**База данных:**
- Без `DATABASE_URL` работает **PGlite** (встроенная Postgres в файле/памяти). На shared-хостинге при перезапуске/очистке **данные админки могут пропасть**.
- Для боевого сайта укажите **`DATABASE_URL`** на Postgres (Neon.tech бесплатно, или Postgres на VPS, или managed DB).

---

## Рекомендуемый путь: VPS на hoster.by

### 1. Заказать VPS
- Ubuntu 22.04 / 24.04  
- 2–4 ГБ RAM, SSD  
- Привязать домен к IP (A-запись)

### 2. Сервер: Node, Nginx, PM2

```bash
ssh root@ВАШ_IP

apt update && apt upgrade -y
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
apt install -y nodejs nginx git
npm i -g pm2

# пользователь без root (по желанию)
adduser --disabled-password deploy
mkdir -p /var/www/neopro
chown -R deploy:deploy /var/www/neopro
```

### 3. Залить код

С вашего компьютера:

```bash
# из папки workspace после npm run build
rsync -avz --exclude node_modules ./ deploy@ВАШ_IP:/var/www/neopro/
```

Или Git:

```bash
cd /var/www/neopro
git clone URL_РЕПО .
```

### 4. Переменные окружения

Создайте файл `/var/www/neopro/.env` (или задайте в PM2):

```env
NODE_ENV=production
PORT=3000
# Обязательно для сохранения данных между рестартами:
DATABASE_URL=postgresql://USER:PASS@HOST:5432/DBNAME?sslmode=require
# Опционально:
OPENROUTER_API_KEY=sk-or-v1-...
ADMIN_SESSION_TTL_HOURS=12
```

Пароль админки по умолчанию задаётся при первом запуске в коде (bootstrap) — **сразу смените** в админке → Настройки.

### 5. Сборка и запуск

```bash
cd /var/www/neopro
npm ci
npm run build
node scripts/copy-pglite-assets.mjs

# Запуск production-превью (Nitro/Vercel output через vite preview)
# Либо используйте entry из .vercel/output если настроите nitro node-server
PORT=3000 pm2 start npm --name neopro -- run preview -- --host 127.0.0.1 --port 3000
pm2 save
pm2 startup
```

> Сборка проекта ориентирована на preset **vercel**. На VPS удобно:  
> `npx vite preview --host 127.0.0.1 --port 3000`  
> после успешного `npm run build`. Nginx проксирует наружу 443 → 3000.

### 6. Nginx + SSL

`/etc/nginx/sites-available/neopro`:

```nginx
server {
  listen 80;
  server_name neoprosistem.by www.neoprosistem.by;
  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

```bash
ln -s /etc/nginx/sites-available/neopro /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
apt install -y certbot python3-certbot-nginx
certbot --nginx -d neoprosistem.by -d www.neoprosistem.by
```

---

## Вариант: shared hoster.by + Node.js selector

По [инструкции hoster.by](https://hoster.by/help/unix-khosting-nastroyka/razvyertyvanie-prilozheniya-na-node-js):

1. cPanel → **Программное обеспечение** → **Настройка Node.js приложений** → Создать.  
2. Node **20** или **22**, режим **production**.  
3. Корневой каталог приложения, URL, **файл запуска**.  
4. Залить файлы проекта (FTP/файловый менеджер).  
5. **Установить NPM пакеты** (если OOM — через SSH `npm ci`).  
6. Сборка: в SSH `npm run build && node scripts/copy-pglite-assets.mjs`.  
7. Точка входа: то, что слушает `process.env.PORT` (часто приложение само берёт PORT из окружения Node-селектора).

Ограничения shared: мало RAM на `npm run build`, PGlite может не переживать рестарты → **обязательно DATABASE_URL**.

---

## Чеклист после выкладки

1. Открыть главную — без белого экрана.  
2. `/admin` — вход, смена пароля.  
3. Стили → применить тему → F5 — тема на месте.  
4. Создать страницу, опубликовать, открыть `/p/slug`.  
5. Форма заявки → проверка в «Заявки».  
6. Бэкап: дамп Postgres или экспорт критичных kv/таблиц.

## Чего не делать

- Не заливать только `public/` как статику — **нужен Node-сервер**.  
- Не оставлять дефолтный пароль админки.  
- Не хранить секреты (`OPENROUTER_API_KEY`, `DATABASE_URL`) в публичном репозитории.
