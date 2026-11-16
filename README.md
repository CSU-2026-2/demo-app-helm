# Helm-чарт По шагам

Чарт создаёт Deployment API с фронтендом, Deployment worker, Service API
и один HTTPRoute для доступа к приложению по HTTPS через существующий Envoy Gateway.
HTTP-перенаправления нет. Nginx Ingress и отдельный балансировщик не нужны.

## Внешние зависимости

- PostgreSQL: Secret из `secrets.database` с ключом `DATABASE_URL`.
- Вход: Secret из `secrets.auth` с ключами `AUTH_USERNAME`, `AUTH_PASSWORD`,
  `AUTH_SESSION_SECRET` (случайная строка от 32 символов, передаётся как `JWT_SECRET_KEY`).
  Передаётся только API. Логин и пароль используются для создания первого пользователя
  и переноса старых задач; последующие пользователи регистрируются через интерфейс,
  хеши паролей и сессии JWT хранятся в PostgreSQL.
  API выполняет Alembic `upgrade head` перед запуском; worker обновляется после API.
- Yandex Cloud: Secret из `secrets.yandex` с ключами `YANDEX_CLOUD_API_KEY`
  и `YANDEX_CLOUD_FOLDER` (только worker).
- Общая Kafka: `kafka.bootstrapServers` и `kafka.topic`.
- `imagePullSecrets` для закрытых образов GHCR.
- Gateway `default/eg` с listener `demo-https` для
  `demo.kubepractice.ru`, допускающими маршруты из namespace `demo`.
  TLS завершается на Gateway: сертификат хранится в `default/gateway-tls`.
- AppProject Argo CD должен разрешать Deployment, ReplicaSet, Pod, Service и HTTPRoute.

Gateway и AppProject настраиваются администратором отдельно от приложения.
В локальном инфраструктурном каталоге `../kubepractice` декларации находятся
в `envoy-gateway/https.yaml` и `argocd/demo-app-project.yaml`.

## Настройки маршрута

`httpRoute.hostname` задаёт домен. `httpRoute.gateway` содержит имя, namespace
и имя HTTPS listener. Домен должен соответствовать listener и сертификату.
`httpRoute.enabled: false` отключает маршрут.

Интерфейс доступен по https://demo.kubepractice.ru/ через существующий Service.
Отдельного образа, TLS-секрета в namespace demo или домена для фронтенда нет.

## Проверка

```sh
helm lint .
helm template demo-app . -n demo
```

Адреса и теги образов обновляются workflows приложения.
## Общая Kafka

Kafka развёрнута отдельно как Application `kafka-cluster` в Argo CD, в namespace `kafka`.
Strimzi Operator 1.3.1 управляет тремя узлами Kafka 4.3.2 controller/broker.
Манифесты кластера: `infrastructure/kafka/`; bootstrap Applications оператора и Kafbat:
`../kubepractice/kafka/`.
В этом чарте нет брокера, диска или Job создания топика.

API и worker используют `shared-kafka-bootstrap.kafka.svc.cluster.local:9092`, топик `todos-created`.
Старый DNS `kafka.kafka.svc.cluster.local:9092` сохранён как алиас.
Адрес доступен из разных namespace Kubernetes. Внешнего порта нет.
В учебной конфигурации включено автосоздание топиков, аутентификация и TLS отключены.
Для других приложений используйте собственные имена топиков и consumer groups.

Каждый брокер размещён на отдельной ноде с постоянным локальным томом 10Gi.
Топик имеет три реплики и `min.insync.replicas=2`. UI: https://kafka.kubepractice.ru.
Задачи и шаги приложения остаются во внешней PostgreSQL.
