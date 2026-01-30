# Kafka (Strimzi) ArgoCD Application (k3s)

Конфигурация для развертывания Apache Kafka через Strimzi Operator и Kafka Cluster в **k3s** с ArgoCD. Стиль развертывания аналогичен MinIO: Operator (Helm) + Cluster (манифесты из Git).

<details>
<summary><strong>🚀 Быстрый старт</strong></summary>

---

**Минимальные шаги для развертывания Kafka:**

1. **StorageClass:** в k3s по умолчанию есть `local-path`. Проверка: `kubectl get storageclass`.

2. **Примените ArgoCD Application для Strimzi Operator:**
   ```bash
   kubectl apply -f argocd-apps/kafka/operator/application.yaml
   ```

3. **Дождитесь готовности Operator (1–2 минуты):**
   ```bash
   kubectl get pods -n strimzi -w
   # Под strimzi-cluster-operator должен быть в состоянии Running
   ```

4. **Создайте Kafka Cluster (через ArgoCD Application):**
   ```bash
   kubectl apply -f argocd-apps/kafka/cluster/application.yaml
   ```

5. **Дождитесь готовности Kafka (3–5 минут):**
   ```bash
   kubectl get pods -n kafka -w
   # Поды my-cluster-kafka, my-cluster-zookeeper, entity-operator должны быть Running
   ```

6. **Подключение к Kafka (внутри кластера):**
   - **Bootstrap (plain):** `my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092`
   - **Bootstrap (TLS):** `my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9093`

📋 **Детальные инструкции:** см. секции ниже

</details>

<details>
<summary><strong>📋 Описание и компоненты</strong></summary>

---

Apache Kafka — распределённая платформа для потоковой обработки событий. Strimzi Operator управляет жизненным циклом Kafka в Kubernetes через CRD.

### Архитектура развертывания

```mermaid
graph TB
    subgraph ArgoCD["ArgoCD"]
        ArgoCD_Op["ArgoCD Application<br/>kafka-operator"]
        ArgoCD_Cluster["ArgoCD Application<br/>kafka-cluster"]
    end

    subgraph Strimzi_NS["Namespace strimzi"]
        Operator["Strimzi Cluster Operator<br/>CRD Controller"]
    end

    subgraph Kafka_NS["Namespace kafka"]
        ZK["ZooKeeper<br/>1 replica"]
        Kafka["Kafka Brokers<br/>1 replica"]
        EO["Entity Operator<br/>Topic + User"]
    end

    subgraph Infrastructure["Infrastructure"]
        Storage["StorageClass<br/>local-path"]
    end

    ArgoCD_Op --> Operator
    ArgoCD_Cluster --> ZK
    ArgoCD_Cluster --> Kafka
    ArgoCD_Cluster --> EO
    Operator --> ZK
    Operator --> Kafka
    Operator --> EO
    Kafka --> Storage
    ZK --> Storage
```

### Компоненты

- **Strimzi Cluster Operator**: Управляет Kafka, ZooKeeper, KafkaTopic, KafkaUser через CRD
  - Развертывается через Helm chart: `https://strimzi.io/charts`
  - Версия chart: 0.49.1
  - Namespace: `strimzi`
  - Следит за namespace: `kafka`

- **Kafka Cluster (my-cluster)**: Одноузловой кластер для dev/test
  - ZooKeeper: 1 реплика, 5Gi PVC
  - Kafka: 1 реплика, 10Gi PVC, listeners plain:9092 и tls:9093
  - Entity Operator: Topic Operator + User Operator

- **Доступ:**
  - Внутри кластера: `my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092` (plain), `:9093` (tls)

</details>

<details>
<summary><strong>📋 Структура файлов</strong></summary>

---

```
kafka/
├── operator/
│   └── application.yaml   # ArgoCD Application для Strimzi Operator (Helm chart)
├── cluster/
│   ├── application.yaml  # ArgoCD Application для Kafka Cluster (указывает на Git)
│   └── kafka.yaml        # Strimzi Kafka CRD (ZooKeeper + Kafka + Entity Operator)
└── README.md             # Этот файл
```

**Пояснение:**

- **`operator/application.yaml`**: ArgoCD Application для Strimzi Operator через Helm. Создаёт namespace `strimzi`, устанавливает CRD и Operator. Operator настроен на наблюдение за namespace `kafka`.

- **`cluster/application.yaml`**: ArgoCD Application, источник — Git, путь `argocd-apps/kafka/cluster`. Sync-wave: "1" (после operator). Назначение — namespace `kafka`.

- **`cluster/kafka.yaml`**: Custom Resource `Kafka` в namespace `kafka`. Описывает ZooKeeper, брокеры Kafka (1 реплика), Entity Operator, storage и ресурсы.

</details>

<details>
<summary><strong>📋 Предварительные требования</strong></summary>

---

1. **Kubernetes 1.27+** (для Strimzi 0.49)
2. **ArgoCD** установлен и настроен
3. **StorageClass** (например, `local-path` в k3s)
4. Репозиторий **k3s-test** добавлен в ArgoCD (или применяете манифесты из локального Git)

</details>

<details>
<summary><strong>📋 Создание топиков и пользователей</strong></summary>

---

Топики и пользователи можно создавать через Strimzi CRD в том же namespace `kafka` (применять вручную или через Git):

**Пример топика (KafkaTopic):**
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: my-topic
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  partitions: 3
  replicas: 1
```

**Пример пользователя (KafkaUser):**
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: my-user
  namespace: kafka
  labels:
    strimzi.io/cluster: my-cluster
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
      - resource:
          type: topic
          name: my-topic
        operations:
          - Read
          - Write
```

После создания KafkaUser в кластере появится Secret с credentials (например, `my-user`).

</details>
