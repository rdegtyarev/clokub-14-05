# Домашнее задание к занятию "14.5 SecurityContext, NetworkPolicies"

## Задача 1: Рассмотрите пример 14.5/example-security-context.yml

<details>

  <summary>Описание задачи</summary> 

Создайте модуль

```
kubectl apply -f 14.5/example-security-context.yml
```

Проверьте установленные настройки внутри контейнера

```
kubectl logs security-context-demo
uid=1000 gid=3000 groups=3000
```
</details>


### Решение

1. Применяем манифест
```
kubectl apply -f ./14.5/example-security-context.yml
pod/security-context-demo created
```

2. Смотрим логи запущенного контейнера
```
kubectl logs security-context-demo
uid=1000 gid=3000 groups=3000
```
Контейнер запущен с правами пользователя c uid 1000 группы 3000 в соотвествии с

```yaml
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
```


---

## Задача 2 (*): Рассмотрите пример 14.5/example-network-policy.yml

<details>

  <summary>Описание задачи</summary> 

Создайте два модуля. Для первого модуля разрешите доступ к внешнему миру
и ко второму контейнеру. Для второго модуля разрешите связь только с
первым контейнером. Проверьте корректность настроек.
</details>


### Решение

1. Создаем манифесты для двух подов.

Первый модуль: ./task-2/pods/01-dp.yaml  
Доступ к внешнему миру и ко второму контейнеру. 
  
Второй модуль: ./task-2/pods/02-dp.yaml  
Связь только с первым контейнером.

 2. Создаем манифест для NetworkPolicy

По умолчанию закрываем все ingress и egress в подах:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

Создаем политику, разрешающую доступ из первого модуля любой внешний доступ (в том числе внешний мир и второй контейнер), и разрешающую входящие соединения только от второго контейнера.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-pod-1
spec:
  podSelector:
    matchLabels:
      role: pod-1
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: pod-2
  egress:
  - {}
```
Создаем политику, разрешающую доступ из второго модуля только к первому контейнеру, и входящие соединения только от первого контейнера:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-pod-2
spec:
  podSelector:
    matchLabels:
      role: pod-2
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: pod-1
  egress:
    - to:
        - podSelector:
            matchLabels:
              role: pod-1

```


3. Создадим неймспейс и задеплоим поды
```
[roman@fedora clokub-14-05]$ kubectl create namespace clokub-14-05
namespace/clokub-14-05 created

kubectl apply -f ./task-2/pods -n clokub-14-05
deployment.apps/pod-1 created
deployment.apps/pod-2 created
```

4. Проверим доступ из подов друг к другу и к внешнему миру

```shell
kubectl get pods -n clokub-14-05 -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE                        NOMINATED NODE   READINESS GATES
pod-1-5ffc7bf84d-7t7kk   1/1     Running   0          22m   10.112.131.7    cl10p6oi6aqsnpccnn8e-imah   <none>           <none>
pod-2-57b874bccd-pkz6v   1/1     Running   0          22m   10.112.130.17   cl10p6oi6aqsnpccnn8e-ekes   <none>           <none>

# проверяем доступность pod-2 из pod-1
kubectl exec -n clokub-14-05 pod-1-5ffc7bf84d-7t7kk -- curl -s -m 1 10.112.130.17
Praqma Network MultiTool (with NGINX) - pod-2-57b874bccd-pkz6v - 10.112.130.17

# проверяем доступность внешнего мира из pod-1
kubectl exec -n clokub-14-05 pod-1-5ffc7bf84d-7t7kk -- ping -c 3 ya.ru
PING ya.ru (87.250.250.242) 56(84) bytes of data.
64 bytes from ya.ru (87.250.250.242): icmp_seq=1 ttl=250 time=0.605 ms
64 bytes from ya.ru (87.250.250.242): icmp_seq=2 ttl=250 time=0.391 ms
64 bytes from ya.ru (87.250.250.242): icmp_seq=3 ttl=250 time=0.429 ms

# проверяем доступность pod-1 из pod-2
kubectl exec -n clokub-14-05 pod-2-57b874bccd-pkz6v -- curl -s -m 1 10.112.131.7
Praqma Network MultiTool (with NGINX) - pod-1-5ffc7bf84d-7t7kk - 10.112.131.7

# проверяем доступность внешнего мира из pod-2
kubectl exec -n clokub-14-05 pod-2-57b874bccd-pkz6v -- ping -c 3 ya.ru
PING ya.ru (87.250.250.242) 56(84) bytes of data.
64 bytes from ya.ru (87.250.250.242): icmp_seq=1 ttl=250 time=0.725 ms
64 bytes from ya.ru (87.250.250.242): icmp_seq=2 ttl=250 time=0.430 ms
64 bytes from ya.ru (87.250.250.242): icmp_seq=3 ttl=250 time=0.404 ms

```


5. Применяем сетевые политики и проверяем:  

```shell
kubectl apply -n clokub-14-05 -f ./task-2/np/
networkpolicy.networking.k8s.io/default-deny-ingress created


# проверяем доступность pod-2 из pod-1
kubectl exec -n clokub-14-05 pod-1-5ffc7bf84d-7t7kk -- curl -s -m 1 10.112.130.17
Praqma Network MultiTool (with NGINX) - pod-2-57b874bccd-pkz6v - 10.112.130.17
# Из первого контейнера доступен второй


# проверяем доступность внешнего мира из pod-1
kubectl exec -n clokub-14-05 pod-1-5ffc7bf84d-7t7kk -- ping -c 3 ya.ru
PING ya.ru (87.250.250.242) 56(84) bytes of data.
64 bytes from ya.ru (87.250.250.242): icmp_seq=1 ttl=250 time=0.371 ms
64 bytes from ya.ru (87.250.250.242): icmp_seq=2 ttl=250 time=0.339 ms
64 bytes from ya.ru (87.250.250.242): icmp_seq=3 ttl=250 time=0.378 ms
# из первого контейнера доступен внешний мир

# проверяем доступность pod-1 из pod-2
kubectl exec -n clokub-14-05 pod-2-57b874bccd-pkz6v -- curl -s -m 1 10.112.131.7
Praqma Network MultiTool (with NGINX) - pod-1-5ffc7bf84d-7t7kk - 10.112.131.7
# Из второго контейнера доступен первый


# проверяем доступность внешнего мира из pod-2
kubectl exec -n clokub-14-05 pod-2-57b874bccd-pkz6v -- ping -c 3 ya.ru
ping: ya.ru: Try again
command terminated with exit code 2
# Из второго контейнера недоступен внешний мир

```

---
