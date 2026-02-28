# docs.telefon1c.ru — k3s + CDN

## Архитектура

```
docs.telefon1c.ru → Traefik (публичный, HTTPS/LE) → Traefik (k3s, NodePort 30629) → nginx pods (v4/v5)
cdn.docs.telefon1c.ru → Yandex CDN → Yandex S3 bucket (docs-telefon1c-cdn)
```

- docs.telefon1c.ru/ → 301 → /v4/ (для переиндексации поисковиками)
- Retype генерирует статику, nginx раздаёт HTML с CDN rewrite для ассетов
- ArgoCD + Image Updater (стратегия digest) автоматически деплоят новые образы
- Контент v4 и v5 хранится в `/usr/share/nginx/html/v4/` и `/v5/` соответственно (одинаковая схема)

## Маршрутизация

| URL | Действие |
|-----|----------|
| `/` | 301 → `/v4/` |
| `/v4/*` | nginx pod docs-v4, sub_filter → CDN |
| `/v5/*` | nginx pod docs-v5, sub_filter → CDN |

## Структура суперрепозитория (telefon1c/docs-k8s)

```
docs.telefon1c.ru/              ← telefon1c/docs-k8s (этот репо)
├── v4/                          # Инфраструктура v4: k8s манифесты + Dockerfile + nginx
├── v5/                          # Инфраструктура v5: k8s манифесты + Dockerfile + nginx
├── content/
│   ├── v4/                      # submodule → telefon1c/wiki (HTTPS)
│   │   ├── src/                 #   контент документации v4
│   │   ├── retype.yml           #   конфиг Retype (url: /v4/)
│   │   └── .github/workflows/   #   CI v4 (caller → reusable)
│   └── v5/                      # submodule → telefon1c/wiki-v5 (HTTPS)
│       ├── src/                 #   контент документации v5
│       ├── retype.yml           #   конфиг Retype (url: /v5/)
│       └── .github/workflows/   #   CI v5 (caller → reusable)
└── .claude/CLAUDE.md
```

## Репозитории

| Репо | Назначение |
|------|-----------|
| `telefon1c/docs-k8s` | Суперрепо: K8s манифесты + Dockerfile + nginx + submodules контента |
| `telefon1c/wiki` | Контент v4, retype.yml (url: /v4/), CI caller (submodule в content/v4) |
| `telefon1c/wiki-v5` | Контент v5, retype.yml (url: /v5/), CI caller (submodule в content/v5) |
| `telefon1c/.github` | Reusable workflow `retype-build.yml` (общий CI) |
| `k3s-miko/Docs-Telefon1c/` | Namespace, Ingress, ArgoCD Applications |
| `headscale-dapl/traefik/dynamic/` | `docs-telefon1c.yaml` — публичный Traefik роутер |

## K8s

- Namespace: `docs-telefon1c`
- Ingress: `/` и `/v4` → svc/docs-v4, `/v5` → svc/docs-v5
- Secrets: `ghcr-pull-secret` (PAT с read:packages для ghcr.io) — в ns `docs-telefon1c` и `argocd`
- ArgoCD apps: `docs-telefon1c-v4`, `docs-telefon1c-v5` (source: `telefon1c/docs-k8s`, paths: `v4/`, `v5/`)
- ArgoCD repo secret `repo-docs-k8s`: enableSubmodules=false (submodules не нужны для деплоя)
- Image Updater: стратегия `digest`, pull-secret `argocd/ghcr-pull-secret`

## CDN и S3

- Bucket: `docs-telefon1c-cdn` (Yandex Object Storage, public read)
- CDN: `cdn.docs.telefon1c.ru` → origin `docs-telefon1c-cdn.storage.yandexcloud.net`
- SSL: Let's Encrypt через Yandex Certificate Manager
- CI загружает assets/ и resources/ в S3, CSS файлы перезаливаются с `--mime-type=text/css`

## Nginx sub_filter (CDN rewrite)

Retype генерирует пути в нескольких форматах:
- `/assets/...` — абсолютные без префикса (custom assets)
- `resources/...` — относительные (Retype core JS/CSS, отдаются nginx напрямую)
- `../assets/`, `../../assets/` — относительные (вложенные страницы)
- `, /assets/...` — внутри srcset

sub_filter заменяет `/assets/` и относительные `../assets/` на CDN URL.
Относительные `resources/` отдаются nginx как статика (30d cache).

## Выполненные фазы

- [x] Фаза 0 — Yandex S3 bucket, CDN resource, SSL, DNS для cdn.docs.telefon1c.ru, GitHub secrets
- [x] Фаза 1 — CI/CD: reusable workflow, Docker build → ghcr.io, S3 upload ассетов
- [x] Фаза 2 — Деплой в k3s: namespace, ingress, ArgoCD apps, imagePullSecret, поды Running
- [x] Фаза 3 — DNS переключен: `docs.telefon1c.ru` → CNAME `traefikserver.miko.ru` (93.188.43.131)
- [x] HTTPS: публичный Traefik, `letsencrypt-http`, конфиг `headscale-dapl/traefik/dynamic/docs-telefon1c.yaml`
- [x] Редирект `/` → `/v4/` (301, https)
- [x] v4 retype.yml: url `/v4/`, контент в `/usr/share/nginx/html/v4/` (аналогично v5)
- [x] ArgoCD Image Updater: digest strategy, pull-secret для ghcr.io, repo secret без submodules
- [x] S3 MIME fix: CSS файлы загружаются с `--mime-type=text/css`

## Невыполненные задачи

- [x] ~~GitHub Pages fallback~~ — удалён шаг и параметр `enable-github-pages` из workflow
- [x] ~~Ротация PAT~~ — PAT без срока окончания
- [ ] **Наполнить wiki-v5 контентом**: сейчас заглушка "Панель телефонии 5"
- [x] ~~Убрать --no-check-md5 из s3cmd sync~~ — уже убран
