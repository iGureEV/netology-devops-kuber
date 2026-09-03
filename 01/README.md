# Краткая инструкция по установке MicroK8s и настройке Dashboard

## 1. Установка MicroK8s (чистая, с удалением старых данных)

```bash
# Полное удаление предыдущей версии (если была)
sudo snap remove microk8s --purge
sudo rm -rf /var/snap/microk8s

# Установка стабильной версии (рекомендуется 1.34)
sudo snap install microk8s --classic --channel=1.34/stable

# Добавление пользователя в группу microk8s
sudo usermod -a -G microk8s $USER
# Применить группу (выйти и зайти или выполнить newgrp)
newgrp microk8s
```

## 2. Запуск и проверка кластера

```bash
microk8s start
microk8s status --wait-ready
microk8s kubectl get nodes
microk8s kubectl get pods --all-namespaces
```

## 3. Настройка `kubectl` для удобства

```bash
mkdir -p ~/.kube
microk8s config > ~/.kube/config
chmod 600 ~/.kube/config
# Теперь можно использовать kubectl без префикса microk8s
kubectl get nodes
```

## 4. Включение полезных аддонов

```bash
microk8s enable dashboard
microk8s enable ingress
microk8s enable registry          # локальный реестр на порту 32000
microk8s enable hostpath-storage  # хранилище для PVC
```

## 5. Доступ к Kubernetes Dashboard

```bash
# Запустите port-forward (в отдельном терминале или фоне)
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```
Откройте в браузере: **https://localhost:8443** (примите предупреждение о сертификате).

### Получение токена для входа

Используйте уже созданный токен (если включили dashboard):
```bash
kubectl -n kube-system describe secret microk8s-dashboard-token | grep '^token:'
```

Если токен не подходит, создайте нового администратора:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

# Получить токен
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}') | grep '^token:'
```
Скопируйте токен и вставьте в форму входа.

---

## 6. Если возникла ошибка containerd "no unpack platforms defined"

**Причина:** неверная конфигурация containerd.  
**Решение (делать до установки аддонов):**

Остановите MicroK8s:
```bash
microk8s stop
```
Отредактируйте `/var/snap/microk8s/current/args/containerd-template.toml` (sudo nano) и приведите секцию `[plugins."io.containerd.grpc.v1.cri".containerd]` к виду:
```toml
[plugins."io.containerd.grpc.v1.cri".containerd]
  snapshotter = "native"
  default_runtime_name = "runc"
  no_pivot = false

  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true
```
Очистите кеш образов:
```bash
sudo rm -rf /var/snap/microk8s/common/var/lib/containerd
```
Запустите MicroK8s снова:
```bash
microk8s start
```

---

## 7. Проверка работоспособности

```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

Все системные поды должны быть в статусе **Running**.  
Доступ к Dashboard через `https://localhost:8443` с полученным токеном.

---

**Готово!** Кластер готов к развертыванию приложений.
