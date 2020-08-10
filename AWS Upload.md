## AWS 인증 설정
*  AWS Console 접속 : https://skccaws.signin.aws.amazon.com/console
* 엑세스 키 만들기 : IAM > 사용자 > 계정 클릭 > [TAB]보안자격증명 > 엑세스키 만들기
  - Access Key ID, Secret Access Key 복사
* Local 에 AWS CLI 설치
  https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html
* ```aws configure``` > Access Key ID, Secret Access Key 등록, Default region name, Default output format 등록

## AWS Tool 설치
* eksctl 설치
```sh
brew tap weaveworks/tap
brew install eksctl
brew upgrade eksctl && brew link --overwrite eksctl
### 확인
eksctl version
```

## 클러스터 생성
```sh
eksctl create cluster --name (Cluster-Name) --version 1.15 --nodegroup-name standard-workers --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 3
```

## ECR Repository 생성 & Docker Client 인증
```
ECR > 리포지토리 > 리포지토리 생성
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 351904566406.dkr.ecr.us-east-2.amazonaws.com
```

## Docker Image Build & Push
```sh
mvn package
docker build -t 351904566406.dkr.ecr.us-east-2.amazonaws.com/aimmvp-order:v1 .
docker images # 생성된 Image 확인
docker push 351904566406.dkr.ecr.us-east-2.amazonaws.com/aimmvp-order:v1 
```

## Deploy & Create Service
  - namespace 만들기
  ```sh
  kubectl create namespace istio-cb-ns
  ```
  - 소스수정
 ```yaml
 #kubernetes/deployment.yml 추가&수정
 metadata.namespace: istio-cb-ns
 spec.template.spec.containers.image: 351904566406.dkr.ecr.us-east-2.amazonaws.com/aimmvp-order:v1 

 #kubernetes/service.yaml 추가&수정
 metadata.namespace: istio-cb-ns
 ```

 - create pod, service
 ```sh
 kubectl create -f deployment.yml
 kubectl create -f service.yml
 ```

 - pod, service 확인
  ```sh
  kubectl get all -n istio-cb-ns
  ```

## Install kafka on k8s
 - helm 버전 확인
```sh
helm version
```

- install helm & install kafka(@helm version is 2.x) 
```sh
# install helm
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
kubectl --namespace kube-system create sa tiller      # helm 의 설치관리자를 위한 시스템 사용자 생성
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller

kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

#install kafka
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
helm repo update
helm install --name my-kafka --namespace kafka incubator/kafka

# kafka running status 
kubectl get po -n kafka
```
## kafka message 확인 on k8s
- create topic "cnademo"
```sh
kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-topics --zookeeper my-kafka-zookeeper:2181 --topic cnademo --create --partitions 1 --replication-factor 1
```

- topic 'cnademo' 확인
```sh
kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-topics --zookeeper my-kafka-zookeeper:2181 --list
```

- produce event
```sh
kubectl -n kafka exec -it my-kafka-0 -- /usr/bin/kafka-console-producer --broker-list my-kafka:9092 --topic cnademo
```
- consume event 
```sh
kubectl -n kafka exec -it my-kafka-0 -- /usr/bin/kafka-console-consumer --bootstrap-server my-kafka:9092 --topic cnademo --from-beginning
```

### kafka 참고 URL : https://dzone.com/articles/how-to-set-up-and-run-kafka-on-kubernetes

## istio
- Install  istio
```sh
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.4.3 sh -
cd istio-1.4.3
export PATH=$PWD/bin:$PATH
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done

kubectl apply -f install/kubernetes/istio-demo.yaml
```
- 설치확인
```sh
istioctl version
```

- order, delivery에 istio 적용
```sh
kubectl get deploy order -n istio-cb-ns -o yaml > order_deploy.yaml
kubectl -n istio-cb-ns apply -f <(istioctl kube-inject -f order_deploy.yaml)

kubectl get deploy delivery -n istio-cb-ns -o yaml > delivery_deploy.yaml
kubectl -n istio-cb-ns apply -f <(istioctl kube-inject -f delivery_deploy.yaml)
```

- delivery 에 scale out 적용
```sh
kubectl scale deploy delivery -n istio-cb-ns --replicas=2
```

- Circuit Breaker 적용
```sh
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: delivery
  namespace: istio-cb-ns
spec:
  host: products
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 2
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 5
      interval: 1s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
EOF
```

