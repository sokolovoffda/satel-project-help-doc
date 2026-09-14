# Перенос MCP на другой компьютер

Чеклист: что у нас подключено, как агент с этим работает, что скопировать и как поднять заново.

Актуально на **2026-09-14** (машина: `d.sokolov`, Cursor user-level config).

---

## Как это устроено (мы + Cursor)

1. **Cursor читает** `~/.cursor/mcp.json` и поднимает MCP-серверы как отдельные Node-процессы.
2. В чате агент видит их как namespaces: `user-jira-server`, `user-gitlab-server`, `user-testrail-server`, `user-figma-bridge`.
3. Перед вызовом агент смотрит схему инструмента (`GetDynamicTools`), потом вызывает tool.
4. **Credentials не лежат в `mcp.json`.** Они в `~/.satel/config.json` (Jira / GitLab / TestRail / Confluence и т.д.).
5. **Skills** (`~/.cursor/skills/`) говорят *когда* звать MCP: «начинаем» → Jira, «что делал сегодня» → GitLab+Jira, макет → Figma Bridge, и т.д.
6. **Rules** (`~/.cursor/rules/` + project `.cursor/rules/`) задают политику: например Figma только через `figma-bridge`, не через Framelink REST.

Итого цепочка: **skill/rule → MCP tool → API (Jira/GitLab/…) → ответ в чат**.

---

## Что сейчас подключено глобально

Файл: `%USERPROFILE%\.cursor\mcp.json`

| Имя в mcp.json | Namespace в чате | Откуда код | Для чего |
|----------------|------------------|------------|----------|
| `jira-server` | `user-jira-server` | `cursor-ai/mcp-servers/jira-server` | Jira + Confluence + WakaTime helpers |
| `gitlab-server` | `user-gitlab-server` | `cursor-ai/mcp-servers/gitlab-server` | Активности, MR, pipelines |
| `testrail-server` | `user-testrail-server` | `cursor-ai/mcp-servers/testrail-server` | TestRail планы/результаты |
| `figma-bridge` | `user-figma-bridge` | `cursor-ai/scripts/figma-bridge-mcp-wrapper.js` + плагин | Локальный bridge к Figma Desktop |

Пример текущего `mcp.json` (пути поправишь под новый user/путь к репо):

```json
{
  "mcpServers": {
    "gitlab-server": {
      "command": "node",
      "args": [
        "C:/Users/<YOU>/Desktop/projects/cursor-ai/mcp-servers/gitlab-server/dist/index.js"
      ]
    },
    "jira-server": {
      "command": "node",
      "args": [
        "C:/Users/<YOU>/Desktop/projects/cursor-ai/mcp-servers/jira-server/dist/index.js"
      ]
    },
    "testrail-server": {
      "command": "node",
      "args": [
        "C:/Users/<YOU>/Desktop/projects/cursor-ai/mcp-servers/testrail-server/dist/index.js"
      ]
    },
    "figma-bridge": {
      "command": "node",
      "args": [
        "C:/Users/<YOU>/Desktop/projects/cursor-ai/scripts/figma-bridge-mcp-wrapper.js"
      ]
    }
  }
}
```

Лучше не копировать руками — прогнать `npm run setup-mcp` и `npm run setup-mcp-figma` из `cursor-ai` (пути пропишутся сами).

---

## Что НЕ в глобальном mcp.json (но полезно знать)

| MCP | Где | Заметка |
|-----|-----|---------|
| **WUI Common Library** (`user-wui-library`) | `common-library/mcp-server` | В README cursor-ai описан; на этой машине в глобальном `mcp.json` сейчас **не** зарегистрирован. При необходимости — отдельный setup из common-library. |
| **Swagger** | `dealing-console-ui/.ai/mcp/swagger` | Для Codex (`%USERPROFILE%\.codex\config.toml`), токен в env `SWAGGER_TOKEN`. Не Cursor MCP. |
| **browser** (`@browsermcp/mcp`) | project `rtu-user-web-app/.cursor/mcp.json` | Только для того проекта. |
| **Framelink MCP for Figma** | устарел | Не переносить. Используем только `figma-bridge`. |

---

## Что скопировать / перенести

### Обязательно

| Что | Путь (сейчас) | Зачем |
|-----|---------------|-------|
| Репозиторий **cursor-ai** | `Desktop/projects/cursor-ai` | Код MCP + scripts + skills |
| Credentials Satel | `%USERPROFILE%\.satel\config.json` | Логины/токены Jira, GitLab, TestRail… |
| (опц.) Project TestRail id | `cursor-ai/.satel/config.json` или project `.satel/config.json` | `testRailProjectId` |

### Желательно (чтобы skills/rules сразу работали)

| Что | Путь |
|-----|------|
| Skills | `%USERPROFILE%\.cursor\skills\` |
| User rules | `%USERPROFILE%\.cursor\rules\` |
| Figma Bridge plugin clone | `%USERPROFILE%\.cursor\figma-mcp-bridge\` |

Или снова: `cd cursor-ai && npm run install` — поставит skills/rules и соберёт MCP.

### Секреты — осторожно

- Не коммить `~/.satel/config.json` в git.
- Переноси флешкой / encrypted copy / вручную пересоздай токены на новом ПК.
- Figma Bridge **не** требует Figma REST token (в отличие от старого Framelink).

---

## Установка на новом компьютере (пошагово)

### 0. Предусловия

- Node.js **≥ 18**
- Cursor установлен
- Доступ к `gitlab.satel.org` (клонировать cursor-ai)
- Figma **Desktop** (если нужен макетный MCP)

### 1. Клон и install

```bash
git clone git@gitlab.satel.org:sergeymitrichev/cursor-ai.git
cd cursor-ai
npm run install
```

Это:

- ставит skills → `~/.cursor/skills/`
- ставит rules → `~/.cursor/rules/`
- собирает MCP в `mcp-servers/*/dist`
- пишет записи в `~/.cursor/mcp.json`

### 2. Figma Bridge отдельно

```bash
cd cursor-ai
npm run setup-mcp-figma
```

Потом в Figma Desktop **один раз**:

1. Import plugin:  
   `%USERPROFILE%\.cursor\figma-mcp-bridge\repo\plugin\manifest.json`
2. Файл открыт → `Plugins → Development → Figma MCP Bridge`
3. Статус: **WebSocket Connected**

### 3. Credentials

Скопируй `~/.satel/config.json` со старого ПК **или** при первом «начинаем» / «что делал сегодня» дай агенту URL/логин/токены — он сохранит через `*_save_config`.

Ожидаемые ключи (примерно, без значений):

- Jira: `jiraUrl`, `jiraUsername`, `jiraPassword`, `jiraProjectKeys`
- Confluence on-prem: `confluenceUrl`
- GitLab: URL + token (`read_api`, `read_user`)
- TestRail: `testRailApiKey` (+ опц. `testRailUrl`, `testRailLogin`)
- Project: `testRailProjectId` в `.satel/config.json` репо

### 4. Перезапуск Cursor

`Settings → MCP` — все четыре сервера зелёные / enabled.

Проверка в чате:

- «проверь jira mcp» → `jira_check_config`
- «проверь gitlab» → `gitlab_check_config`
- Figma: выделить фрейм → «что выделено» → `get_selection` / `list_files`

---

## Типичные сценарии работы с MCP

| Фраза / ситуация | MCP | Skill |
|------------------|-----|-------|
| «начинаем», «start», WUI-XXXX | jira (+ testrail) | jira-dev-assistant |
| «готово», комментарий в Jira, worklog | jira | jira-dev-assistant |
| «что делал сегодня» | gitlab + jira | what-i-did-today |
| «опубликовать в Confluence», release notes | jira (confluence_*) | confluence-publish |
| макет / pixel-perfect / Figma link | figma-bridge | figma-pixel-perfect |
| ревью MR / pipelines | gitlab | review-like-a-king / what-i-did-today |
| UI Kit компоненты | (если подключён) wui-library | wui-common-library |

---

## Troubleshooting

| Симптом | Что проверить |
|---------|----------------|
| MCP красный в Settings | Путь в `mcp.json` существует? `dist/index.js` собран? `node -v` ≥ 18? |
| Tools есть, API падает | `~/.satel/config.json`, VPN/доступ к satel |
| Figma пустой `list_files` | Плагин запущен в **Desktop**, WebSocket Connected; не браузер |
| Два Figma MCP | Удали Framelink; оставь только `figma-bridge` |
| Skills «не срабатывают» | Есть ли они в `~/.cursor/skills/`? Перезапусти Cursor после `npm run install` |

---

## Минимальный чеклист переноса

- [ ] Склонировать `cursor-ai`
- [ ] `npm run install`
- [ ] `npm run setup-mcp-figma`
- [ ] Скопировать `~/.satel/config.json` (или пересоздать токены)
- [ ] Импортировать Figma MCP Bridge plugin
- [ ] Перезапустить Cursor → Settings → MCP всё enabled
- [ ] Смоук: Jira issue / GitLab activity / Figma selection

---

## Связанные файлы

- `~/.cursor/mcp.json` — регистрация серверов
- `~/.satel/config.json` — секреты
- `cursor-ai/scripts/setup-mcp.js`
- `cursor-ai/scripts/setup-mcp-figma.js`
- `cursor-ai/README.md` — полный overview skills + MCP
- Project rule Figma: `dealing-console-ui/.cursor/rules/figma-mcp.mdc`
