# Helm-чарт По шагам

Чарт создаёт Deployment API с фронтендом, Deployment worker, Kafka, Service API
и один HTTPRoute для доступа к приложению по HTTPS через существующий Envoy Gateway.
HTTP-перенаправления нет. Nginx Ingress и отдельный балансировщик не нужны.

## Внешние зависимости

- PostgreSQL: Secret из `secrets.database` с ключом `DATABASE_URL`.
- Yandex Cloud: Secret из `secrets.yandex` с ключами `YANDEX_CLOUD_API_KEY`
  и `YANDEX_CLOUD_FOLDER` (только worker).
- Внешняя Kafka нужна только при `kafka.enabled: false`.
- `imagePullSecrets` для закрытых образов GHCR.
- Gateway `default/eg` с listener `demo-https` для
  `demo.kubepractice.ru`, допускающими маршруты из namespace `demo`.
  TLS завершается на Gateway: сертификат хранится в `default/gateway-tls`.
- AppProject Argo CD должен разрешать HTTPRoute, StatefulSet и Job.

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
## Kafka в составе проекта

По умолчанию `kafka.enabled: true`: запускается один брокер Apache Kafka 3.9.1
в режиме KRaft (без ZooKeeper). StatefulSet использует временный диск `emptyDir`
размером до `kafka.storageSizeLimit` (по умолчанию 1 GiB). При перезапуске контейнера
в том же Pod данные сохраняются, при пересоздании Pod сообщения и consumer offsets
теряются. Это согласованная конфигурация учебного стенда; задачи остаются в PostgreSQL.

В namespace demo для релиза demo-app адрес брокера — `demo-app-kafka:9092`.
API и worker получают его автоматически. Брокер доступен внутри кластера
через ClusterIP, внешний порт не открывается. Аутентификация и TLS для Kafka
в этом учебном варианте не включены.

Порядок синхронизации Argo CD: сервисы и Kafka (wave -2), создание топика
`kafka.topic` с одной партицией и одной репликой (Sync hook, wave -1), затем
обновление API и worker. Создание топика идемпотентно; Job удаляется после успеха.
Автосоздание топиков включено также для восстановления после пересоздания Pod.

Для внешнего брокера задайте `kafka.enabled: false` и `kafka.bootstrapServers`.
В этом режиме брокер и Job не создаются; топик готовится на внешней стороне.
