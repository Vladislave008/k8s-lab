minikube delete

minikube start --nodes 2

kubectl get nodes

kubectl label node minikube role=system
kubectl label node minikube-m02 role=app
kubectl label node minikube-m02 disk=fast

helm install csi-s3 yandex-s3/csi-s3 --namespace kube-system --version 0.42.1

kubectl get pods -n kube-system | Select-String "csi-s3"

kubectl create namespace messager-team

kubectl apply -f k8s/base/csi-s3-secret.yaml 
kubectl apply -f k8s/base/csi-s3-secret-system.yaml

kubectl apply -k k8s/base/

kubectl get pods -n messager-team

kubectl apply -f k8s/base/users-migrations-cm.yaml
kubectl apply -f k8s/base/messages-migrations-cm.yaml
kubectl apply -f k8s/base/migrate-users-job.yaml
kubectl apply -f k8s/base/migrate-messages-job.yaml

kubectl apply -f k8s/base/user-service-deployment.yaml
kubectl apply -f k8s/base/message-service-deployment.yaml
kubectl apply -f k8s/base/bff-deployment.yaml
kubectl apply -f k8s/base/frontend-deployment.yaml

minikube addons enable ingress
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
http://localhost:8080



Чекнуть бд:
kubectl exec -n messager-team deployment/postgres -- psql -U admin -d messager-db -c "\dt"

Обнулить гуся если не создал таблицу:
# 1. Удалить запись о версии для messages
kubectl exec -n messager-team deployment/postgres -- psql -U admin -d messager-db -c "DELETE FROM goose_db_version WHERE version_id=1 AND (SELECT EXISTS (SELECT 1 FROM goose_db_version WHERE id=2));"

# 2. Удалить джобу
kubectl delete job migrate-messages-v2 -n messager-team

# 3. Запустить заново
kubectl apply -f k8s/base/migrate-messages-job.yaml
