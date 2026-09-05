# Домашнее задание к занятию «Kubernetes. Причины появления. Команда kubectl»

### Цель задания

Для экспериментов и валидации ваших решений вам нужно подготовить тестовую среду для работы с Kubernetes. Оптимальное решение — развернуть на рабочей машине или на отдельной виртуальной машине MicroK8S.

------

### Чеклист готовности к домашнему заданию

1. Личный компьютер с ОС Linux или MacOS 

или

2. ВМ c ОС Linux в облаке либо ВМ на локальной машине для установки MicroK8S  

------

### Инструкция к заданию

1. Установка MicroK8S:
    - sudo apt update,
    - sudo apt install snapd,
    - sudo snap install microk8s --classic,
    - добавить локального пользователя в группу `sudo usermod -a -G microk8s $USER`,
    - изменить права на папку с конфигурацией `sudo chown -f -R $USER ~/.kube`.

2. Полезные команды:
    - проверить статус `microk8s status --wait-ready`;
    - подключиться к microK8s и получить информацию можно через команду `microk8s command`, например, `microk8s kubectl get nodes`;
    - включить addon можно через команду `microk8s enable`; 
    - список addon `microk8s status`;
    - вывод конфигурации `microk8s config`;
    - проброс порта для подключения локально `microk8s kubectl port-forward -n kube-system service/kubernetes-dashboard 10443:443`.

3. Настройка внешнего подключения:
    - отредактировать файл /var/snap/microk8s/current/certs/csr.conf.template
    ```shell
    # [ alt_names ]
    # Add
    # IP.4 = 123.45.67.89
    ```
    - обновить сертификаты `sudo microk8s refresh-certs --cert front-proxy-client.crt`.

4. Установка kubectl:
    - curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl;
    - chmod +x ./kubectl;
    - sudo mv ./kubectl /usr/local/bin/kubectl;
    - настройка автодополнения в текущую сессию `bash source <(kubectl completion bash)`;
    - добавление автодополнения в командную оболочку bash `echo "source <(kubectl completion bash)" >> ~/.bashrc`.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Инструкция](https://microk8s.io/docs/getting-started) по установке MicroK8S.
2. [Инструкция](https://kubernetes.io/ru/docs/reference/kubectl/cheatsheet/#bash) по установке автодополнения **kubectl**.
3. [Шпаргалка](https://kubernetes.io/ru/docs/reference/kubectl/cheatsheet/) по **kubectl**.

------

### Задание 1. Установка MicroK8S

1. Установить MicroK8S на локальную машину или на удалённую виртуальную машину.
2. Установить dashboard.
3. Сгенерировать сертификат для подключения к внешнему ip-адресу.

------

### Задание 2. Установка и настройка локального kubectl
1. Установить на локальную машину kubectl.
2. Настроить локально подключение к кластеру.
3. Подключиться к дашборду с помощью port-forward.

------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода команд `kubectl get nodes` и скриншот дашборда.

---
---

# Краткая руководство по установке MicroK8s и настройке Dashboard

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
