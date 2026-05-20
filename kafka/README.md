# Kafka (single node, KRaft) для k3d / локального кластера

Один процесс `broker + controller`, без ZooKeeper, без репликации партиций — только для **dev / smoke**.

## Предусловия

- k3d (или другой Kubernetes) со **storage class** `local-path` (как в k3d по умолчанию). Если другой класс — поменяйте `storageClassName` в `kafka-pvc.yaml`.

## Установка

```bash
kubectl apply -f kafka-pvc.yaml
kubectl apply -f kafka-deployment.yaml
kubectl apply -f kafka-service.yaml
kubectl apply -f kafka-ui-deployment.yaml
kubectl apply -f kafka-ui-service.yaml
```

Проверка:

```bash
kubectl get pods -l app=kafka
kubectl get pods -l app=kafka-ui
kubectl logs -l app=kafka --tail=50
```

## Kafka UI

Веб-интерфейс [provectuslabs/kafka-ui](https://github.com/provectus/kafka-ui) к брокеру `kafka:9092`.

Доступ с машины (port-forward):

```bash
kubectl port-forward svc/kafka-ui 8080:8080
```

Открыть: http://localhost:8080

UI должен быть в **том же namespace**, что и Kafka (иначе bootstrap `kafka:9092` не резолвится).

## Важно

- Это **не** продакшен: один под, при потере ноды или PV возможна потеря данных.
- Для прода обычно используют Strimzi / оператор или managed Kafka.

## Подключение из кластера

- **Тот же namespace, что и деплой Kafka:** `kafka:9092`
- **Другой namespace:** `kafka.<namespace-где-kafka>.svc.cluster.local:9092` (namespace подставляется в `KAFKA_BOOTSTRAP_SERVERS` у клиента, не в манифест Kafka)

Брокер в metadata отдаёт `kafka.<свой-namespace>.svc.cluster.local:9092` (через `POD_NAMESPACE` в Deployment). Namespace в yaml Kafka **не зашит**.

**Protocol:** PLAINTEXT (без TLS и SASL)

## Данные

Том: PVC `kafka-data`, каталог в контейнере `/var/lib/kafka/data` (`KAFKA_LOG_DIRS`).

Если раньше пробовали Bitnami-образ — **удалите старый PVC** перед первым запуском `apache/kafka` (формат данных другой):

```bash
kubectl delete pvc kafka-data --ignore-not-found
kubectl apply -f kafka-pvc.yaml
```

## Образы

- Kafka: **`docker.io/apache/kafka:3.9.0`** — публичный официальный образ ([Docker Hub](https://hub.docker.com/r/apache/kafka))
- Kafka UI: `provectuslabs/kafka-ui:v0.7.2`

### Почему не bitnami/kafka:4.2

Тег **`docker.io/bitnami/kafka:4.2` в публичном Docker Hub отсутствует**: Bitnami перевёл образы в [Bitnami Secure Images](https://github.com/bitnami/containers/blob/main/bitnami/kafka/README.md) (нужна подписка). Старые Debian-сборки — в **`bitnamilegacy/kafka`** (например `3.9.0`), если захотите вернуться к Bitnami env-переменным.
