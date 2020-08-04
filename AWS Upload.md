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
docker push351904566406.dkr.ecr.us-east-2.amazonaws.com/aimmvp-order:v1 
```