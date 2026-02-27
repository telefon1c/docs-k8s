# docs.telefon1c.ru — Миграция в k3s с CDN

## Архитектура

```
docs.telefon1c.ru → Traefik (публичный, HTTPS/LE) → Traefik (k3s, NodePort 30629) → nginx pods (v4/v5)
cdn.docs.telefon1c.ru → Yandex CDN → Yandex S3 bucket (docs-telefon1c-cdn)
```

- Retype генерирует статику, nginx раздаёт HTML с CDN rewrite для ассетов
- ArgoCD + Image Updater автоматически деплоят новые образы

## Структура суперрепозитория (telefon1c/docs-k8s)

```
docs.telefon1c.ru/              ← telefon1c/docs-k8s (этот репо)
├── v4/                          # Инфраструктура v4: k8s манифесты + Dockerfile + nginx
├── v5/                          # Инфраструктура v5: k8s манифесты + Dockerfile + nginx
├── content/
│   ├── v4/                      # submodule → telefon1c/wiki
│   │   ├── src/                 #   контент документации v4
│   │   ├── retype.yml           #   конфиг Retype
│   │   └── .github/workflows/   #   CI v4 (caller → reusable)
│   └── v5/                      # submodule → telefon1c/wiki-v5
│       ├── src/                 #   контент документации v5
│       ├── retype.yml           #   конфиг Retype
│       └── .github/workflows/   #   CI v5 (caller → reusable)
└── .claude/CLAUDE.md
```

## Репозитории

| Репо | Назначение |
|------|-----------|
| `telefon1c/docs-k8s` | Суперрепо: K8s манифесты + submodules контента |
| `telefon1c/wiki` | Контент v4, retype.yml, CI caller (submodule в content/v4) |
| `telefon1c/wiki-v5` | Контент v5, retype.yml, CI caller (submodule в content/v5) |
| `telefon1c/.github` | Reusable workflow `retype-build.yml` (общий CI) |
| `k3s-miko/Docs-Telefon1c/` | Namespace, Ingress, ArgoCD Applications |

## K8s

- Namespace: `docs-telefon1c`
- Ingress: `/` и `/v4` → svc/docs-v4, `/v5` → svc/docs-v5
- Secret: `ghcr-pull-secret` (PAT с read:packages для ghcr.io)
- ArgoCD apps: `docs-telefon1c-v4`, `docs-telefon1c-v5` (source: `telefon1c/docs-k8s`, paths: `v4/`, `v5/`)

## CDN

- Bucket: `docs-telefon1c-cdn` (Yandex Object Storage, public read)
- CDN: `cdn.docs.telefon1c.ru` → origin `docs-telefon1c-cdn.storage.yandexcloud.net`
- SSL: Let's Encrypt через Yandex Certificate Manager
- Host header: `docs-telefon1c-cdn.storage.yandexcloud.net`

## GitHub Secrets

| Репо | Секреты |
|------|---------|
| `telefon1c/wiki` | `RETYPE_SECRET`, `YC_ACCESS_KEY`, `YC_SECRET_KEY` |
| `telefon1c/wiki-v5` | `RETYPE_SECRET`, `YC_ACCESS_KEY`, `YC_SECRET_KEY` |

## Выполненные фазы

- [x] Фаза 0 — Yandex S3 bucket, CDN resource, SSL, DNS для cdn.docs.telefon1c.ru, GitHub secrets
- [x] Фаза 1 — CI/CD: reusable workflow, Docker build → ghcr.io, S3 upload ассетов
- [x] Фаза 2 — Деплой в k3s: namespace, ingress, ArgoCD apps, imagePullSecret, поды Running
- [x] Фаза 3 — DNS переключен: `docs.telefon1c.ru` → CNAME `traefikserver.miko.ru` (93.188.43.131)
- [x] HTTPS на основном домене: публичный Traefik, `letsencrypt-http`, конфиг `headscale-dapl/traefik/dynamic/docs-telefon1c.yaml`

## Невыполненные задачи
- [ ] **GitHub Pages fallback**: убрать через 2 недели после DNS — удалить шаг из workflow и параметр `enable-github-pages`
- [ ] **Ротация PAT**: ghcr-pull-secret использует PAT с read:packages, ротировать до истечения (или сделать пакеты публичными)
- [x] ~~Удалить k8s/ из wiki и wiki-v5~~ — выполнено
- [ ] **Наполнить wiki-v5 контентом**: сейчас заглушка "Панель телефонии 5"
