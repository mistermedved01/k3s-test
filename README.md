# k3s-test

Автоматизированная установка k3s, Helm и ArgoCD для тестового окружения.

## Быстрый старт

```bash
git clone https://github.com/mistermedved01/k3s-test.git
cd k3s-test
sudo chmod +x install-all.sh
sudo bash install-all.sh
```

## Настройка DNS

Добавьте запись в `/etc/hosts` (замените IP на адрес вашей VM):

```
192.168.77.77 argocd.lab.local
```

## Доступ к ArgoCD

**Web UI:** https://argocd.lab.local:30443

**Логин:** `admin`

**Пароль:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Приложения

В репозитории доступны готовые ArgoCD Applications:

- **cert-manager** — автоматическое управление TLS сертификатами
- **MinIO** — S3-совместимое объектное хранилище (Operator + Tenant)
- **Kafka** — Apache Kafka через Strimzi (Operator + Cluster + веб-интерфейс Kafka UI)
- **Media Server Stack** — Jellyfin, Prowlarr, qBittorrent, Radarr

Подробная документация в директории `argocd-apps/`.

## Особенности k3s

- **Traefik** — встроенный Ingress-контроллер
- **local-path** — StorageClass по умолчанию для PVC
- **containerd** — контейнерный рантайм 
- Подходит для малых и тестовых кластеров