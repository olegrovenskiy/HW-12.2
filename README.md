# HW-12.2

## Домашнее задание к занятию "12.2 Команды для работы с Kubernetes"

### Задание 1: Запуск пода из образа в деплойменте

Создание deployment с 2-мя репликами

    root@mck-devops-tools:/etc/kubernetes/minikube# kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4 --replicas=2
    deployment.apps/hello-node created

Проверка deployment

    root@mck-devops-tools:/etc/kubernetes/minikube# kubectl get deployment
    NAME         READY   UP-TO-DATE   AVAILABLE   AGE
    hello-node   2/2     2            2           23s

Проверка pods

    root@mck-devops-tools:/etc/kubernetes/minikube# kubectl get pods
    NAME                          READY   STATUS    RESTARTS   AGE
    hello-node-6b89d599b9-4p2fl   1/1     Running   0          34s
    hello-node-6b89d599b9-q7ctp   1/1     Running   0          34s
    root@mck-devops-tools:/etc/kubernetes/minikube#


Задание 2: Просмотр логов для разработки

Конфигурация выполнялась согласно документа на сайте:

https://docs.bitnami.com/tutorials/configure-rbac-in-your-kubernetes-cluster/

Step 1: Create the app-namespace namespace

    kubectl create namespaceapp-namespace

    root@mck-devops-tools:/etc/kubernetes/minikube# kubectl get namespaces
    NAME                   STATUS   AGE
    app-namespace          Active   5s
    default                Active   8d
    ingress-nginx          Active   7d12h
    kube-node-lease        Active   8d
    kube-public            Active   8d
    kube-system            Active   8d
    kubernetes-dashboard   Active   8d
    root@mck-devops-tools:/etc/kubernetes/minikube#

Step 2: Create the user credentials

    Username: employee
    Group: bitnami

Генерация ключей и сертификата средствами openssl

    openssl genrsa -out employee.key 2048

    root@mck-devops-tools:/home/employee/.certs# openssl genrsa -out employee.key 2048
    Generating RSA private key, 2048 bit long modulus (2 primes)
    .....+++++
    ...........+++++
    e is 65537 (0x010001)
    root@mck-devops-tools:/home/employee/.certs# ls -l
    total 4
    -rw------- 1 root root 1679 Mar 23 06:20 employee.key
    root@mck-devops-tools:/home/employee/.certs#


    openssl req -new -key employee.key -out employee.csr -subj "/CN=employee/O=bitnami"

    root@mck-devops-tools:/home/employee/.certs# openssl req -new -key employee.key -out employee.csr -subj "/CN=employee/O=bitnami"
    root@mck-devops-tools:/home/employee/.certs# ls -l
    total 8
    -rw-r--r-- 1 root root  915 Mar 23 06:21 employee.csr
    -rw------- 1 root root 1679 Mar 23 06:20 employee.key
    root@mck-devops-tools:/home/employee/.certs#


    openssl x509 -req -in employee.csr -CA /root/.minikube/ca.crt -CAkey /root/.minikube/ca.key -CAcreateserial -out employee.crt -days 500


    где /root/.minikube/ca.key путь до cluster CA


    root@mck-devops-tools:/home/employee/.certs# openssl x509 -req -in employee.csr -CA /root/.minikube/ca.crt -CAkey /root/.minikube/ca.key -CAcreateserial -out employee.crt -days 500
    Signature ok
    subject=CN = employee, O = bitnami
    Getting CA Private Key
    root@mck-devops-tools:/home/employee/.certs# ls -l
    total 12
    -rw-r--r-- 1 root root 1017 Mar 23 06:27 employee.crt
    -rw-r--r-- 1 root root  915 Mar 23 06:21 employee.csr
    -rw------- 1 root root 1679 Mar 23 06:20 employee.key
    root@mck-devops-tools:/home/employee/.certs#


    kubectl config set-credentials employee --client-certificate=/home/employee/.certs/employee.crt  --client-key=/home/employee/.certs/employee.key

    root@mck-devops-tools:/home/employee/.certs# kubectl config set-credentials employee --client-certificate=/home/employee/.certs/employee.crt  --client-key=/home/employee/.certs/employee.key
    User "employee" set.
    root@mck-devops-tools:/home/employee/.certs#


    kubectl config set-context employee-context --cluster=minikube --namespace=app-namespace --user=employee

    root@mck-devops-tools:/home/employee/.certs# kubectl config set-context employee-context --cluster=minikube --namespace=app-namespace --user=employee
    Context "employee-context" created.
    root@mck-devops-tools:/home/employee/.certs#

Step 3: Create the role для просмотра логов

Создан файл для роли

    role-log-manager.yaml

    kind: Role
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      namespace: app-namespace
      name: log-manager
    rules:
    - apiGroups: ["", "extensions", "apps"]
      resources: ["pods"]
      verbs: ["get", "logs"]


    kubectl create -f /home/employee/role-log-manager.yaml

    root@mck-devops-tools:/home/employee# kubectl create -f /home/employee/role-log-manager.yaml
    role.rbac.authorization.k8s.io/log-manager created
    root@mck-devops-tools:/home/employee#


Step 4: Bind the role to the employee user

Создан файл для привязки роли к пользователю

    rolebinding-log-manager.yaml

    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: log-manager-binding
      namespace: app-namespace
    subjects:
    - kind: User
      name: employee
      apiGroup: ""
    roleRef:
      kind: Role
      name: log-manager
      apiGroup: ""

    kubectl create -f /home/employee/rolebinding-log-manager.yaml

    root@mck-devops-tools:/home/employee# kubectl create -f /home/employee/rolebinding-log-manager.yaml
    rolebinding.rbac.authorization.k8s.io/log-manager-binding created
    root@mck-devops-tools:/home/employee#

Step 5: Test the RBAC rule

    root@mck-devops-tools:/home/employee# kubectl --context=employee-context get deployments
    Error from server (Forbidden): deployments.apps is forbidden: User "employee" cannot list resource "deployments" in API group "apps" in the    namespace "app-namespace"

доступ к подам есть

    root@mck-devops-tools:/home/employee# kubectl --context=employee-context get pods
    No resources found in app-namespace namespace.
    root@mck-devops-tools:/home/employee#

    root@mck-devops-tools:/home/employee# kubectl --context=employee-context get pods --namespace=default
    Error from server (Forbidden): pods is forbidden: User "employee" cannot list resource "pods" in API group "" in the namespace "default"

Доступ к деплойментам и подам в другом ns закрыт

###  Задание 3: Изменение количества реплик

в deployment из задания 1 изменено количество реплик на 5

kubectl -n default scale --replicas=5 deploy hello-node


    root@mck-devops-tools:/home/employee# kubectl -n default scale --replicas=5 deploy hello-node
    deployment.apps/hello-node scaled
    root@mck-devops-tools:/home/employee#
    root@mck-devops-tools:/home/employee# kubectl get deployment
    NAME         READY   UP-TO-DATE   AVAILABLE   AGE
    hello-node   5/5     5            5           70m
    root@mck-devops-tools:/home/employee# kubectl get pods
    NAME                          READY   STATUS    RESTARTS   AGE
    hello-node-6b89d599b9-48h99   1/1     Running   0          70m
    hello-node-6b89d599b9-4nt4j   1/1     Running   0          15s
    hello-node-6b89d599b9-5kvrt   1/1     Running   0          15s
    hello-node-6b89d599b9-wbjp8   1/1     Running   0          70m
    hello-node-6b89d599b9-z67hd   1/1     Running   0          15s
    root@mck-devops-tools:/home/employee#



