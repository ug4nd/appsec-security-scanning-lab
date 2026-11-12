# AppSec Security Scanning Lab

Учебный AppSec-проект: автоматическое сканирование намеренно уязвимого приложения [OWASP Juice Shop](https://github.com/juice-shop/juice-shop) (форк, лицензия MIT). Сканируется только мой локальный стенд.

## Что сделано

| Слой | Инструмент | Что внутри |
|---|---|---|
| SAST | Semgrep | CI-workflow `.github/workflows/semgrep.yml`, результаты в GitHub Security (SARIF), своё правило `.semgrep/raw-sequelize-query.yaml` |
| SCA | Trivy | CI-workflow `.github/workflows/trivy.yml` (PR, push, раз в сутки), lockfile в репозитории, исключения в `.trivyignore` с причиной |
| DAST | OWASP ZAP | Ручной обход, AJAX Spider, Active Scan по `127.0.0.1:3000`, отчёт в `reports/zap/` |

## Результаты (2026-11-12)

- Semgrep: 40 находок
- Trivy: 35 (6 Critical, 29 High)
- ZAP: 3 High, 11 Medium, 8 Low, 10 Info

SQL-инъекции в `/rest/user/login` и `/rest/products/search` нашёл ZAP, и те же места указал Semgrep в `routes/login.ts` и `routes/search.ts`: SAST и DAST совпали.

Таблица находок с CWE, категорией OWASP Top 10, способом подтверждения и рекомендациями: [`reports/README.md`](reports/README.md).

## Ограничения

- Forced Browse в ZAP был прерван (падение целевого контейнера), в отчёте 81% сетевых ошибок: покрытие могло быть неполным.
- Часть алертов не проверена вручную.
- Trivy-job красный намеренно: в Juice Shop уязвимости заложены по замыслу.
- Из отчётов удалены токены и значения паролей.

## Оригинальный проект

Описание Juice Shop: [`README.juice-shop.md`](README.juice-shop.md). Автор и лицензия: OWASP Juice Shop, см. `LICENSE`.
