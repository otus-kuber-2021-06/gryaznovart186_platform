# gryaznovart186_platform
gryaznovart186 Platform repository

## Homeworks
### HW 1 kubernetes-intro
0. За работой coredns следит `Replication Controller` компонента `kube-controller-manager`. За работой `api-server` следит `kubelet` на мастер ноде k8s
1. Для запуска nginx под определенным пользователем необходимо в манифесте пода добавить в `securityContext` строку runAsUser: 1001 и примонтировать дириктории `/var/log/nginx` `/var/cache/nginx` `/run` как emptyDir, что бы были права на запись у пользователя 1001
Для того что бы nginx отдавал страницы из директории /app и слушал 8000 порт необходимо в Dockerfile копировать конфиг:
```cosole
server {
    listen       8000;
    server_name  _;

    access_log  /var/log/nginx/access.log  main;

    location / {
        root   /app;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
в котором `root`  задан как  `/app;` и указать `WORKDIR` в Dockerfile.

2. Причина по которой не запускается фронтенд hipster shop заключается в том что не хватает переменных окружения:
    * PRODUCT_CATALOG_SERVICE_ADDR
    * CURRENCY_SERVICE_ADDR
    * CART_SERVICE_ADDR
    * RECOMMENDATION_SERVICE_ADDR
    * SHIPPING_SERVICE_ADDR
    * CHECKOUT_SERVICE_ADDR
    * AD_SERVICE_ADDR

После добавления данных переменых окружения в манифест запуска пода фронтенд приложение запускается без проблем запускается без проблем.


### HW 2 kubernetes-controlers
0. Контролер `ReplicaSet` отслеживает только количество запущеных подов и не следит за изменениями в темплейте развертывания. Что бы измененния в темплейте запускали новык версии подов, необходимо использовать например `Deployment`
1. Для реализации аналога blue-green развертывания, необходимо в спецификацию деплоймента  добавить rollingUpdate опцию с параметрами maxSurge: 100% и maxUnavailable: 0%, что позволит сразу запустить все новые поды, и помешать одновремено с этим начать удалять старые
```yaml
spec:
  minReadySeconds: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 100%
      maxUnavailable: 0%
```
2. Для реализации Reverse Rolling Update необходимо прописать в  maxSurge: 0 и maxUnavailable: 1, что не даст запускать больше подов чем прописано в деплойменте, но при этом разрешит 1 рабочий под начать удалять
```yaml
spec:
  minReadySeconds: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
```
По пунктам 1 и 2 написаны манифесты `paymentservice-deployment-bg.yaml` и `paymentservice-deployment-reverse.yaml`

3. Что бы DaemonSet'ы запускались на всех нодах необходимо в манифест добавить секцию с `tolerations` которая обойдет какие либо запреты на размещения подов `taints`
```yaml
spec:
  tolerations:
  - key: node-role.kubernetes.io/master
    operator: Exists
    effect: NoSchedule
  containers:
```
### HW 3 kubernetes-security
Создание неймспейса:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace_name>

```
Cоздание сервис аккаунта:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <sa_name>
  namespace: <ns(optional)>

```
Создание кластерной роли:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: view-all-pods
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```
Создание кластероного роль биндинга:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: carol-pod-viewer
subjects:
- kind: ServiceAccount
  name: carol
  namespace: prometheus
roleRef:
  kind: ClusterRole
  name: view-all-pods
  apiGroup: rbac.authorization.k8s.io
```
Создание роли:
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mynamespace-user-full-access
  namespace: mynamespace
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]
```
Создание биндинга роли:
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: mynamespace-user-view
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: mynamespace-user
  namespace: mynamespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: mynamespace-user-full-access
```

### HW 4 kubernetes-networks
Основыное задание сделано по методичке
Задание со звездочкой
1) Необходимо создать 2 сервиса для tcp и udp с одним задданым ип и разрешить шаринг ип через анатации металлб
```yaml
apiVersion: v1
kind: Service
metadata:
  name: core-dns-udp
  namespace: kube-system
  annotations:
      metallb.universe.tf/allow-shared-ip: kube-dns
spec:
  selector:
    k8s-app: kube-dns
  type: LoadBalancer
  ports:
  - port: 53
    protocol: UDP
    targetPort: 53
  loadBalancerIP: 172.17.255.10
---
apiVersion: v1
kind: Service
metadata:
  name: core-dns-tcp
  namespace: kube-system
  annotations:
      metallb.universe.tf/allow-shared-ip: kube-dns
spec:
  selector:
    k8s-app: kube-dns
  type: LoadBalancer
  ports:
  - port: 53
    protocol: TCP
    targetPort: 53
  loadBalancerIP: 172.17.255.10

```
2) Дашборд кубернетеса, разворачивается стандартным инресом с дополнительной анатацией `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kube-dashboard
  namespace: kubernetes-dashboard
  labels:
    name: kube-dashboard
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: "/dashboard"
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
```
3) Для канаречного релиза необходимо создать копию манифестов для разворачивания новой версии приложения и в инресс добавить несколько анатаций, также ингресс должен слушать конкретный хост
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "50"
spec:
  rules:
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: canary
            port:
              number: 8000
    host: canary.example
```
