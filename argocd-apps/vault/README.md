# Vault на K3s (ArgoCD Application)

Развёртывание HashiCorp Vault через ArgoCD в кластере **K3s**.

Домен и доступ: **vault.lab.local** (окружение lab.local, Traefik).

## Использование

1. **Подключить Vault:** применить ArgoCD Application из этого каталога:
   ```bash
   kubectl apply -f argocd-apps/vault/application.yaml
   ```
2. Перед синхом выполнить **preconfigure** (сертификаты) — см. раздел «Сертификаты» ниже.
3. В `/etc/hosts` (или DNS) должна быть запись для **vault.lab.local** (IP ноды или Ingress).

## Отличия от варианта для Kubernetes

- **Ingress:** Traefik (встроенный в K3s), не nginx.
- **StorageClass:** явно задан `local-path` (стандартный provisioner K3s).
- **PSP:** отключены (в K3s используют Pod Security Standards).
- **Чарт:** vault-1.14.0, пути в репо — `argocd-apps/vault/`.

## Требования

- **K3s** с встроенным Traefik (Ingress).
- **cert-manager** и ClusterIssuer `selfsigned-issuer` (TLS для Ingress).
- **preconfigure** для Vault server и injector — см. раздел «Сертификаты» ниже.

## Сертификаты

- **Ingress:** cert-manager по аннотации создаёт Secret `vault-ingress-tls`. Отдельный Certificate не нужен.
- **Vault server и injector:** preconfigure — в `preconfigure/<PROJECT>/` лежат манифесты `vault-ca` (ConfigMap), `vault-crt`, `vault-injector-crt` (Secret). Чарт монтирует их в под.

Перед деплоем: namespace и preconfigure (сертификаты Vault/injector):

```bash
kubectl create namespace vault --dry-run=client -o yaml | kubectl apply -f -
cd argocd-apps/vault/preconfigure/scripts
./generate-vault-certs.sh lab.local vault vault
kubectl apply -f argocd-apps/vault/preconfigure/lab.local/
```

### Схема

| Компонент        | Источник              |
|------------------|-----------------------|
| Ingress (Traefik)| cert-manager → `vault-ingress-tls` |
| Vault server     | preconfigure → `vault-ca` (ConfigMap) + `vault-crt` (Secret) |
| Injector         | preconfigure → `vault-injector-crt` + caBundle |

## Установка и удаление через Helm

Развёртывание без ArgoCD: с хоста, где есть `kubectl` и репозиторий. Предварительно выполнить шаги из раздела «Сертификаты».

**Установка или обновление (из корня репо):**

```bash
helm upgrade --install vault argocd-apps/vault/helm/charts/vault-1.14.0 \
  -n vault --create-namespace \
  -f argocd-apps/vault/helm/custom-values/lab.local.yaml
```

Или из каталога `argocd-apps/vault`:

```bash
cd argocd-apps/vault
helm upgrade --install vault helm/charts/vault-1.14.0 \
  -n vault --create-namespace \
  -f helm/custom-values/lab.local.yaml
```

**Удаление:**

```bash
helm uninstall vault -n vault
kubectl delete pvc -n vault -l app.kubernetes.io/name=vault   # при необходимости
```

При values `lab.local.yaml` (fullnameOverride: vault) под называется **vault-0**.

---

## Порядок развёртывания (ArgoCD)

1. **Namespace и preconfigure** — команды из раздела «Сертификаты» выше.

2. **ServersTransport для Traefik** (чтобы Traefik подключался к HTTPS-бэкенду Vault без проверки сертификата):

   ```bash
   kubectl apply -f argocd-apps/vault/serverstransport.yaml
   ```

3. **ArgoCD Application:**

   ```bash
   kubectl apply -f argocd-apps/vault/application.yaml
   ```

   Либо установить через **Helm** — см. выше.

4. **Проверка:**
   ```bash
   kubectl get pods -n vault
   kubectl get pvc -n vault
   kubectl get ingress -n vault
   ```
   Должен появиться PVC `data-vault-0` (StorageClass `local-path`).

5. **Доступ:** https://vault.lab.local (hosts/DNS при необходимости).

6. **Инициализация и unseal** — см. раздел ниже.

---

## Инициализация Vault (init и unseal)

После деплоя Vault в состоянии **sealed**. Один раз выполнить init и unseal.

**Предварительно:** под `vault-0` в namespace `vault` в состоянии Running.

### Инициализация (один раз)

```bash
kubectl exec -it vault-0 -n vault -- vault operator init
```

Или с одним unseal-ключом (удобно для тестов):

```bash
kubectl exec -it vault-0 -n vault -- vault operator init -key-shares=1 -key-threshold=1
```

Сохраните unseal-ключи и root token.

### Распечатывание (unseal)

```bash
kubectl exec -it vault-0 -n vault -- vault operator unseal <Unseal Key 1>
# … при необходимости ещё ключи
kubectl exec -it vault-0 -n vault -- vault status
```

Через UI: https://vault.lab.local → Unseal → ввести ключи → войти по Root Token.

### Шпаргалка

| Действие       | Команда |
|----------------|--------|
| Init (1 ключ)   | `kubectl exec -it vault-0 -n vault -- vault operator init -key-shares=1 -key-threshold=1` |
| Unseal          | `kubectl exec -it vault-0 -n vault -- vault operator unseal <KEY>` |
| Статус         | `kubectl exec -it vault-0 -n vault -- vault status` |

---

## Структура каталога

```
argocd-apps/vault/
├── application.yaml              # ArgoCD Application (path: helm/charts/vault-1.14.0, valueFiles: ../../custom-values/lab.local.yaml)
├── serverstransport.yaml         # Traefik: insecureSkipVerify для HTTPS-бэкенда (применить вручную)
├── README.md                     # Документация (этот файл)
├── preconfigure/
│   ├── scripts/
│   │   ├── generate-vault-certs.sh
│   │   └── README.md
│   ├── keys/                     # в .gitignore
│   └── lab.local/                # манифесты: vault-ca, vault-crt, vault-injector-crt (или <PROJECT>/)
├── vault_backup_job/             # CronJob бэкапа Vault
│   ├── application.yaml
│   ├── custom-values/
│   └── charts/manual-apply-v1/
└── helm/
    ├── custom-values/
    │   └── lab.local.yaml        # Values для K3s / lab.local (Traefik, local-path, PSP off)
    └── charts/
        ├── vault-1.14.0/
        └── vault-1.14.0/        # используется в application.yaml
```
