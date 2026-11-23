# TROUBLESHOOTING — Sentry on Kubernetes

Документ описывает порядок диагностики и восстановления для четырёх сценариев из практического задания. Команды предполагают Kubernetes namespace `sentry`; при другом namespace замените `-n sentry`.

> Принцип восстановления: сначала сохранить данные и диагностические артефакты, затем локализовать первопричину, устранить её и только после этого перезапускать или пересоздавать workload. Не удаляйте PVC, ClickHouse data parts или каталоги PostgreSQL ради быстрого снятия ошибки.

## Общая первичная диагностика

```bash
kubectl -n sentry get pods -o wide
kubectl -n sentry get events --sort-by=.lastTimestamp
kubectl -n sentry get pvc
kubectl -n sentry get statefulsets,deployments
kubectl top nodes
kubectl top pods
```

Для конкретного pod:

```bash
kubectl -n sentry describe pod <pod>
kubectl -n sentry logs <pod> --all-containers --tail=300
kubectl -n sentry logs <pod> --all-containers --previous --tail=300
```

`--previous` особенно важен при CrashLoopBackOff: он показывает логи предыдущего завершившегося контейнера.

---

## 1. ClickHouse CrashLoopBackOff после аварийного отключения

### Симптомы

- ClickHouse pod находится в `CrashLoopBackOff`.
- В логах сообщения о повреждённых data parts, metadata или checksums.
- Sentry / Snuba не могут читать или записывать аналитические события.

### Архитектурный подход

ClickHouse хранит данные на persistent volume. При подозрении на повреждение сначала фиксируем состояние и сохраняем копию диска / snapshot, если это поддерживает StorageClass. Не следует удалять PVC или вручную удалять файлы таблиц до определения повреждённых объектов.

### Шаг 1. Найти pod и получить причину завершения

```bash
kubectl -n sentry get pods -o wide | grep -i clickhouse
kubectl -n sentry describe pod <clickhouse-pod>
kubectl -n sentry logs <clickhouse-pod> --previous --tail=500
```

Проверить события node и volume:

```bash
kubectl -n sentry get events --sort-by=.lastTimestamp | tail -n 50
kubectl describe node <node-name>
kubectl -n sentry get pvc
```

Проверить, не было ли `OOMKilled`, `Evicted`, проблем с filesystem или detach/attach PVC. Если ClickHouse запущен не в namespace `sentry`, выполнять команды в namespace его StatefulSet.

### Шаг 2. Зафиксировать состояние и сделать snapshot

1. Остановить автоматические изменения конфигурации / rollout, чтобы не усугубить ситуацию.
2. Сохранить логи и `describe`.
3. По возможности сделать VolumeSnapshot средствами CSI либо storage-level snapshot.
4. Проверить целостность и доступность самого volume.

```bash
kubectl -n sentry get pvc
kubectl get volumesnapshot -A
```

Если VolumeSnapshot CRD / CSI snapshotter не установлен, использовать штатный snapshot механизм storage platform. Не запускать repair на единственной копии данных.

### Шаг 3. Проверить ClickHouse изолированно

Если контейнер удаётся запустить, проверить системные таблицы:

```sql
SELECT database, table, engine
FROM system.tables
ORDER BY database, table;

SELECT *
FROM system.detached_parts
ORDER BY database, table;
```

Проверить ошибки в логах ClickHouse и список повреждённых / detached parts. Если pod не стартует, диагностику выполнять на копии volume в изолированном recovery pod, используя ту же версию ClickHouse и read-only mount, если это допускает процедура.

### Шаг 4. Восстановить данные

Приоритет вариантов:

1. Восстановить ClickHouse из проверенного backup.
2. Если повреждены отдельные parts, использовать документированную для конкретной версии ClickHouse процедуру восстановления после анализа логов и detached parts.
3. Для replicated tables — проверить состояние реплик и Keeper/ZooKeeper, затем восстановить репликацию штатными средствами.
4. Если повреждение затрагивает только часть данных и backup отсутствует, согласовать допустимую потерю данных с владельцем сервиса до любых destructive операций.

Не использовать `rm -rf` по каталогам `store`, `metadata`, `detached`, не удалять PVC и не выполнять `DROP TABLE` / `DETACH TABLE` без подтверждённого плана восстановления.

### Шаг 5. Возвратить workload и проверить Sentry

```bash
kubectl -n sentry get pods -w
kubectl -n sentry logs <clickhouse-pod> --tail=200
kubectl -n sentry get endpoints
```

Проверить доступность ClickHouse по HTTP/TCP из namespace Sentry, затем проверить логи Snuba API и consumers. Убедиться, что backlog Kafka уменьшается и новые события появляются в Sentry.

### Профилактика

- Persistent volumes и регулярные snapshots.
- Backup с проверкой восстановления.
- Корректное завершение ClickHouse и корректная обработка node shutdown.
- Мониторинг disk latency, filesystem errors, ClickHouse parts и replica health.
- Не использовать production values, в которых persistence ClickHouse отключён.

---

## 2. OOMKilled при резком росте входящего трафика (event spike)

### Симптомы

- Sentry web, Relay, Snuba consumers или task workers перезапускаются.
- `kubectl describe pod` показывает `Reason: OOMKilled` / exit code `137`.
- Увеличивается Kafka consumer lag, задерживается обработка событий.

### Шаг 1. Определить компонент и масштаб проблемы

```bash
kubectl -n sentry get pods
kubectl -n sentry describe pod <pod>
kubectl -n sentry get events --sort-by=.lastTimestamp | grep -i -E 'oom|killing'
kubectl top pods -n sentry --containers
kubectl top nodes
```

Посмотреть предыдущие логи:

```bash
kubectl -n sentry logs <pod> --previous --all-containers --tail=300
```

Проверить фактические requests/limits:

```bash
kubectl -n sentry get pod <pod> \
  -o jsonpath='{range .spec.containers[*]}{.name}{" requests="}{.resources.requests}{" limits="}{.resources.limits}{"\n"}{end}'
```

Отличать OOMKilled контейнера от нехватки памяти на node. Проверить node pressure и системные логи на уровне инфраструктуры.

### Шаг 2. Снизить давление на систему

- Ограничить или временно снизить объём входящего трафика на upstream / SDK уровне, если это возможно.
- Применить sampling / rate limiting для шумных проектов и повторяющихся событий.
- Проверить, нет ли poison messages или бесконечных retries.
- Не масштабировать все компоненты одновременно: это может увеличить нагрузку на PostgreSQL, Kafka и ClickHouse.

### Шаг 3. Найти узкое место

Проверить:
- Kafka consumer lag и throughput;
- CPU / memory потребление Relay, Snuba consumers, task workers;
- число и размер batch;
- ClickHouse insert/query latency;
- PostgreSQL connections, locks и slow queries;
- node allocatable memory и memory pressure.

Для Kafka используйте штатные consumer group tools / метрики используемого Kafka deployment. Для ClickHouse:

```sql
SELECT *
FROM system.metrics
WHERE metric ILIKE '%Memory%';

SELECT query, elapsed, memory_usage
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 20;
```

### Шаг 4. Увеличить доступные ресурсы и/или горизонтально масштабировать

Сначала скорректировать `resources.requests` и `resources.limits` для конкретного workload в values. Requests должны отражать ожидаемое нормальное потребление, limits — согласованный верхний предел, который не приводит к постоянным OOM при штатной нагрузке.

Для stateless consumers / workers можно увеличить replicas, если downstream способен принять дополнительную нагрузку. Проверить HPA:

```bash
kubectl -n sentry get hpa
kubectl -n sentry describe hpa <hpa-name>
```

Не увеличивать replicas stateful database без понимания модели репликации и требований к storage.

Применить изменения через Helm/GitOps, а не ручным `kubectl edit`, чтобы desired state не перезаписал исправление:

```bash
helm upgrade --install sentry infrastructure/sentry \
  -n sentry \
  -f infrastructure/sentry/prod-values/env-prod.yaml \
  -f /secure/path/incident-overrides.yaml \
  --wait --timeout 30m
```

`incident-overrides.yaml` должен содержать только согласованные resource/replica overrides и не содержать секретов.

### Шаг 5. Проверить восстановление обработки

```bash
kubectl -n sentry get pods -w
kubectl -n sentry top pods --containers
kubectl -n sentry get hpa
```

Убедиться, что:
- OOM рестарты прекратились;
- Kafka lag стабилизируется и снижается;
- Snuba consumers успевают обрабатывать сообщения;
- ClickHouse и PostgreSQL не стали новым bottleneck;
- latency и error rate вернулись к норме.

### Профилактика

- Requests/limits для каждого компонента.
- HPA для подходящих stateless workloads.
- Capacity planning и нагрузочное тестирование event spike.
- Sampling / quotas / rate limits.
- Алерты на OOMKilled, memory working set, Kafka lag и node memory pressure.

---

## 3. DiskPressure, заполнение диска на 99%, Read-Only и Evicted

### Симптомы

- Node condition `DiskPressure=True`.
- Pod получает статус `Evicted`.
- PVC или filesystem почти заполнены.
- PostgreSQL / ClickHouse переходят в read-only или не могут записывать данные.
- Появляются ошибки `No space left on device`.

### Шаг 1. Определить, какой именно диск заполнен

Проверить node:

```bash
kubectl describe node <node-name>
kubectl get events -A --sort-by=.lastTimestamp | grep -i -E 'diskpressure|evict|no space'
```

Проверить PVC и pod mounts:

```bash
kubectl -n sentry get pvc
kubectl -n sentry describe pvc <pvc-name>
kubectl -n sentry describe pod <pod>
```

Если есть доступ на node:

```bash
df -h
df -ih
sudo du -xhd1 /var/lib/containerd /var/log /var/lib/kubelet 2>/dev/null | sort -h
```

В контейнере / DB pod (если он работает):

```bash
df -h
du -xhd1 /var/lib/clickhouse /var/lib/postgresql 2>/dev/null | sort -h
```

Нужно различать:
- ephemeral storage node (container logs, writable layers, kubelet);
- PVC, на котором лежат данные БД;
- inode exhaustion;
- quota / capacity на стороне storage backend.

### Шаг 2. Безопасно освободить место

1. Найти крупнейших потребителей.
2. Удалять только подтверждённо ненужные container logs, временные файлы и старые артефакты согласно retention policy.
3. Проверить ClickHouse TTL / retention и политику хранения событий.
4. Проверить PostgreSQL WAL, replication slots и backup retention. Не удалять WAL вручную.
5. Не удалять данные из PVC и каталоги баз вручную.

Для логов Kubernetes использовать штатную ротацию container runtime / logrotate. Для ClickHouse — TTL / partition lifecycle и штатные SQL операции после проверки retention. Для PostgreSQL — штатные механизмы архивации и очистки WAL.

### Шаг 3. Увеличить ёмкость

Если заполнен PVC и StorageClass поддерживает expansion:

1. Проверить `allowVolumeExpansion: true` у StorageClass.
2. Увеличить `spec.resources.requests.storage` у PVC (или параметр размера в Helm values, если PVC управляется chart).
3. Дождаться расширения filesystem.
4. Проверить `kubectl describe pvc` и `df -h` внутри pod.

```bash
kubectl get storageclass
kubectl describe storageclass <storage-class>
kubectl -n sentry patch pvc <pvc-name> \
  -p '{"spec":{"resources":{"requests":{"storage":"<new-size>"}}}}'
```

Пример `<new-size>` — `200Gi`; выберите размер по фактической нагрузке и доступной ёмкости storage. Увеличение PVC не всегда обратимо и не заменяет backup.

Если заполнен диск node, расширить node disk / увеличить capacity node pool и только затем возвращать workload. Для ephemeral storage задать адекватные requests/limits и log rotation.

### Шаг 4. Восстановить workloads

```bash
kubectl get nodes
kubectl describe node <node-name>
kubectl -n sentry get pods
kubectl -n sentry get pvc
```

После освобождения места kubelet обычно снимает DiskPressure самостоятельно. Evicted pods, управляемые Deployment/StatefulSet, должны быть пересозданы контроллерами; если этого не произошло, выяснить причину scheduling failure прежде чем удалять pod вручную.

Проверить read-only recovery баз:
- PostgreSQL: логи сервера, состояние filesystem, доступность WAL и свободное место;
- ClickHouse: логи, состояние data parts и доступность volume.

Не выполнять `fsck` на смонтированном read-write filesystem. Процедуры filesystem repair выполнять только в согласованном maintenance окне и после snapshot/backup.

### Профилактика

- Алерты на PVC usage, node filesystem usage и inode usage (например, 70/85/95%).
- Retention policy для событий, логов и replay data.
- Регулярное расширение storage capacity по тренду.
- `ephemeral-storage` requests/limits и log rotation.
- Проверенные backup и restore.
- Включённый persistence для ClickHouse. В текущем ClickHouse production values PVC отключён — это блокер для production эксплуатации.

---

## 4. Session Replays: `Replays not available`

### Симптомы

- Sentry UI показывает `Replays not available`.
- Replay metadata может отображаться, но воспроизведение не загружает запись.
- Ошибка может быть связана с ingest, consumer, storage backend, правами доступа или сетевым доступом.

### Архитектурный подход

Session Replay требует согласованной работы:
1. SDK и ingest endpoint должны отправлять replay events.
2. Relay / ingest и Snuba consumers должны обработать сообщения.
3. Replay blobs должны сохраняться в настроенное хранилище.
4. Sentry API / Vroom должны иметь доступ к тому же хранилищу.
5. Browser должен иметь доступ к replay API и URL ресурсов.

В текущих production values `sentry.features.enableSessionReplay` включён, но `replay.storage.backend` пустой, а `replay.storage.filesystem.persistence.enabled` выключен. Это нужно исправить до проверки функциональности: включённый feature flag сам по себе не означает, что replay storage настроен.

### Шаг 1. Проверить настройки Helm

```bash
grep -n -A80 '^replay:' infrastructure/sentry/prod-values/env-prod.yaml
grep -n -A20 'enableSessionReplay' infrastructure/sentry/prod-values/env-prod.yaml
```

Проверить:
- выбран ли backend для replay storage;
- задан ли bucket / endpoint / credentials для object storage, если используется S3/GCS;
- включён ли PVC, если используется filesystem;
- совпадают ли настройки backend у компонентов, которые записывают и читают replay;
- включён ли доступ к хранилищу из namespace / pod security context.

Для filesystem проверить PVC и mount:

```bash
kubectl -n sentry get pvc
kubectl -n sentry get pods -o wide
kubectl -n sentry describe pod <sentry-web-or-vroom-pod>
```

### Шаг 2. Проверить компоненты ingest и consumers

```bash
kubectl -n sentry get deployments | grep -E 'relay|snuba|replay|vroom|ingest'
kubectl -n sentry logs deploy/sentry-relay --tail=300
kubectl -n sentry logs deploy/sentry-snuba-api --tail=300
```

Имена deployment уточнить через `kubectl get deployments`. Проверить логи replay consumer / ingest consumer, Relay и Vroom. Искать ошибки авторизации, timeout, `NoSuchKey`, `AccessDenied`, connection refused, ошибки Kafka и записи в storage.

Проверить Kafka consumer lag и наличие новых сообщений соответствующих replay topics средствами Kafka deployment.

### Шаг 3. Проверить storage backend

Для S3-compatible storage:
- проверить endpoint, bucket и region;
- проверить credentials и IAM/policy permissions;
- убедиться, что pod может разрешить DNS и установить соединение;
- проверить TLS CA / certificate validation;
- проверить, что URL, возвращаемый API, доступен браузеру клиента.

Для filesystem:
- проверить, что PVC смонтирован в нужный путь;
- проверить `fsGroup` / UID/GID и права на каталог;
- убедиться, что writer и reader используют общий volume, если архитектура требует shared storage;
- проверить свободное место и inode.

### Шаг 4. Проверить конкретную replay запись

Выбрать одну проблемную запись и сопоставить:
- replay ID;
- timestamp;
- project / organization;
- ошибки ingest в тот же временной интервал;
- наличие соответствующего объекта в storage;
- HTTP status ответа replay API и storage URL в browser DevTools.

Если metadata есть, а blob отсутствует, проблема вероятнее в ingest/consumer/storage write. Если blob существует, но API получает 403/404/5xx, проверить read permissions, URL, proxy, CORS и сетевую доступность.

### Шаг 5. Исправить конфигурацию и проверить end-to-end

Настроить backend в Helm values и secret references, включить persistence для filesystem либо настроить object storage. Затем применить изменение через Helm/GitOps:

```bash
helm lint infrastructure/sentry \
  -f infrastructure/sentry/prod-values/env-prod.yaml

helm template sentry infrastructure/sentry \
  -n sentry \
  -f infrastructure/sentry/prod-values/env-prod.yaml \
  > /tmp/sentry.yaml
```

После deployment отправить тестовую сессию с включённым Replay SDK, дождаться обработки и проверить воспроизведение в UI.

### Профилактика

- Явно задавать replay storage backend и credentials через Secret.
- Включить persistence или использовать отказоустойчивое object storage.
- Мониторить ошибки replay ingest / consumer lag / storage errors.
- Проверять permissions, TLS, DNS, CORS и доступность URL из браузера.
- Добавить end-to-end smoke test, который создаёт и воспроизводит тестовый replay.

---

## Чек-лист после инцидента

- [ ] Первопричина установлена и зафиксирована.
- [ ] Данные не удалялись без backup / snapshot.
- [ ] Все affected pods / PVC / nodes в штатном состоянии.
- [ ] Kafka lag и ошибки обработки вернулись к норме.
- [ ] Sentry принимает новые события.
- [ ] Session Replay проверен end-to-end (если затронут).
- [ ] Добавлены мониторинг, алерт или regression test.
- [ ] Зафиксированы corrective actions и ответственные.
