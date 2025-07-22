# **Основные концепции сетей в Docker и Kubernetes**

Сети в Docker и Kubernetes обеспечивают связь между контейнерами, сервисами и внешним миром. Разберём ключевые модели, драйверы и принципы работы.

---

## **1. Сети в Docker**

### **1.1. Основные драйверы сетей**
Docker поддерживает несколько драйверов сетей, каждый решает разные задачи:

| **Драйвер**       | **Описание**                                                                 | **Когда использовать**                     |
|--------------------|-----------------------------------------------------------------------------|--------------------------------------------|
| **Bridge** (default) | Виртуальная сеть между контейнерами на одном хосте. Контейнеры получают IP из подсети Docker (`172.17.0.0/16`). | Локальная разработка, изолированные контейнеры. |
| **Host**           | Контейнер использует сетевой стек хоста (нет изоляции). Порт контейнера = порт хоста. | Когда нужна максимальная производительность. |
| **Overlay**        | Объединяет несколько Docker-хостов в единую сеть (для Swarm/Kubernetes). | Распределённые приложения в кластере. |
| **Macvlan**        | Контейнеру назначается реальный MAC-адрес, он становится частью физической сети. | Когда контейнер должен быть "как физический сервер". |
| **None**           | Полная изоляция сети (контейнер без сети).                                  | Для специальных сценариев (например, только локальный IPC). |

---

### **1.2. Примеры работы с сетями в Docker**
#### **Создание bridge-сети**
```bash
docker network create my-bridge-network
docker run -d --name nginx --network my-bridge-network nginx
```

#### **Проверка сетей**
```bash
docker network ls
docker inspect nginx | grep IPAddress
```

#### **Подключение контейнера к двум сетям**
```bash
docker network create backend
docker run -d --name app --network my-bridge-network --network backend my-app
```

---

### **1.3. Проброс портов**
Чтобы связать порт контейнера с портом хоста:
```bash
docker run -d -p 8080:80 nginx  # Хост:8080 → контейнер:80
```

---

## **2. Сети в Kubernetes (k8s)**

Kubernetes использует более сложную модель сетей, где:
- **Каждый под имеет уникальный IP** (из диапазона кластера).
- **Поды могут общаться друг с другом без NAT** (даже на разных нодах).
- **Сервисы (Services)** обеспечивают стабильные точки входа для подов.

---

### **2.1. Ключевые концепции**
#### **a) Pod Network**
- Каждый под получает IP из общей сети кластера (например, `10.244.0.0/16`).
- Реализуется через **CNI-плагины** (Calico, Flannel, Cilium, WeaveNet).

#### **b) Service Network**
- **Сервис** — это абстракция над группой подов.
- Типы сервисов:
  - **ClusterIP** (доступ только внутри кластера).
  - **NodePort** (проброс порта на все ноды).
  - **LoadBalancer** (внешний балансировщик в облаке).
  - **ExternalName** (CNAME-запись для внешнего сервиса).

#### **c) Ingress**
- Маршрутизация HTTP/HTTPS трафика (например, `example.com → сервис`).
- Работает с **Ingress-контроллерами** (Nginx, Traefik, Istio).

---

### **2.2. Примеры работы с сетями в k8s**
#### **Создание пода и сервиса**
```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

#### **Проверка сети**
```bash
kubectl get pods -o wide      # Узнать IP пода
kubectl get svc              # Узнать ClusterIP сервиса
kubectl exec -it nginx -- curl http://nginx-service  # Доступ к сервису из пода
```

---

### **2.3. CNI-плагины (Container Network Interface)**
Плагины, которые настраивают сеть для подов. Популярные:
- **Calico** — поддерживает политики безопасности (NetworkPolicies).
- **Flannel** — простой и быстрый, использует overlay-сеть.
- **Cilium** — работает на основе eBPF, поддерживает L7-фильтрацию.

#### **Установка Calico**
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

### **2.4. Network Policies**
Ограничивают доступ между подами (например, "запретить всем, кроме фронтенда, доступ к БД").
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 5432
```

---

## **3. Сравнение Docker и Kubernetes сетей**
| **Аспект**         | **Docker**                          | **Kubernetes**                     |
|---------------------|-------------------------------------|------------------------------------|
| **Базовая единица** | Контейнер                          | Pod (1+ контейнер)                |
| **Сеть по умолчанию** | `bridge`                          | Зависит от CNI (Calico/Flannel)   |
| **Service Discovery** | Docker DNS (по имени контейнера) | kube-dns/CoreDNS (по имени сервиса) |
| **Масштабирование**  | Ручное (для кластера — Swarm)     | Автоматическое (встроено в k8s)   |
| **Балансировка**    | Через `-p` (проброс портов)       | Service (ClusterIP/LoadBalancer)  |

---

## **4. Типичные проблемы и диагностика**
### **Docker**
- **Контейнеры не видят друг друга** → Проверить, что они в одной сети (`docker network inspect`).
- **Порт недоступен снаружи** → Проверить `-p` и фаервол (`iptables -L`).

### **Kubernetes**
- **Под не получает IP** → Проверить CNI-плагин (`kubectl get pods -n kube-system`).
- **Сервис не отвечает** → Проверить `selector` в сервисе и лейблы пода (`kubectl describe svc`).
- **Ingress не работает** → Проверить Ingress-контроллер (`kubectl get ingress`).

---

## **5. Полезные команды**
### **Docker**
```bash
docker network ls
docker inspect <container> | grep IPAddress
docker logs <container>
```

### **Kubernetes**
```bash
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod>
kubectl get svc
kubectl get networkpolicies
```

---

## **Вывод**
- **Docker** предлагает простые сети для локального развёртывания (bridge, host).
- **Kubernetes** использует сложные модели (CNI, Services, Ingress) для кластеров.
- **Общее правило**: Если контейнеры/поды не общаются — проверьте:
  1. Сетевые политики.
  2. DNS (работает ли разрешение имён?).
  3. Firewall/iptables.
