# kubernetis-dz10
Как работает сеть в K8s

# Домашнее задание к занятию «Как работает сеть в K8s»

### Цель задания

Настроить сетевую политику доступа к подам.

Фиксируем сетевые интерфейсы до установки Calico:

![alt text](image-3.png)

и маршрутизацию

![alt text](image-4.png)

### Чеклист готовности к домашнему заданию

1. Кластер K8s с установленным сетевым плагином Calico.

Установка оператора Calico:
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
```
![alt text](image-5.png)

![alt text](image-11.png)

Политики созданы.



### Задание 1. Создать сетевую политику или несколько политик для обеспечения доступа

1. Создать deployment'ы приложений frontend, backend и cache и соответсвующие сервисы.

[text](homework-dz10.yaml)

2. В качестве образа использовать network-multitool.

![alt text](image-12.png)

3. Разместить поды в namespace App.

![alt text](image-13.png)

4. Создать политики, чтобы обеспечить доступ frontend -> backend -> cache. Другие виды подключений должны быть запрещены.

Структура манифеста:
- Namespace `app`
- Deployment `frontend` (2 реплики) + Service
- Deployment `backend` (2 реплики) + Service  
- Deployment `cache` (2 реплики) + Service
- NetworkPolicy для контроля трафика


5. Продемонстрировать, что трафик разрешён и запрещён.

![alt text](image-14.png)

![alt text](image-15.png)

Траблшутинг, 53 порт блокируется

![alt text](image-16.png)

![alt text](image-17.png)

Метка есть
![alt text](image-18.png)

Проверяем фаерволы
![alt text](image-19.png)

Network Policy работают
![alt text](image-20.png)

UDP трафик проходит TCP и  ICMP ping непроходят

Проверяем сетевую связанность между worker и master

![alt text](image-21.png)

 Calico блокирует VXLAN трафик (порт 4789 UDP) правилом:
ipset cali40all-vxlan-net — если воркеры не добавлены в этот набор, VXLAN пакеты DROPаются.

![alt text](image-22.png)

Фиксим
![alt text](image-23.png)

Проверяем
```
FRONTEND=$(kubectl get pod -n app -l app=frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND=$(kubectl get pod -n app -l app=backend -o jsonpath='{.items[0].metadata.name}')

echo "frontend -> backend (разрешено)"
kubectl exec -n app "$FRONTEND" -- curl -m 3 http://backend

echo
echo "backend -> cache (разрешено)"
kubectl exec -n app "$BACKEND" -- curl -m 3 http://cache

echo
echo "frontend -> cache (запрещено)"
kubectl exec -n app "$FRONTEND" -- curl -m 3 http://cache

```

![alt text](image-24.png)


Тестирование
![alt text](image-25.png)

Каждому серверу выделена своя подсеть:
![alt text](image-26.png)

Сервисная сеть:
![alt text](image-27.png)

Тестирование:
 frontend -> backend разрешен
![alt text](image-28.png)

Добавим блокировку и снимем ее после теста: [text](block-frontend-egress.yaml)

![alt text](image-29.png)

давай для backend → cache и frontend → cache) аналогично: 

 frontend -> cache запрещен
![alt text](image-30.png)

 backend -> cache разрешён

backend -> cache трафик разрешён – проверяем
kubectl exec -n app deploy/backend -- nc -zv -w 3 cache 6379

Ожидаем: Connection refused

![alt text](image-31.png)

backend -> cache включаем блокировку egress для backend
kubectl apply -f deny-backend-egress.yaml

![alt text](image-32.png)

backend -> cache  проверяем – трафик заблокирован (таймаут)
![alt text](image-33.png)


backend -> cache  Откатываем
kubectl delete networkpolicy deny-backend-egress -n app

и сразу

backend -> cache проверяем что трафик снова доходит (Connection refused)
kubectl exec -n app deploy/backend -- nc -zv -w 3 cache 6379

![alt text](image-34.png)

Применяем полный конфиг политик

[text](networkpolicy.yaml)


Что включено в этот конфиг:
Политика	                Назначение
allow-icmp	                Пинг между всеми подами и namespace'ами (для проверки базовой связности)
allow-dns-egress	        Исходящие DNS-запросы в kube-system
allow-dns-ingress	        Ответы от DNS-сервера обратно к подам
default-deny	            Запрещает всё, что не разрешено явно
frontend-egress-backend	    frontend → backend:80
backend-ingress-frontend	backend принимает от frontend:80
backend-egress-cache	    backend → cache:6379
cache-ingress-backend	    cache принимает от backend:6379

Проверка 1   (DNS)

![alt text](image-35.png)

Проверка 2   (трафик: frontend -> backend)

![alt text](image-36.png)

Проверка 3   (трафик: backend → cache)

![alt text](image-37.png)
* трафик дошел и был refused 

Проверка 4   (Запрещённый трафик: frontend -> cache)

![alt text](image-38.png)
* виден тайм-out

Проверка 5   (Запрещённый трафик: backend -> frontend)
![alt text](image-39.png)
* блокировка

Проверка 6   (Запрещённый трафик: cache -> Any where)

![alt text](image-40.png)


Проверка 7   (Запрещаем и разрешаем в политике, применяем)

![alt text](image-41.png)

![alt text](image-42.png)
** при наложении запрещающей и разрешающей полити в кубере приоритет имеет та политика, которая разрешающая

Удаляем разрешающие политики и получаем блок
![alt text](image-43.png)

Возвращаем обратно в исхоное положение:

![alt text](image-44.png)


:-)  Соррян что задержался...

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Calico](https://www.tigera.io/project-calico/).
2. [Network Policy](https://kubernetes.io/docs/concepts/services-networking/network-policies/).
3. [About Network Policy](https://docs.projectcalico.org/about/about-network-policy).

-----