# Helm-чарт По шагам

Состав: Deployment API (также раздаёт фронтенд), Deployment worker, Service API
и опциональный Ingress. Дополнительные CRD не нужны.

Убраны Argo Rollouts, канареечные этапы, AnalysisTemplate, Istio Gateway,
VirtualService, DestinationRule, HPA, KEDA, PDB, outbox и отдельный ServiceAccount.
Для каждого компонента задаётся фиксированное число реплик. API использует
обычное rolling update Kubernetes. Селекторы API и worker сохранены раздельными.

## Внешние зависимости

- PostgreSQL: Secret из `secrets.database` с ключом `DATABASE_URL`.
- Yandex Cloud: Secret из `secrets.yandex` с ключами `YANDEX_CLOUD_API_KEY`
  и `YANDEX_CLOUD_FOLDER` (только worker).
- Kafka и заранее созданный топик: `kafka.bootstrapServers` и `kafka.topic`.
  Одинаковые настройки передаются API и worker.
- Ingress-контроллер и TLS Secret, если включён Ingress.
- `imagePullSecrets` для GHCR, если образы закрыты.

Интерфейс доступен на `/` через Service/Ingress API.
Отдельного образа или домена для фронтенда нет.

## Проверка и установка

```sh
helm lint .
helm template demo .
helm upgrade --install demo . -n demo --create-namespace
```

Адреса и теги образов в `values.yaml` обновляются workflows приложения.
Перед ручной установкой укажите уже опубликованные совместимые образы.

## Обновление предыдущей версии

Чарт 0.2.0 заменяет Rollout обычным Deployment и оставляет только Nginx Ingress.
Старый адрес Istio больше не обслуживается этим чартом.
При GitOps-обновлении требуется удаление ресурсов, убранных из шаблонов (prune).
Без него старые Rollout, HPA, ScaledObject и маршруты Istio останутся в кластере.
Сначала проверьте `helm template`/diff в целевом окружении. Изменение файлов
не применяет чарт и не удаляет работающие ресурсы.
Старые секции values для rollout, autoscaling, keda, istioIngress и outbox удалены;
передавайте актуальный values вместо `--reuse-values`.
