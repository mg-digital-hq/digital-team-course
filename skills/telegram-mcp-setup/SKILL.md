---
name: telegram-mcp-setup
description: "Мастер подключения Telegram к Claude Code через MCP. Проводит по всем шагам: получение API-ключей, генерация session string, безопасное хранение, регистрация MCP-сервера. Домашка встречи 3 курса Цифровая команда компании."
license: Apache-2.0
compatibility: |
  Windows (основной путь, PowerShell), macOS/Linux (Terminal).
  Нужен Python 3.10+. Docker не требуется.
metadata:
  category: setup
  underlying-package: "telegram-mcp (pip), репозиторий github.com/chigwell/telegram-mcp, лицензия Apache-2.0"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - AskUserQuestion
---

# Мастер подключения Telegram

Подключает Telegram-аккаунт к Claude Code через MCP - после этого ассистент сможет читать сообщения, искать по истории переписки и готовить черновики ответов.

## Ключевые понятия

- **MCP (Model Context Protocol)** - способ, которым Claude Code подключается к внешним сервисам. По сути "разъём" для плагинов.
- **Session string** - код авторизации, который даёт доступу к Telegram-аккаунту без ввода кода каждый раз. Это чувствительнее пароля (см. раздел "Риски" ниже).
- **API ID и API Hash** - бесплатные учётные данные, которые Telegram выдаёт для регистрации "приложения". Получаются за 2 минуты на my.telegram.org.

## Как устроены данные

Будь честен с человеком на этот счёт:

1. **telegram-mcp работает локально** на компьютере человека и забирает сообщения с серверов Telegram.
2. Когда Claude обрабатывает эти сообщения (суммирует, ищет, готовит черновик ответа) - **содержимое сообщений уходит в Anthropic API** как часть разговора.
3. **Сам session string остаётся локально** - никуда не отправляется.

Итог: учётные данные - только локально. Содержимое сообщений - уходит в Anthropic, когда просишь Claude его обработать. Это то же самое, что вручную скопировать сообщение в чат с Claude.

## Предварительная проверка

Запусти сначала (PowerShell):
```powershell
Write-Host "Python: $(try { python --version } catch { 'НЕ НАЙДЕН' })"
Write-Host "pip: $(try { pip --version } catch { 'НЕ НАЙДЕН' })"
Write-Host "Claude CLI: $(try { claude --version } catch { 'НЕ НАЙДЕН' })"
```

Если Python не найден - поставь с python.org (галочка "Add python.exe to PATH" при установке обязательна) или `winget install Python.Python.3.12`.

## Шаг 1. Получи API ID и API Hash (~5 минут)

1. Открой https://my.telegram.org в браузере.
2. Введи номер телефона (с кодом страны, например +7...), затем код подтверждения из Telegram.
3. Нажми "API development tools".
4. Если приложения ещё нет - создай:
   - App title: "Claude Code Integration" (или любое своё)
   - Short name: должно быть уникальным глобально, 5-32 символа, только буквы/цифры - если "claudecode" занято, добавь цифры или инициалы
   - Platform: Desktop
5. Сохрани **App api_id** (число) и **App api_hash** (строка из 32 символов).

**Если видишь CAPTCHA** - это нормально, просто пройди её вручную.

## Шаг 2. Подключи MCP-сервер (~3 минуты)

Установи пакет:
```powershell
pip install telegram-mcp
```

Это ставит пакет `telegram-mcp` (открытый исходный код, github.com/chigwell/telegram-mcp, лицензия Apache-2.0) - не Docker, не облачный сервис, работает полностью на компьютере человека.

Зарегистрируй MCP-сервер в Claude Code (замени значения на реальные из шага 1; SESSION_STRING подставь после шага 3, пока можно оставить заглушку и обновить позже командой `claude mcp remove` + `claude mcp add`):

```powershell
claude mcp add telegram-mcp -s user -e TELEGRAM_API_ID="ТВОЙ_API_ID" -e TELEGRAM_API_HASH="ТВОЙ_API_HASH" -e TELEGRAM_SESSION_STRING="ТВОЙ_SESSION_STRING" -- telegram-mcp
```

**Важно:** команду `claude mcp add` нужно выполнять в отдельном окне PowerShell, не внутри текущей сессии Claude Code - иначе она зависнет.

## Шаг 3. Получи session string (~3-5 минут)

Это отдельный шаг от шага 1 - не путай их. Здесь ты входишь в сам аккаунт (номер телефона + код), а не регистрируешь приложение.

Открой **отдельное** окно PowerShell (это интерактивный шаг, здесь нужно будет вводить ответы) и выполни:

```powershell
python -m venv $env:TEMP\telegram-session-venv
& "$env:TEMP\telegram-session-venv\Scripts\pip.exe" install telethon
& "$env:TEMP\telegram-session-venv\Scripts\python.exe" -c "
from telethon.sync import TelegramClient
from telethon.sessions import StringSession
api_id = int(input('Введи API ID: '))
api_hash = input('Введи API Hash: ')
with TelegramClient(StringSession(), api_id, api_hash) as client:
    print()
    print('=== ТВОЙ SESSION STRING (скопируй всё, что ниже) ===')
    print(client.session.save())
    print('=== КОНЕЦ SESSION STRING ===')
"
```

Введи API ID и API Hash из шага 1, затем номер телефона (+7...), код подтверждения из Telegram и, если включена двухфакторная защита, пароль от неё.

**Проверка:** результат - длинная строка из ~300-400 символов. Если короче 100 символов - что-то пошло не так, повтори.

**Если код подтверждения не приходит** больше 2 минут - не запрашивай новый повторно (это включает лимит запросов у Telegram), см. раздел FAQ ниже про VPN и прокси.

Обнови MCP-сервер реальным session string:
```powershell
claude mcp remove telegram-mcp
claude mcp add telegram-mcp -s user -e TELEGRAM_API_ID="ТВОЙ_API_ID" -e TELEGRAM_API_HASH="ТВОЙ_API_HASH" -e TELEGRAM_SESSION_STRING="ТВОЙ_SESSION_STRING" -- telegram-mcp
```

## Шаг 4. Проверь подключение (~1 минута)

1. Перезапусти Claude Code (закрой и открой заново - иначе новый MCP-сервер не подхватится).
2. Спроси: "Покажи мои чаты в Telegram" или "Прочитай последние 3 сообщения".

Если список чатов/сообщений появился - подключение готово.

## Риски session string

Будь честен с человеком:

- Пароль требует двухфакторного кода. Session string **обходит двухфакторную защиту** - это уже готовый токен после входа.
- Действует **бессрочно**, пока сессию явно не завершат в настройках Telegram.
- **Нет уведомления**, если кто-то использует украденный session string.

Если session string украден - злоумышленник может читать всю переписку, писать сообщения от имени человека, скачивать файлы и фото, видеть контакты - и всё это незаметно.

## Полный отзыв доступа

Если нужно отключить Telegram или есть подозрение на утечку:

1. Заверши сессию в Telegram: Настройки → Конфиденциальность → Активные сессии → найти нужную (или "Завершить все другие сессии").
2. Удали переменные окружения (если использовались):
   ```powershell
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_API_ID', $null, 'User')
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_API_HASH', $null, 'User')
   [System.Environment]::SetEnvironmentVariable('TELEGRAM_SESSION_STRING', $null, 'User')
   ```
3. Убери MCP-сервер: `claude mcp remove telegram-mcp`
4. При необходимости отзови API-ключи на https://my.telegram.org/apps.

## FAQ / частые проблемы

**Код подтверждения не приходит, или my.telegram.org не открывается / выдаёт ошибку регистрации приложения**

Обычно дело в VPN или блокировке по IP. Что попробовать по порядку:
1. Отключи VPN и попробуй снова - иногда VPN сам мешает, а не помогает.
2. Попробуй проверенный прокси, например [astroproxy.com](https://astroproxy.com/) - попроси Claude помочь его настроить. VPN, даже платный, не всегда полностью скрывает IP так, как нужно Telegram.
3. Если ничего не помогло и номер российский - попроси знакомого за границей сгенерировать API-ключи на своём аккаунте, либо используй зарубежный номер телефона для регистрации.

**"Short name уже занято" при создании приложения**

Имя должно быть уникальным глобально. Добавь цифры или инициалы, например "ivan_claude_2026".

**MCP-сервер не появляется после регистрации**
```powershell
claude mcp remove telegram-mcp
claude mcp add telegram-mcp -s user -e TELEGRAM_API_ID="..." -e TELEGRAM_API_HASH="..." -e TELEGRAM_SESSION_STRING="..." -- telegram-mcp
claude mcp list
```
Затем перезапусти Claude Code.

**Session string перестал работать (AUTH_KEY_UNREGISTERED)**

Сессия истекла или была завершена в настройках Telegram. Повтори шаг 3 полностью и обнови MCP-сервер новым значением.

## Правила безопасности

1. Никогда не пересылай и не показывай session string никому - это как пароль, только без второго фактора.
2. Всегда используй режим черновика для сообщений - пусть ассистент готовит, а не отправляет сам.
3. Периодически проверяй активные сессии: Telegram → Настройки → Конфиденциальность → Активные сессии.
4. Если есть чувствительная деловая переписка - для первого теста лучше использовать второй, менее важный Telegram-аккаунт.
