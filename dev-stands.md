# Dev / Stage / QA стенды — устройство и работа

Артефакт к задаче WUI-4143 (и общее для работы с пультами на лабораторных хостах).  
Дата фиксации: **2026-09-25**. Конфиги на стендах могли измениться после — перед правками сверять `docker ps` / `nginx -T`.

---

## Хосты

| Имя | IP | DNS / UI | Роль |
|-----|-----|----------|------|
| **WebClientRtuDev** | `192.168.232.234` | `webclientrtudev.satel.org` | Dev RTU + docker-пульт |
| **WebClientRtuStage** | `192.168.232.235` | `webclientrtustage.satel.org` | Stage RTU + docker-пульт |
| **astra-18093** | `192.168.136.91` | (тестерская) | Host/deb установка (как у QA), **без** docker пульта |

SSH: учётка `user` (sudo) на Dev/Stage; на astra часто `root`.

---

## Карта портов (типичная)

| Порт | Назначение |
|------|------------|
| **3333** | UI пульта (HTTPS), nginx в `dealing-console` (Dev/Stage) или host nginx (astra) |
| **5059** | sipproxy / mvts (`mvts3g-server`), обычно localhost + LAN |
| **6001** | RTU WebAPI (MOA): `/api/user/*`, login, contacts |
| **8145** | IM |
| **8441 / 8442** | prompt / files |
| **8444** | Centrex **admin UI** (`/var/www/web_admin`) — Dev |
| **8448** | API admin Centrex (не UI) |
| **8453 / 8455** | классический APS (nginx → dotnet `:8454`) |
| **9900** | rtu-admin-moa (docker) |
| **9904** | rtu-moa-aps (docker MOA UI), **не** prefs пульта `/api/v1` |
| **9996** | dealing-admin пульта — **только Dev** (`dealing-admin-app` → 3000) |
| **9999** | `rtu-web-client` (веб-клиент) на Dev и Stage; **не** dealing-admin |

---

## Dev `.234` (WebClientRtuDev)

### Пульт

- Контейнер: **`dealing-console`** (nginx + статика `/usr/share/nginx/html`).
- URL: `https://webclientrtudev.satel.org:3333/`
- Штатный nginx (как в репо): `/api/` → `127.0.0.1:6001`, `/sip` → `127.0.0.1:5059`.

### Turret prefs API (`/api/v1/me/*`)

Это **не** RTU `:6001` и **не** классический APS `:8455`.

- Backend: контейнер **`dealing-admin-app`** (`0.0.0.0:9996→3000`), БД `dealing-admin-db` (`:9995`).
- Auth: admin валидирует JWT через MOA `who-am-i` (URL задаётся в **setup админки**).
- 25.09: в nginx `dealing-console` добавлены location’ы **перед** `location /api/`:

```nginx
location /api/setup/  { proxy_pass http://172.17.0.1:9996; ... }
location /api/admin/  { proxy_pass http://172.17.0.1:9996; ... }
location /api/secure/ { proxy_pass http://172.17.0.1:9996; ... }
location /api/v1/     { proxy_pass http://172.17.0.1:9996; ... }
```

Бэкап: `/etc/nginx/nginx.conf.bak-api-v1-*` внутри контейнера.

- Если после логина RTU снова 401 на `/api/v1` → в админке (`:9996`) проверить **MOA base URL** (должен быть Dev, не Stage).

### Полезные команды

```bash
sudo docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Image}}"
sudo docker exec dealing-console grep -n "location /api\|sip\|6001\|9996" /etc/nginx/nginx.conf
sudo docker logs --tail 50 dealing-admin-app
curl -sk -o /dev/null -w "%{http_code}\n" https://127.0.0.1:3333/api/v1/me/main-settings
```

Правки nginx в контейнере **не персистятся** гарантированно после recreate — после редеплоя контейнера проверять conf заново.

---

## Stage `.235` (WebClientRtuStage)

### Пульт

- Контейнер: **`dealing-console`** (аналогично Dev).
- URL: `https://webclientrtustage.satel.org:3333/`
- Есть также **`dealing-console-ui`** (часто Restarting — конфликт порта / старый контейнер).

### Admin / prefs

- **`dealing-admin-app` на Stage нет** — так задумано (см. `Dev-машины и CI-CD.md`: dealing-admin только Dev `:9996`).
- Порт **`:9999` на Stage** = `rtu-web-client` (веб), не админка пульта.
- Без своего admin пульт на Stage отдаёт **404** на `/api/v1/me/*` (весь `/api/` уходит в `:6001`).
- 25.09 ошибочно пробовали проксировать `/api/v1/` на локальный `:9999` — **откатано** (`nginx.conf.bak-api-v1`).
- Классический APS host: `:8455` → `:8454` (dotnet); на Stage на момент съёма **`:8454` мог быть не поднят**.

Проброс Stage → Dev `:9996` и зависимость от MOA URL — см. секцию **«Auth dealing-admin / MOA URL»** ниже.

### Hop-эксперимент (WUI-4143)

Временно `/sip` и `/api` уводили на `.234`; затем откат:

```bash
sudo docker exec dealing-console sh -c \
  'cp /etc/nginx/nginx.conf.bak-065959 /etc/nginx/nginx.conf && nginx -t && nginx -s reload'
```

---

## QA astra-18093 (`192.168.136.91`)

### Отличие от Dev/Stage

- **Нет** docker `dealing-console` для пульта.
- Статика: `/var/www/rtu-turret-console-dealing` (deb / ручной деплой `dist`).
- Единый nginx site: `/etc/nginx/sites-enabled/rtu-turret-console-dealing.conf`
  - listen `:3333` (из `/etc/rtu-turret-admin/nginx/listen.conf`)
  - `/api/v1|setup|admin|secure`, `/admin`, `/_nuxt`, `/ws/` → `turret_aps_node` (`127.0.0.1:3000`)
  - остальной `/api/` → `rtu_webapi` (`127.0.0.1:6001`)
- Сервис: `rtu-turret-admin.service` → Node `/usr/lib/rtu-turret-admin/.output/server/index.mjs`
- Env: `/etc/rtu-turret-admin/env`

### Классический APS

- Site: `aps-server-proxy.conf` — `:8455` ssl / `:8453` → `127.0.0.1:8454` (dotnet)
- Conf: `/etc/rtu-cl-aps/aps.conf`

### Деплой UI develop (как делали 25.09)

1. Локально: `npm run build` (не `vite build develop`).
2. На astra: бэкап каталога → копирование `dist/*` в `/var/www/rtu-turret-console-dealing`.
3. Reload nginx при необходимости (статика часто подхватывается без reload).

Бэкап сессии: `/var/www/rtu-turret-console-dealing.bak-20260925-1222`.

---

## Centrex admin (Dev)

- UI: `https://webclientrtudev.satel.org:8444/`
- Колонка «Адрес регистрации» — смотреть здесь после REGISTER.
- `:8448` — API admin, не полноценный UI.

---

## Схема потоков пульта (Dev docker)

```text
Браузер
  │
  ├─ HTTPS :3333  ──статика──► dealing-console nginx
  │
  ├─ /api/user/*  ─────────────► :6001 RTU WebAPI
  ├─ /api/v1/me/* ──(патч)─────► :9996 dealing-admin-app ──who-am-i──► MOA URL из setup
  ├─ /sip         ─────────────► :5059 sipproxy (localhost)
  ├─ /im          ─────────────► :8145
  └─ /prompt/     ─────────────► :8441
```

На **astra** `/api/v1/*` и `/api/user/*` разведены в **одном** host nginx (эталон для сравнения с docker).

---

## Auth dealing-admin / MOA URL

### Как связан логин пульта и админка `:9996`

```text
логин пульта → JWT выдаёт RTU стенда (/api/user/login → :6001)
     ↓
запросы /api/v1/me/* → dealing-admin (:9996)
     ↓
admin валидирует токен: GET {MOA_URL}/api/user/who-am-i  (URL из setup админки)
     ↓
JWT и MOA_URL от одного RTU → 200 / prefs ок
JWT от другого RTU, чем MOA_URL → 401 → axios interceptor → logout
```

Та же логика, что в **локальном dev**: сменил env / target стенда (куда бьёт UI и login) → нужно зайти в dealing-admin и **поменять сервер авторизации (MOA)** на тот же стенд. Иначе RTU-логин проходит, а prefs/`/api/v1` валят сессию.

Пример 25.09 на Dev: в setup был `https://webclientrtustage.satel.org:9999` → Stage JWT path; после смены на Dev — вход с prefs заработал.

Админка: `http://webclientrtudev.satel.org:9996/` (или https, как открыто на стенде).

### Одна админка на два стенда?

`dealing-admin` живёт **только на `.234:9996`**. На `.235` своей нет.

Технически можно на Stage в `dealing-console` пробросить:

```nginx
location /api/v1/ { proxy_pass http://192.168.232.234:9996; ... }
# + /api/setup/, /api/admin/, /api/secure/ при необходимости
```

Но «залогинится ли» для prefs зависит от MOA в **этой** (единственной) админке:

| Пульт | JWT от | MOA URL в admin `:9996` | Prefs |
|-------|--------|-------------------------|-------|
| Dev `:3333` | Dev | Dev | ок |
| Stage `:3333` | Stage | Stage | ок, но данные prefs в **Dev БД** |
| Stage | Stage | Dev | 401 |
| Dev | Dev | Stage | 401 |

Один setup = один MOA URL: переключил на Stage — отвалится Dev, и наоборот. Плюс общая БД prefs на Dev.

**Вывод:** проброс Stage→`:9996` — временный костыль. Нормально — своя `dealing-admin` на Stage (или обновление от QA), а не общий Dev admin. Для повседневной работы на Dev: сменил target RTU ↔ сразу синхронизируй MOA в `:9996`.

---

## Эмуляция hop заказчика (кратко)

Цель: UI на хосте A, sipproxy на хосте B.

1. На A в `dealing-console`: `/sip` → `https://B:3333/sip` (+ `proxy_ssl_verify off`, Host B), не напрямую `B:5059`.
2. На B: `set_real_ip_from <IP A>; real_ip_header X-Real-IP;`
3. Логин/API при необходимости тоже на B (`/api` → B:6001), иначе сессия на другом RTU.
4. Смотреть адрес в админке B (`:8444` на Dev).
5. **Всегда откатывать** conf из `.bak-*` после эксперимента.

Подробности и результаты — в `case.md` (секция «Лабораторная проверка hop»).

---

## Бэкапы nginx, созданные 25.09

| Хост | Контейнер | Файл | Содержание |
|------|-----------|------|------------|
| `.235` | dealing-console | `nginx.conf.bak-065959` | До hop `/sip`+`/api` |
| `.234` | dealing-console | `nginx.conf.bak-xreal-075207` | До `real_ip` |
| `.234` | dealing-console | `nginx.conf.bak-api-v1-*` | До location `/api/v1` |
| `.235` | dealing-console | `nginx.conf.bak-api-v1` | Попытка `/api/v1`→9999 (откат выполнен) |

Восстановление:

```bash
sudo docker exec dealing-console sh -c \
  'cp /etc/nginx/nginx.conf.bak-XXXX /etc/nginx/nginx.conf && nginx -t && nginx -s reload'
```

---

## Чеклист перед правками на стенде

1. `docker ps` — имена контейнеров и порты.
2. Бэкап conf **внутри** контейнера с меткой времени.
3. `nginx -t` до reload.
4. После эксперимента — откат или явная фиксация «оставляем».
5. Для prefs: не путать `:8455` (классический APS), `:9904` (moa-aps docker), `:9996` (dealing-admin, только Dev), `:9999` (rtu-web-client).
6. Сменил стенд/env логина → проверь MOA URL в dealing-admin (`:9996`).
