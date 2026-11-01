# Helm-чарт По шагам

Чарт создаёт Deployment API с фронтендом, Deployment worker, Service API
и один HTTPRoute для доступа к приложению по HTTPS через существующий Envoy Gateway.
HTTP-перенаправления нет. Nginx Ingress и отдельный балансировщик не нужны.

## Внешние зависимости

- PostgreSQL: Secret из `secrets.database` с ключом `DATABASE_URL`.
- Yandex Cloud: Secret из `secrets.yandex` с ключами `YANDEX_CLOUD_API_KEY`
  и `YANDEX_CLOUD_FOLDER` (только worker).
- Kafka с заранее созданным топиком: `kafka.bootstrapServers`, `kafka.topic`.
- `imagePullSecrets` для закрытых образов GHCR.
- Gateway `default/eg` с listener `demo-https` для
  `demo.kubepractice.ru`, допускающими маршруты из namespace `demo`.
  TLS завершается на Gateway: сертификат хранится в `default/gateway-tls`.
- AppProject Argo CD должен разрешать `gateway.networking.k8s.io/HTTPRoute`.

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
