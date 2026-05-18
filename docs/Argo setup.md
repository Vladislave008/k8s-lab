# Запуск ArgoCD

## Подготовка кластера и лейблы
```
minikube delete
minikube start --nodes 2
kubectl get nodes
kubectl label node minikube role=system
kubectl label node minikube-m02 role=app
kubectl label node minikube-m02 disk=fast
```

## Установка CSI S3 драйвера
```
helm install csi-s3 yandex-s3/csi-s3 --namespace kube-system --version 0.42.1
kubectl get pods -n kube-system | Select-String "csi-s3"
```

## Секреты
```
kubectl create namespace messager-team
kubectl apply -f k8s/base/csi-s3-secret.yaml
kubectl apply -f k8s/base/csi-s3-secret-system.yaml
kubectl apply -f k8s/base/secret.yaml
```

## Argo CD (установка и настройка)
```
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --namespace argocd --create-namespace
kubectl get pods -n argocd
```

*ErrImagePull для Redis пришлось фиксить вручную:*
```
docker pull redis:7.4-alpine
minikube image load redis:7.4-alpine
kubectl delete pod -n argocd -l app.kubernetes.io/component=redis
```

## Открыть доступ к ArgoCD по адресу https://localhost:8443/
```
kubectl port-forward svc/argocd-server -n argocd 8443:443
```

## Получить пароль для ArgoCD и войти
```
$password = kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($password))
```

## Применить сборку
```
kubectl apply -f argocd/application.yaml
```