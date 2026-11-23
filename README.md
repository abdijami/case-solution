# Sentry on Kubernetes — IOKA Senior DevOps case

## 1. Назначение

Репозиторий содержит инфраструктурную основу для развёртывания Sentry в Kubernetes с помощью Helm, конфигурации ClickHouse, а также GitLab CI pipeline для проверки и доставки изменений.

Решение использует Kubernetes-вариант из задания. В репозитории находятся:

- `infrastructure/sentry` — Helm chart Sentry;
- `infrastructure/clickhouse` — отдельный Helm chart ClickHouse;
- `app/.gitlab-ci.yaml` — GitLab CI pipeline;
- `app/self-hosted` — Dockerfile и конфигурация self-hosted Sentry;
- `infrastructure/sentry/tests` — проверки Helm-рендеринга и конфигурации.

> Важно: README описывает фактическое состояние переданного архива. Некоторые CI jobs пока являются заглушками, а production values требуют донастройки перед реальным production deployment. Эти ограничения перечислены в разделе [Известные ограничения](#9-известные-ограничения-и-доработки).

## 2. Архитектура

```text
                         GitLab
                           |
             +-------------+-------------+
             |             |             |
           test         security      validate
             |             |             |
             +-------------+-------------+
                           |
                     build / deploy
                           |
                    GitOps repository
                           |
                    Kubernetes cluster
                           |
          +----------------+----------------+
          |                |                |
       Sentry            Kafka           Redis
          |
      Snuba consumers
          |
      ClickHouse
          |
      PostgreSQL
```

### Компоненты

| Компонент | Назначение | Конфигурация в репозитории |
|---|---|---|
| Sentry Web / Relay | UI, API и приём событий | `infrastructure/sentry` |
| Snuba | Обработка и запросы к аналитическим данным | `infrastructure/sentry` |
| Kafka | Очередь событий и фоновой обработки | bundled dependency в Sentry chart включена в production values |
| Redis | Кэш и вспомогательная координация | bundled dependency включена |
| PostgreSQL | Реляционные метаданные Sentry | bundled dependency включена |
| ClickHouse | Аналитическое хранилище событий | внешний сервис для Sentry chart; отдельный chart находится в `infrastructure/clickhouse` |
| Filestore / Replay storage | Файлы и данные Session Replay | файловый backend задаётся в values |

Sentry chart имеет `version: 36.1.0` и `appVersion: 26.11.0`. Для ClickHouse chart указаны `version: 4.1.1` и `appVersion: 23.8.16.16`; сам chart помечен upstream как deprecated.

## 3. Требования

Для работы с Kubernetes deployment:

- Kubernetes cluster (для локального сценария подойдёт Minikube или kind);
- `kubectl`, настроенный на нужный cluster/context;
- Helm 3;
- Git;
- доступ к container registry, если собирается собственный образ;
- настроенный StorageClass / provisioner для PVC;
- настроенные секреты и доступ к ним из deployment pipeline.

Для CI тестов дополнительно:

- Python 3;
- PyYAML;
- Docker — для интеграционного теста PgBouncer;
- Helm 3.

Проверить доступ к кластеру:

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
helm version
```

## 4. Структура репозитория

```text
.
├── app/
│   ├── .gitlab-ci.yaml
│   ├── self-hosted/
│   │   ├── Dockerfile
│   │   ├── config.yml
│   │   ├── sentry.conf.py
│   │   └── docker_entrypoint.py
│   └── src/                         # исходный код Sentry
└── infrastructure/
    ├── clickhouse/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── prod-values/env-prod.yaml
    └── sentry/
        ├── Chart.yaml
        ├── values.yaml
        ├── prod-values/env-prod.yaml
        ├── ci/
        │   ├── kind-values.yaml
        │   └── services/clickhouse/
        ├── templates/
        └── tests/
```

`infrastructure/sentry/README.md` и `infrastructure/clickhouse/README.md` содержат справочную документацию соответствующих Helm charts. Этот файл описывает запуск и эксплуатацию именно данного репозитория.

## 5. Подготовка конфигурации и секретов

Все настройки окружения следует хранить в отдельных values-файлах, не меняя базовый `values.yaml` без необходимости.

Production values находятся в:

```text
infrastructure/sentry/prod-values/env-prod.yaml
infrastructure/clickhouse/prod-values/env-prod.yaml
```

В Sentry production values включены `feature-complete`, Kafka, Redis и PostgreSQL. Включён Session Replay (`sentry.features.enableSessionReplay: true`). Для файлового хранилища Sentry используется PVC размером `10Gi`.

### Секреты

В production values встречаются Vault reference strings для `CLICKHOUSE_PASSWORD` и `POSTGRES_PASSWORD`. Это не встроенная функция Helm: до передачи значений Helm должен существовать настроенный механизм разрешения таких ссылок (например, Vault-aware GitOps/CI integration).

Если такого механизма нет, создайте Kubernetes Secrets вне Git и подключите их через поддерживаемые chart-параметры `existingSecret` / `existingSecretKeys`. Пример для пароля администратора Sentry:

```bash
kubectl create namespace sentry

kubectl -n sentry create secret generic sentry-admin-password \
  --from-literal=admin-password="$(read -rsp 'Sentry admin password: ' value; echo "$value")"
```

Не передавайте реальные пароли через `--set`, не коммитьте `.env`, Secret manifests с открытыми значениями и kubeconfig.

Перед deployment проверьте, что секреты созданы и доступны в том же namespace:

```bash
kubectl -n sentry get secrets
```

> Имена Secret и ключей должны соответствовать значениям `existingSecret` / `existingSecretKey` в Helm values. Не создавайте Secret с приведённым примером имени, если chart настроен на другое имя.

## 6. Развёртывание

### 6.1. Подготовка namespace

```bash
kubectl create namespace sentry --dry-run=client -o yaml | kubectl apply -f -
kubectl config set-context --current --namespace=sentry
```

Убедитесь, что в кластере есть StorageClass:

```bash
kubectl get storageclass
```

Если используется не default StorageClass, задайте его явно в values для PVC-компонентов.

### 6.2. Зависимости Helm

Sentry chart содержит зависимости (Memcached, Redis, Kafka, PostgreSQL и Nginx с условиями включения). Загрузите зависимости:

```bash
cd infrastructure/sentry
helm dependency build
```

Если для конкретной версии Helm dependencies не закешированы, потребуется сетевой доступ к OCI registry, указанным в `Chart.yaml`.

### 6.3. Установка ClickHouse

Sentry настроен на внешний ClickHouse (`externalClickhouse`). Поэтому перед установкой Sentry должен быть доступен ClickHouse endpoint, соответствующий `externalClickhouse.host`, `tcpPort` и `httpPort`.

В репозитории есть отдельный chart:

```bash
cd infrastructure/clickhouse
helm dependency build
helm lint .
helm template clickhouse . \
  --namespace sentry \
  -f prod-values/env-prod.yaml
```

После проверки можно установить chart:

```bash
helm upgrade --install clickhouse . \
  --namespace sentry \
  -f prod-values/env-prod.yaml \
  --wait --timeout 15m
```

Это команда для имеющейся конфигурации, а не гарантия production readiness: в текущем `prod-values/env-prod.yaml` ClickHouse PVC отключены (`persistentVolumeClaim.enabled: false`). Перед реальным использованием необходимо включить постоянное хранилище, задать StorageClass и проверить backup/restore.

Проверьте доступность ClickHouse:

```bash
kubectl -n sentry get pods,svc
kubectl -n sentry logs statefulset/clickhouse-clickhouse
```

Имена ресурсов могут отличаться в зависимости от release name и helper templates; при необходимости найдите pod через `kubectl get pods -n sentry`.

### 6.4. Установка Sentry

Перед установкой:
1. Убедитесь, что ClickHouse доступен из namespace `sentry`.
2. Подготовьте PostgreSQL, Kafka и Redis — текущие production values включают bundled dependencies для этих компонентов.
3. Настройте обработку Vault references либо замените их на ссылки на Kubernetes Secrets.
4. Убедитесь, что задан пароль initial admin user через `user.password` или `user.existingSecret`.
5. Проверьте storage settings и включите persistence для всех stateful компонентов.

Проверка шаблонов:

```bash
cd infrastructure/sentry
helm dependency build
helm lint . -f prod-values/env-prod.yaml
helm template sentry . \
  --namespace sentry \
  -f prod-values/env-prod.yaml
```

Установка:

```bash
helm upgrade --install sentry . \
  --namespace sentry \
  -f prod-values/env-prod.yaml \
  --wait --timeout 30m
```

Проверка:

```bash
kubectl -n sentry get pods
kubectl -n sentry get svc
kubectl -n sentry get pvc
kubectl -n sentry get events --sort-by=.lastTimestamp
```

Production values сейчас задают `ingress.enabled: false`. Для доступа к UI потребуется настроить Ingress / Gateway / LoadBalancer и DNS/TLS в соответствии с инфраструктурой кластера.

### 6.5. Локальный CI-профиль kind

В репозитории есть `infrastructure/sentry/ci/kind-values.yaml`. Он предназначен для CI kind-кластера: профиль `errors-only`, однорепличная Kafka KRaft и Redis без persistence, а ClickHouse указывается как внешний сервис CI.

Также приложены манифесты ClickHouse для CI:

```text
infrastructure/sentry/ci/services/clickhouse/
```

Этот профиль не является production-конфигурацией. Он снижает требования к ресурсам и не должен использоваться для production данных.

## 7. Проверки

### 7.1. Helm lint и render

```bash
helm lint infrastructure/sentry \
  -f infrastructure/sentry/prod-values/env-prod.yaml

helm template sentry infrastructure/sentry \
  --namespace sentry \
  -f infrastructure/sentry/prod-values/env-prod.yaml \
  > /tmp/sentry-rendered.yaml

helm lint infrastructure/clickhouse \
  -f infrastructure/clickhouse/prod-values/env-prod.yaml

helm template clickhouse infrastructure/clickhouse \
  --namespace sentry \
  -f infrastructure/clickhouse/prod-values/env-prod.yaml \
  > /tmp/clickhouse-rendered.yaml
```

### 7.2. Python tests

Тесты проверяют рендеринг Helm chart, настройки credentials, Kafka, PgBouncer, mail, metrics, Argo CD annotations и persistence Symbolicator.

```bash
python3 -m pip install pyyaml

for test in infrastructure/sentry/tests/check_*.py; do
  echo "==> $test"
  python3 "$test"
done
```

`check_pgbouncer.py` использует Docker и запускает временные PostgreSQL/PgBouncer containers. Для него нужен работающий Docker daemon и возможность скачать образы.

### 7.3. Проверка после установки

```bash
kubectl -n sentry get pods -o wide
kubectl -n sentry get pvc
kubectl -n sentry get events --sort-by=.lastTimestamp

# Логи конкретного компонента:
kubectl -n sentry logs deploy/sentry-web --tail=200
kubectl -n sentry logs deploy/sentry-snuba-api --tail=200
```

Имена Deployment могут зависеть от release name и chart templates. Для получения точных имён используйте:

```bash
kubectl -n sentry get deployments,statefulsets
```

## 8. CI/CD

Pipeline находится в `app/.gitlab-ci.yaml`. Он подключает общие GitLab CI templates:

- `build/kaniko-template.yml`;
- `deploy/gitops-template.yml`.

Определены стадии:

| Stage | Job | Назначение |
|---|---|---|
| `test` | `test` | Тестирование |
| `security` | `security_scan` | Поиск уязвимостей |
| `validate` | `validate` | Проверка конфигурации |
| `build` | `build_image:test`, `build_image:prod` | Сборка образа через Kaniko |
| `deploy` | `deploy:test`, `deploy:prod` | Обновление image tag через GitOps |

Для test и prod используются разные registry variables и разные Helm values paths. Production build настроен как manual job на `main` / `master`.

### Необходимые GitLab CI variables

Согласно pipeline:

- `GITOPS_BOT_LOGIN`;
- `GITOPS_BOT_TOKEN`;
- `REGISTRY_URL_TEST`;
- `REGISTRY_BASE64_TEST`;
- `REGISTRY_URL_PROD`;
- `REGISTRY_BASE64_PROD`.

Переменные с токенами и registry credentials следует хранить в GitLab CI/CD Variables с Masked и Protected flags там, где это применимо.

### Что необходимо исправить в текущем pipeline

Текущие `test`, `security_scan` и `validate` содержат только `echo` и комментарии. Они не выполняют фактические проверки. Рекомендуемые команды для реализации:

```bash
# Test: Python chart tests
python3 -m pip install pyyaml
for test in infrastructure/sentry/tests/check_*.py; do
  python3 "$test"
done

# Security: filesystem / IaC / image scanning
trivy fs --exit-code 1 --severity HIGH,CRITICAL .

# Validate: Helm and YAML
helm lint infrastructure/sentry -f infrastructure/sentry/prod-values/env-prod.yaml
helm lint infrastructure/clickhouse -f infrastructure/clickhouse/prod-values/env-prod.yaml
helm template sentry infrastructure/sentry -f infrastructure/sentry/prod-values/env-prod.yaml > /tmp/sentry.yaml
```

Команды выше — рекомендуемое наполнение jobs, а не утверждение, что они уже выполняются текущим pipeline. Для production следует добавить image scan после сборки образа, закрепить версии инструментов и определить политику блокировки pipeline по severity.

Перед включением deploy необходимо сверить `needs` (`build_image:new_test` / `build_image:new_prod`) с фактическими именами build jobs (`build_image:test` / `build_image:prod`) и проверить контракты подключённых шаблонов.

Также `DOCKERFILE_PATH` сейчас указывает на `${CI_PROJECT_DIR}/Dockerfile`, хотя Dockerfile в архиве находится в `app/self-hosted/Dockerfile`. Исправьте `DOCKERFILE_PATH` и `CONTEXT_PATH` под фактическую структуру репозитория и убедитесь, что pipeline запускается из нужного project root.

## 9. Известные ограничения и доработки

Следующие пункты обнаружены при статическом анализе архива:

1. **CI проверки не реализованы полностью.** `test`, `security_scan` и `validate` — заглушки. Наличие стадий в YAML ещё не означает выполнение проверок.
2. **Build path требует проверки.** Dockerfile находится в `app/self-hosted/Dockerfile`, но CI указывает путь `${CI_PROJECT_DIR}/Dockerfile`.
3. **`needs` может ссылаться на отсутствующие jobs.** В deploy jobs указаны `build_image:new_test` и `build_image:new_prod`, в то время как в этом файле объявлены `build_image:test` и `build_image:prod`. Нужно проверить итоговую конфигурацию с подключёнными templates.
4. **Vault references требуют интеграции.** Строки `vault:secret/...` сами по себе не извлекают секреты из Vault. Необходимо подтвердить, какой компонент разрешает их до применения Helm.
5. **ClickHouse persistence отключён в текущем production values.** `persistentVolumeClaim.enabled`, `dataPersistentVolume.enabled` и `logsPersistentVolume.enabled` заданы как `false`. Для stateful production сервиса это необходимо исправить.
6. **ClickHouse chart deprecated.** Следует заменить его на поддерживаемый способ эксплуатации ClickHouse (например, ClickHouse Operator / поддерживаемый chart) либо явно принять и документировать риск.
7. **Session Replay storage требует настройки.** Feature включён, но `replay.storage.backend` пустой, а filesystem persistence для replay выключен. Нужен согласованный backend и проверка доступности хранилища.
8. **Ingress выключен.** `ingress.enabled: false`; для внешнего доступа нужны Ingress/Gateway, DNS и TLS.
9. **Resources не заданы для многих workloads.** Перед нагрузочным использованием необходимо определить requests/limits, HPA и capacity для Kafka, ClickHouse, PostgreSQL, Snuba и Sentry workers.
10. **Репозиторий содержит `.idea/workspace.xml`.** IDE workspace-файл не относится к инфраструктуре и обычно не должен попадать в репозиторий.

## 10. Troubleshooting

Пошаговые runbooks для инцидентов из задания находятся в отдельном файле [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md):

- ClickHouse CrashLoopBackOff после повреждения данных;
- OOMKilled при event spike;
- DiskPressure / Evicted / Read-Only;
- Session Replays: `Replays not available`.

## 11. Безопасность и эксплуатация

- Не хранить пароли, токены, kubeconfig и приватные ключи в Git.
- Использовать Kubernetes Secrets / Vault integration с минимально необходимыми правами.
- Включить TLS на внешних endpoint и ограничить сетевой доступ NetworkPolicy.
- Настроить регулярные backup PostgreSQL и ClickHouse и периодически проверять restore.
- Использовать PVC с подходящим StorageClass и мониторить свободное место.
- Задать requests/limits и алерты на рестарты, OOMKilled, lag Kafka consumers, ClickHouse disk usage и PVC capacity.
- Фиксировать версии Helm chart, образов и зависимостей; обновлять их через проверяемый процесс.
