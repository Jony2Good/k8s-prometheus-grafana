# Задание
*Цель*: научиться инструментировать сервис в формате Prometheus-Grafana

### Задача

1. Сделать дашборд в Графане, в котором были бы метрики с разбивкой по API методам:
   - Latency (response time) с квантилями по 0.5, 0.95, 0.99, max
   - RPS
   - Error Rate - количество 500ых ответов

2. Добавить в дашборд графики с метрикам в целом по сервису, взятые с nginx-ingress-controller:
   - Latency (response time) с квантилями по 0.5, 0.95, 0.99, max
   - RPS
   - Error Rate - количество 500ых ответов

3. Настроить алертинг в графане на Error Rate и Latency.

#### На выходе должны быть предоставлена

1. Cкриншоты дашбордов с графиками в момент стресс-тестирования сервиса (например, после 5-10 минут нагрузки).
2. json-дашборды.

------------

### Результат
1. [![Пример Grafana доски для метрик от app](https://github.com/Jony2Good/assets/blob/main/laravel-grafana.png "Пример доски для метрики от Laravel")](https://github.com/Jony2Good/assets/blob/main/laravel-grafana.png "Пример доски для метрики от Laravel")

2. [![Пример Grafana доски для метрик от ingress-nginx](https://github.com/Jony2Good/assets/blob/main/nginx-ingress-grafana.png "Пример Grafana доски для ingress-nginx")](https://github.com/Jony2Good/assets/blob/main/nginx-ingress-grafana.png "Пример Grafana доски для ingress-nginx")

3. [![Пример настройки алертинга](https://github.com/Jony2Good/assets/blob/main/grafana-alerting.png "Пример настройки алертинга")](https://github.com/Jony2Good/assets/blob/main/grafana-alerting.png "Пример настройки алертинга")

4. Grafana доска в формате json [panel][1]
------------
### Инструкция по запуску приложения

**Клонирум проект в локальный репозиторий**

 ```
 git clone https://github.com/Jony2Good/k8s-prometheus-grafana.git
```

**Стартуем minikube**

```
minikube start
```

**Устанавливаем Prometheus стэк из helm репозитория**

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```
helm repo update
```

```
helm install prometheus prometheus-community/kube-prometheus-stack --namespace k8s-basics --create-namespace
```

**Инициализируем манифесты**

```
kubectl apply -f k8s/
```

**Проверяем pods**

```
kubectl get pods -n k8s-basics
```

### Установка ingress-nginx + конфигурация Prometheus

***Проверяем Ingress (его не будет, далее установим)***

```
kubectl get ingress -n k8s-basics
```

**Получаем нужный репозиторий из helm**

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

**Устанавливаем ingress-controller в namespase k8s-basics и конфигурируем Prometheus**

```
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx --namespace k8s-basics --set controller.metrics.enabled=true --set-string controller.podAnnotations."prometheus.io/scrape"="true" --set-string controller.podAnnotations."prometheus.io/port"="10254"
```

**Проверяем services**

```
kubectl get svc -n k8s-basics
```

Должна появится таблица. Нас интересует только ingress-nginx-controller. У него должен быть *Type:LoadBalanser* и *External-IP:pending*

| NAME                    | TYPE         | CLUSTER-IP     | EXTERNAL-IP    | PORT(S)                    | AGE |
| ----------------------- | ------------ | -------------- | -------------- | -------------------------- | --- |
| ngress-nginx-controller | LoadBalancer | 10.104.118.219 |  pending  | 80:31047/TCP,443:31617/TCP | 95m |


### Проброс ingress-nginx наружу 

**Нам необходимо, чтобы домен arch.homework открывался без порта и по специальному url. Для этого мы должны прописать команду:**

```
minikube tunnel
```

Далее снова смотрим в таблицу на значение в колонке **EXTERNAL-IP** (вместо pending должно появится значение, к примеру, 10.107.105.245) в ней должен прописаться внешний IP-адрес.

```
kubectl get svc -n k8s-basics
```

Копируем данный внешний адрес и прописываем в файле hosts машины правило маршрутизации, где развернут кластер:

```
10.107.105.245 arch.homework
```
Справка: для ОС windows прописываем в файле и сохраняем:
```
C:\Windows\System32\drivers\etc\hosts
```

### Работа с Prometheus + Grafana

**Проверяем наличие ServiceMonitor**

```
kubectl get servicemonitor -n k8s-basics
```

В таблице должны отобразиться записи:
  - ingress-nginx-monitor
  - laravel-service-monitor

**Запускаем Prometheus dashboard**

```
kubectl -n k8s-basics port-forward svc/prometheus-kube-prometheus-prometheus 9090
```

Вводим в браузере url: http://127.0.0.1:9090/targets

Находим записи в статусе Up:
  - serviceMonitor/k8s-basics/laravel-service-monitor/0 - метрики приложения laravel
  - serviceMonitor/k8s-basics/ingress-nginx-monitor/0 - метрики ingress-nginx


**Запускаем Gafana**

```
kubectl -n k8s-basics port-forward svc/prometheus-grafana 3000:80
```

Вводим в браузере url: http://127.0.0.1:3000


**Получаем пароль**

```
kubectl -n k8s-basics get secret prometheus-grafana -o jsonpath="{.data.admin-password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

### Создаем Grafana dashboard

1. Метрики по API методам

- Latency (квантиль 0.5 / 0.95 / 0.99)

```
histogram_quantile(0.5, sum by (le, method, path) (rate(app_http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le, method, path) (rate(app_http_request_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le, method, path) (rate(app_http_request_duration_seconds_bucket[5m])))
max by (method, path) (rate(app_http_request_duration_seconds_sum[5m]) / rate(app_http_request_duration_seconds_count[5m]))
```

-  RPS по методам/эндпоинтам

```
sum by (method, path) (rate(app_http_requests_total[1m]))
```

- Error Rate (количество 5xx ответов)

```
sum by (method, path) (rate(app_http_requests_total{status=~"5.."}[5m]))
```

2. ingress-nginx (общие метрики по сервису)

- Latency (Ingress, квантиль 0.5 / 0.95 / 0.99)

```
histogram_quantile(0.5, sum by (le) (rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le) (rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le) (rate(nginx_ingress_controller_request_duration_seconds_bucket[5m])))
max(rate(nginx_ingress_controller_request_duration_seconds_sum[5m]) / rate(nginx_ingress_controller_request_duration_seconds_count[5m]))
```

- RPS

```
sum(rate(nginx_ingress_controller_requests[1m]))
```

- Error Rate

```
sum(rate(nginx_ingress_controller_requests{status=~"5.."}[5m]))
```

***Базовый url приложения:*** http://arch.homework/otusapp/aemelyanenko

[1]: https://github.com/Jony2Good/k8s-prometheus-grafana/blob/main/panel.json "Grafana-panel"
