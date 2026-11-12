# Отчёты сканирования OWASP Juice Shop

Цель: локальный стенд OWASP Juice Shop (Docker, `127.0.0.1:3000`). Это намеренно уязвимое учебное приложение, сканировался только мой локальный стенд.
Дата: 2026-11-12.

| Инструмент | Тип | Результат | Файл |
|---|---|---|---|
| Semgrep 1.182.1 | SAST | 40 находок (конфиг из CI: `.semgrep/` + `p/owasp-top-ten`, без `reports/`) | `semgrep/semgrep.sarif` |
| Trivy 0.76.1 | SCA | 35 уязвимостей: 6 Critical, 29 High (`--scanners vuln`, с `.trivyignore`) | `trivy/trivy-fs.json` |
| ZAP 2.17.0 | DAST | 3 High, 11 Medium, 8 Low, 10 Info (Default Policy) | `zap/zap-juice-shop-2026-11-12.html` |

## Ключевые находки

| # | Находка | CWE | OWASP Top 10 (2021) | Где | Как подтверждено | Что исправить |
|---|---|---|---|---|---|---|
| 1 | SQL Injection | 89 | A03 Injection | `/rest/user/login`, `/rest/products/search?q=` | ZAP Active Scan (2) + Semgrep `express-sequelize-injection`, `raw-sequelize-query` в `routes/login.ts`, `routes/search.ts`. SAST и DAST совпали | Параметризованные запросы (replacements/bind), без конкатенации строк |
| 2 | External / Off-site Redirect | 601 | A03 по тегу ZAP, A01 по MITRE | параметр редиректа | ZAP Active Scan + Semgrep `express-open-redirect` | Белый список разрешённых адресов |
| 3 | Секрет подписи JWT в коде; JWT в localStorage | 798, 922 | не присвоено | `lib/insecurity.ts`; localStorage браузера | Semgrep `hardcoded-jwt-secret`; ZAP (пассивно) | Ключ из переменных окружения или хранилища секретов; сессия в HttpOnly-cookie. Вместе с XSS даёт кражу сессии |
| 4 | CSP отсутствует или слабая (5 алертов) | 693 | не присвоено | все ответы | ZAP (пассивно) | Задать `Content-Security-Policy` без `unsafe-eval`, `unsafe-inline`, wildcard |
| 5 | CORS `Access-Control-Allow-Origin: *` | 264 по ZAP (точнее 942) | не присвоено | все ответы | ZAP (пассивно) | Ограничить список источников |
| 6 | Persistent XSS weakness в JSON-ответе (9) | 79 | A03 | API-ответы | ZAP (пассивно), эксплуатация не проверялась | Кодирование на выводе, проверить вручную |
| 7 | Cookie без HttpOnly / SameSite | 1004, 1275 | не присвоено | cookies | ZAP (пассивно) | Выставить флаги |
| 8 | Листинг каталогов и `res.sendFile` | 548, 73 | не присвоено | серверный код (`/ftp`) | Semgrep `express-check-directory-listing` (5), `express-res-sendfile` (4) | Отключить листинг, фиксировать корень и проверять путь |
| 9 | Уязвимые зависимости | разные | A06 Vulnerable Components | `package-lock.json`, `frontend/package-lock.json` | Trivy: 6 Critical, 29 High. Исключения в `.trivyignore` с причиной (decompress, marsdb) | Обновить; lodash 2.4.2 приходит через `sanitize-html@1.4.2` |
| 10 | IaC: публичные IP подсетей, старый TLS у балансировщика | н/д | не присвоено | `infrastructure/terraform`, `terraform/` | Semgrep `aws-subnet-has-public-ip-address` (5), `insecure-load-balancer-tls-version` (5) | Отключить публичные IP, TLS 1.2+ |
| 11 | GitHub Actions по изменяемым тегам | н/д | A08 | `.github/workflows` | Semgrep `github-actions-mutable-action-tag` (5) | Закрепить версию по SHA |

Категории Top 10 со значением «не присвоено» я пока не сверял с тегами ZAP, их надо проверить в Alert Tags перед публикацией.

## Ограничения (честно)

- Forced Browse запускался и был прерван: Node в контейнере упал (heap out of memory). Контейнер пересоздан с `NODE_OPTIONS=--max-old-space-size=3072`, Active Scan шёл с 2 потоками на хост.
- В Insights отчёта ZAP стоит 81% сетевых ошибок. Найденные алерты подтверждены ответами сервера, но отсутствие других находок ничего не доказывает: покрытие могло быть неполным.
- Часть алертов (например, Format String Error) не проверена вручную и может быть ложным срабатыванием.
- В отчёте ZAP удалены JWT и значения полей `password`, `totpSecret` (заменены на `REDACTED`). Сканировался отдельный тестовый аккаунт.
- Три приватных ключа в Juice Shop намеренные, это часть учебного приложения.

## Воспроизведение

    semgrep scan --config .semgrep/ --config p/owasp-top-ten --exclude reports --sarif -o reports/semgrep/semgrep.sarif .
    trivy fs --scanners vuln --severity CRITICAL,HIGH --ignorefile .trivyignore -f json -o reports/trivy/trivy-fs.json .

ZAP: ручной обход + AJAX Spider + Active Scan по `http://127.0.0.1:3000` (Default Context, режим Protected).
