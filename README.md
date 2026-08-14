REST API на **FastAPI**, который распознаёт жесты из видео с помощью нейросети **MoViNet**.  
Загружаешь видео чанками по сессии — на финальном чанке получаешь человекочитаемую метку жеста.

---

## 🚀 Что умеет

- 📤 Принимает видео **чанками** через `multipart/form-data`
- 🧠 Классифицирует жест по видео моделью **MoViNet A0**
- 💾 Хранит промежуточные чанки в оперативной памяти (in-memory сессии)
- 🐳 Упакован в Docker
- 🧪 Управляется через `uv` + `Makefile`

---

## 🛠 Стек технологий

| Технология | Назначение |
|------------|-----------|
| **Python 3.12** | Язык проекта |
| **FastAPI** | Веб-фреймворк, REST API |
| **Uvicorn** | ASGI-сервер |
| **PyTorch** | Инференс нейросети |
| **MoViNet** | Лёгкая 3D-CNN для классификации видео |
| **OpenCV** | Чтение и препроцессинг кадров |
| **uv** | Менеджер зависимостей и виртуального окружения |
| **Docker** | Контейнеризация |

---

## 📁 Структура проекта

gesture_recognition/
├── app/
│   ├── api/
│   │   ├── init.py          # сборка api_router
│   │   ├── api_di.py            # зависимости FastAPI (DI)
│   │   └── video_router.py      # эндпоинт загрузки видео
│   ├── core/
│   │   ├── init.py
│   │   └── app.py               # фабрика FastAPI-приложения
│   ├── ml/
│   │   ├── init.py
│   │   ├── pipeline.py          # обёртка над MoViNet: preprocess + predict
│   │   └── weights/
│   │       └── best_model_new.pt # веса обученной модели
│   ├── services/
│   │   ├── init.py
│   │   └── video_service.py     # бизнес-логика: сборка чанков + инференс
│   ├── storage/
│   │   ├── init.py
│   │   └── memory_storage.py    # синглтон SessionManager (in-memory)
│   └── run.py                   # точка входа: uvicorn
├── .python-version              # 3.12
├── .gitignore
├── Dockerfile
├── Makefile
├── pyproject.toml               # зависимости + метаданные
├── uv.lock
└── README.md

---

## 🧩 Классы жестов

Модель распознаёт **13 классов**:

| ID | Жест |
|----|------|
| 0 | Жарить |
| 1 | Лицо |
| 2 | Лопата |
| 3 | Пока |
| 4 | Привет! |
| 5 | С |
| 6 | большой |
| 7 | меня |
| 8 | положительный |
| 9 | пять |
| 10 | рот |
| 11 | свет |
| 12 | сердце |

---

## ⚡ Быстрый старт

### 1. Через `uv` (рекомендуется)

```bash
# установить зависимости
uv sync

# запустить сервер
uv run python app/run.py
Сервер поднимется на http://0.0.0.0:8080.

2. Через Docker
# собрать образ
make build

# запустить контейнер
make docker-run

3. Make-команды
make run      # локальный запуск через uvicorn
make build    # билд Docker-образа
make docker-run # запуск контейнера
make format   # black + isort
make lint     # flake8
make test     # pytest

📡 API
POST /api/upload-video
Загружает видео чанками. На финальном чанке возвращает распознанный жест.
Content-Type: multipart/form-data
Параметры:
Поле	Тип
file	File
sessionId	UUID
chunkIndex	int
isFinal	bool

Ответ на промежуточном чанке:
{
  "finalText": null
}

Ответ на финальном чанке:
{
  "finalText": "Привет!"
}

Пример запроса (curl)
curl -X POST "http://localhost:8080/api/upload-video" \
  -F "file=@chunk_0.mp4" \
  -F "sessionId=123e4567-e89b-12d3-a456-426614174000" \
  -F "chunkIndex=0" \
  -F "isFinal=false"

🧠 Как работает инференс
1. Видео собирается из чанков в BytesIO (SessionManager).
2. На финальном чанке VideoService отдаёт видео в MoViNetPipeline.
3. Пайплайн:
- сохраняет видео во временный файл,
- извлекает кадры через cv2.VideoCapture,
- ресайзит каждый кадр до 320x320,
- конвертирует BGR → RGB,
- собирает тензор формата [B, C, T, H, W],
- прогоняет через MoViNet A0,
- возвращает метку с максимальной вероятностью.

🔧 Зависимости
Основные зависимости указаны в pyproject.toml:
[project]
dependencies = [
    "fastapi>=0.117.1",
    "movinet-pytorch",
    "opencv-python>=4.12.0.88",
    "torch>=2.8.0",
    "uvicorn>=0.37.0",
    "python-multipart>=0.0.6"
]

movinet-pytorch ставится напрямую из GitHub:
[tool.uv.sources]
movinet-pytorch = { git = "https://github.com/Atze00/MoViNet-pytorch.git" }


📝 TODO
- Привести entrypoint к единому виду: Makefile и Dockerfile сейчас ссылаются на app.main:app, а в проекте точка входа — app/run.py.
- Добавить MoViNetPipeline в DI-контейнер, чтобы VideoService получал и SessionManager, и model_pipeline.
- Добавить тесты (pytest).
- Добавить валидацию формата видео и обработку ошибок.
- Вынести конфигурацию (путь к весам, device, размер кадра) в .env.
