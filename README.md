# Civitai Downloader для Stable Diffusion WebUI

![Python](https://img.shields.io/badge/python-3.10%2B-blue)
![Platform](https://img.shields.io/badge/platform-AUTOMATIC1111-green)
![Tests](https://github.com/kushalka/sd-webui-civitai-downloader/actions/workflows/tests.yml/badge.svg)
![API](https://img.shields.io/badge/REST%20API-%2Fcivitai%2F*-orange)

Расширение для [AUTOMATIC1111 Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui), которое скачивает модели (LoRA и другие) с [Civitai](https://civitai.com) по обычной ссылке — и позволяет управлять этим удалённо через REST API. Идеально подходит для связки «Telegram-бот дома + SD WebUI на сервере/вторым ПК».

## Возможности

- 📥 **Скачивание по ссылке** — вставляете `https://civitai.com/models/123456` или ссылку с `?modelVersionId=`, файл кладётся в папку LoRA
- 🖼 **Превью автоматически** — вместе с моделью скачивается картинка-превью (`модель.preview.png`)
- 🔁 **Повторы при обрыве** — 3 попытки с экспоненциальной задержкой, проверка размера файла, битые файлы удаляются
- 🧠 **Умный выбор файла** — из версии берётся именно модель (`Model`), а не training-архивы; поддерживаются `.safetensors`, `.pt`, `.ckpt`
- 📡 **REST API** — скачать модель, загрузить свою, докачать превью ко всем LoRA
- 🔑 **Работа с API-ключом Civitai** — для early access и закрытых моделей; ключ можно задать в запросе, файлом `default_api_key.txt` или запомнить
- 🤖 **Готовый клиент для Telegram-бота** — `telegram_bot_example.py` с поддержкой нескольких серверов

## Установка

1. Откройте вкладку **Extensions → Install from URL** в SD WebUI
2. Вставьте `https://github.com/kushalka/sd-webui-civitai-downloader` и нажмите **Install**
3. Перезапустите WebUI (или **Apply and restart UI**)

Либо вручную:

```bash
git clone https://github.com/kushalka/sd-webui-civitai-downloader extensions/sd-webui-civitai-downloader
```

Зависимость одна — `requests` (уже есть в окружении WebUI).

## Использование

### Веб-интерфейс не нужен — всё через API

После запуска WebUI расширение регистрирует эндпоинты на том же порту, что и WebUI (по умолчанию `7860`).

| Метод | Эндпоинт | Описание |
|---|---|---|
| `GET` | `/civitai/status` | Проверка доступности |
| `POST` | `/civitai/download` | Скачать модель с Civitai в папку LoRA |
| `POST` | `/civitai/upload` | Загрузить свой `.safetensors` файл |
| `POST` | `/civitai/fetch-previews` | Докачать превью для всех LoRA без превью |

### Скачать модель

```bash
curl -X POST http://127.0.0.1:7860/civitai/download \
  -H "Content-Type: application/json" \
  -d '{"url": "https://civitai.com/models/123456?modelVersionId=789", "api_key": "необязательно"}'
```

Ответ:

```json
{
  "success": true,
  "message": "Model downloaded successfully",
  "filename": "MyLora_v2.safetensors",
  "path": "models/Lora/MyLora_v2.safetensors",
  "model_name": "My LoRA",
  "version_name": "v2.0",
  "preview_path": "models/Lora/MyLora_v2.preview.png"
}
```

> ⚠️ Запрос держит соединение, пока идёт скачивание (может занять несколько минут) — ставьте большой таймаут у HTTP-клиента.

### Загрузить свой файл

```bash
curl -X POST http://127.0.0.1:7860/civitai/upload \
  -F "file=@my_lora.safetensors" \
  -F "filename=custom_name.safetensors"
```

### Докачать превью ко всем LoRA

Расширение ищет модели по SHA256-хэшу файла и скачивает первую картинку с Civitai:

```bash
curl -X POST http://127.0.0.1:7860/civitai/fetch-previews \
  -H "Content-Type: application/json" -d '{}'
```

## API-ключ Civitai

Нужен для закрытых и early access моделей. Получить: https://civitai.com/user/account

Три способа (в порядке приоритета):

1. Передать в запросе: `{"url": "...", "api_key": "ваш_ключ"}`
2. Положить в файл `default_api_key.txt` в корне расширения (не коммитится)
3. Не передавать — расширение запомнит ключ из последнего запроса в `civitai_api_key.json` (не коммитится)

## Пример: Telegram-бот

В репозитории есть готовый клиент [`telegram_bot_example.py`](telegram_bot_example.py) — класс `CivitaiDownloaderClient` с поддержкой нескольких серверов:

```python
from telegram_bot_example import CivitaiDownloaderClient

client = CivitaiDownloaderClient(servers=[
    {"name": "Home PC", "url": "http://192.168.1.100:7860"},
    {"name": "Server",  "url": "http://192.168.1.101:7860"},
])

# живой сервер?
online = [s for s in client.servers if client.check_server_status(s["url"])]

# скачать модель
result = client.download_model(
    server_url="http://192.168.1.100:7860",
    civitai_url="https://civitai.com/models/123456",
)
print(result["message"])
```

## Структура проекта

```
├── scripts/
│   ├── civitai_downloader.py   # ядро: скачивание, превью, повторы
│   └── civitai_api.py          # REST API-эндпоинты
├── tests/
│   └── test_file_selection.py  # тесты выбора файла модели
├── telegram_bot_example.py     # пример интеграции с Telegram-ботом
├── requirements.txt
└── default_api_key.txt.example # шаблон для дефолтного API-ключа
```

## Разработка

```bash
pip install -r requirements.txt pytest
pytest tests/
```

## Частые ошибки

| Сообщение | Причина и решение |
|---|---|
| `Доступ запрещён. Эта модель требует API ключ` | Модель закрытая — укажите `api_key` или создайте `default_api_key.txt` |
| `Неверный API ключ или ключ не предоставлен` | Ключ истёк/опечатка — получите новый на civitai.com/user/account |
| `Превышен лимит запросов` | Rate limit Civitai — подождите минуту |
| `Версия не содержит поддерживаемого файла модели` | В версии нет `.safetensors`/`.pt`/`.ckpt` |
