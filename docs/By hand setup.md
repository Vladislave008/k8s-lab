**! Предоставленные команды актуальны под виндой**

# Стандартный запуск

## Запуск minikube и создание меток нод

``` 
minikube delete
minikube start --nodes 2
kubectl get nodes
```

*Дождаться, пока ноды станут Ready*

```
kubectl label node minikube role=system
kubectl label node minikube-m02 role=app
kubectl label node minikube-m02 disk=fast
```

## Установка S3-CSI драйвера

```
helm repo add yandex-s3 https://yandex-cloud.github.io/k8s-csi-s3/charts
helm install csi-s3 yandex-s3/csi-s3 --namespace kube-system --version 0.42.1
kubectl get pods -n kube-system | Select-String "csi-s3"
```
*Дождаться запуска всех подов драйвера*

## Поэтапный запуск подов

```
kubectl create namespace messager-team

kubectl apply -f k8s/base/secret.yaml 
kubectl apply -f k8s/base/csi-s3-secret.yaml 
kubectl apply -f k8s/base/csi-s3-secret-system.yaml
```

```
kubectl apply -k k8s/base/
kubectl get pods -n messager-team
```

*Дождаться запуска всех деплойментов. Ошибки миграций и методы их решения описаны ниже*

В отдельной консоли:
```
minikube addons enable ingress
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```
В результате приложение доступно по адресу http://localhost:8080

# Проблемы и решения

## Goose и миграции

Иногда goose неверно накатывает миграции. Если какие-то деплои изначально  в Running, но 10 раз перезапускаются на health-чеках, стоит проверить таблицы:

```
kubectl exec -n messager-team deployment/postgres -- psql -U admin -d messager-db -c "\dt"
```
Если какой-то таблицы (users/messages/files) нет, необходимо очистить системную таблицу гуся и затем применить миграцию снова:

1. Удалить запись о версиях.
```
kubectl exec -n messager-team deployment/postgres -- psql -U admin -d messager-db -c "
DELETE FROM goose_db_version WHERE version_id IN (1,2,3);
"
```

2. Удалить джобы
```
kubectl delete job migrate-users -n messager-team --ignore-not-found
kubectl delete job migrate-messages-v2 -n messager-team --ignore-not-found
```

3. Запустить заново
```
kubectl apply -f k8s/base/migrate-users-job.yaml
kubectl apply -f k8s/base/migrate-messages-job.yaml
```

## ErrImagePull
Часто образы не качаются, вызывая эту ошибку. Тут спасает только включение ВПН или отдельная ручная установка файлов.