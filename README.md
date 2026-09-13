# devops-gitops (стартер)

GitOps-репозиторий для ДЗ-7. Создан из template `devops-gitops-starter`. Содержит
Helm-чарты приложения и конфигурацию ArgoCD. **Это отдельный ПУБЛИЧНЫЙ репозиторий** —
отдельно от репозитория с кодом приложения (`devops-hometask-01`).

> Репо публичный, потому что секреты в нём **зашифрованы SOPS** — шифротекст безопасно
> лежит в открытом git. За счёт этого ArgoCD читает репо по HTTPS без ключей, а
> проверяющий может убедиться, что секреты действительно зашифрованы. **Единственное
> железное правило: приватный `age-key.txt` НИКОГДА не коммить** (он в `.gitignore`,
> а CI с gitleaks дополнительно ловит его, если он случайно попадёт в коммит).

## Структура

```
helm/
├── postgres/          # StatefulSet БД — ставится ВРУЧНУЮ (helm install), вне ArgoCD
└── myapp/             # backend + frontend + ingress — под управлением ArgoCD
argocd-values.yaml     # values для установки ArgoCD (helm-secrets + сервисный аккаунт + ingress)
application.yaml       # ArgoCD Application для myapp
.sops.yaml             # правило шифрования секретов (твой age-публичный ключ)
.gitleaks.toml         # конфиг gitleaks
.github/workflows/gitleaks.yml  # CI: скан истории на утечку секретов/ключа
```

## Что заполнить (плейсхолдеры)

| Где | Плейсхолдер | На что заменить |
|---|---|---|
| `helm/myapp/values.yaml` | `ВАШ_ЛОГИН` | твой логин Docker Hub |
| `helm/myapp/values.yaml` | `ВАШ_ДОМЕН` | твой домен (как в hw3/hw6) |
| `argocd-values.yaml` | `argocd.ВАШ_ДОМЕН` | сабдомен под ArgoCD UI |
| `application.yaml` | `https://github.com/ВАШ_ЛОГИН/devops-gitops.git` | HTTPS-адрес ЭТОГО репо |
| `.sops.yaml` | `age1ПЛЕЙСХОЛДЕР...` | твой публичный age-ключ |

## Порядок (кратко — подробности в методичке 07/02)

1. **Секреты.** `age-keygen -o age-key.txt` → вставь публичный ключ в `.sops.yaml`.
   Скопируй `secrets.example.yaml` → `secrets.yaml` в обоих чартах, подставь значения,
   зашифруй: `sops -e -i helm/postgres/secrets.yaml && sops -e -i helm/myapp/secrets.yaml`.
   Закоммить зашифрованные `secrets.yaml` (приватный ключ — НЕ коммить).

   > ⚠️ Креды БД (`DB_USER`/`DB_PASSWORD`) в `helm/postgres/secrets.yaml` и
   > `helm/myapp/secrets.yaml` **должны совпадать** — postgres ими инициализируется,
   > backend ими же подключается.

2. **Postgres (вручную, один раз):**
   ```bash
   helm install postgres ./helm/postgres \
     -f helm/postgres/values.yaml \
     -f secrets://helm/postgres/secrets.yaml
   ```

3. **dockerhub-cred** (доступ к приватным образам) — создаётся `kubectl` в namespace default.

4. **ArgoCD:** приватный ключ в кластер + установка:
   ```bash
   kubectl create namespace argocd
   kubectl create secret generic helm-secrets-private-keys \
     -n argocd --from-file=key.txt=age-key.txt
   helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
   helm install argocd argo/argo-cd -n argocd --version 10.2.2 -f argocd-values.yaml
   ```

5. **Создать Application** (репо публичный → deploy key НЕ нужен, ArgoCD читает по HTTPS):
   ```bash
   kubectl apply -f application.yaml
   argocd app get myapp     # → Synced / Healthy
   ```

6. **CI** (в репо `devops-hometask-01`) бампает теги образов в `values-production.yaml`
   этого репо через PR и дёргает `argocd app sync`.

## Что НЕ коммитить
`age-key.txt` (приватный ключ), `charts/`, `*.tgz`. Всё в `.gitignore`.
`secrets.yaml` коммитим — но **только зашифрованными** (значения `ENC[AES256_GCM,...]`).
gitleaks в CI страхует: если ключ или плейнтекст-секрет попадёт в коммит — джоба упадёт.
