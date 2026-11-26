# Мобильное приложение для учащихся автошколы

Информационная система записи на занятия по вождению и учёта успеваемости в автошколах ДОСААФ: мобильное приложение для учащихся и инструкторов (iOS, Android) и веб-панель для администрации филиалов.

## О проекте

Сейчас учащиеся автошкол ДОСААФ не могут сами выбрать инструктора, а запись на вождение ведётся через переписку в мессенджерах: инструкторы отвечают с задержкой, учащиеся не видят свободного времени.

Приложение позволяет учащемуся:

- войти в систему и увидеть свой прогресс обучения: часы вождения, посещаемость, результаты зачётов;
- выбрать инструктора по карточке (стаж, автомобиль и КПП, учебная площадка, рейтинг, процент сдачи);
- записаться на свободное время закреплённого инструктора и видеть календарь занятий.

Инструктор открывает время для записи и отмечает проведённые занятия, администратор филиала через веб-панель ведёт учащихся, группы, зачёты и отчёты.

Аналогичных решений в Республике Беларусь не выявлено. Конечные пользователи — около 50 тысяч учащихся в год в 87 филиалах ДОСААФ; доход — абонентская плата филиалов.

| Показатель | Значение |
|---|---|
| Объём инвестиций | 586 092,49 бел. руб. |
| Срок реализации | 18 мес. (окт. 2026 – март 2028) |
| Выручка в год после подключения всей сети | 712 800 бел. руб. |
| Срок окупаемости | ≈ 2,3 года |
| Точка безубыточности | 26 филиалов |

Подробнее — в [технико-экономическом обосновании](docs/01-feasibility/lab1-feasibility-study.pdf).


Техническое задание по проекту: **[docs/02-requirements/lab2-technical-specification.pdf](docs/02-requirements/lab2-technical-specification.pdf)**
## Структура репозитория

```
autoschool-app/
├── docs/                  проектная документация
│   ├── 01-feasibility/      ТЭО
│   ├── 02-requirements/     техническое задание
│   ├── 03-uml/              UML-диаграммы
│   ├── 04-architecture/     ER-модель, API (openapi.yaml), архитектура
│   └── 05-manuals/          руководства пользователя и системного администратора
├── prototypes/            прототипы экранов мобильного приложения
├── backend/               серверная часть и веб-панель (Django) — см. backend/README.md
│   ├── autoschool/          настройки проекта
│   ├── apps/                модули: accounts, branches, instructors,
│   │                        scheduling, progress, notifications,
│   │                        reports, audit
│   └── tests/
├── mobile/                мобильное приложение (Flutter) — см. mobile/README.md
│   ├── lib/                 core/ (API, хранилище, тема), features/ (экраны по сценариям)
│   └── test/
├── config/                конфигурация окружения: .env.example,
│                          docker-compose.yml, nginx
└── .github/               шаблоны Issue и Pull Request
```

## Сборка и запуск

> Шаги описывают **предполагаемый** порядок сборки. Они будут уточняться по мере разработки MVP.

**Требования к окружению:** Python 3.12, PostgreSQL 16, Flutter 3.24+ (Dart 3.5+), Docker и Docker Compose (для запуска всей системы), Git.

Систему можно запустить двумя способами, и в обоих сначала из шаблона создаётся файл .env. Первый способ — для разработки, по частям: в Docker поднимается PostgreSQL, применяются миграции, запускается Django-сервер с REST API и веб-панелью, а затем к нему подключается мобильное приложение на Flutter; второй — одной командой docker compose, которая поднимает всю систему (PostgreSQL, Django и Nginx с HTTPS) так же, как она будет развёрнута на серверах ДОСААФ.
### Серверная часть и веб-панель

```bash
# 1. Переменные окружения
cp config/.env.example .env            # заполнить пароли и секретный ключ

# 2. База данных PostgreSQL
docker compose -f config/docker-compose.yml --env-file .env up -d db

# 3. Зависимости и миграции
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements-dev.txt    # зависимости + инструменты проверки
python manage.py migrate
python manage.py createsuperuser       # администратор сети

# 4. Запуск
python manage.py runserver
```

- REST API: `http://localhost:8000/api/v1/`
- Веб-панель администратора: `http://localhost:8000/admin/`
- Документация API (генерируется из кода): `http://localhost:8000/api/v1/docs/`

### Мобильное приложение

```bash
cd mobile
flutter create . --platforms=android,ios --org by.dosaaf   # один раз: создать папки android/ и ios/
flutter pub get
flutter run --dart-define=API_BASE_URL=http://10.0.2.2:8000/api/v1   # эмулятор Android
```

### Вся система в Docker

```bash
docker compose -f config/docker-compose.yml --env-file .env up --build
```

Поднимаются PostgreSQL, сервер приложения (Gunicorn) и Nginx с HTTPS. Промышленная установка выполняется на серверах ДОСААФ (8 ядер, 16 ГБ ОЗУ, 500 ГБ SSD в RAID 1).

## Для разработчиков

### Технологии

| Слой | Технологии |
|---|---|
| Мобильное приложение | Flutter, Dart; dio (HTTP), flutter_secure_storage (Keychain / Android Keystore) |
| Сервер приложения | Python 3.12, Django 5, Django REST Framework, SimpleJWT |
| Веб-панель | Django Admin |
| База данных | PostgreSQL 16 |
| API | REST поверх HTTPS, JSON, базовый адрес `/api/v1`, OpenAPI 3 (drf-spectacular) |
| Развёртывание | Docker Compose, Gunicorn, Nginx (TLS 1.2+) |

### Архитектура сервера

Обработчики запросов (`views`) не обращаются к базе данных напрямую: каждый сценарий — закрепление инструктора, запись на занятие, смена статуса записи, оценка — вызывается одним методом фасада `BookingFacade` ([backend/apps/scheduling/services/facade.py](backend/apps/scheduling/services/facade.py)). Правила записи филиала реализуются «Стратегией» (`BookingLimitPolicy`), действия после изменения записи (уведомления, рейтинг, журнал аудита) — «Наблюдателем» (`EventPublisher`). Обоснование — в ЛР 4, раздел 3.
