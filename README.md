# * 사전 작업
 # 1. Azure Cloud Shell 로그인
 # 2. 환경변수 지정
 REGION_NAME=koreacentral
 SUBNET_NAME=aks-snet
 VNET_NAME=aks-vnet
 RESOURCE_GROUP=serahp-rg

 # 3. 확인
 echo $REGION_NAME

# * Azure CLI를 사용하여 네트워킹 구성
 # 1. 가상 네트워크 및 서브넷 구성
 az network vnet create \
 --resource-group $RESOURCE_GROUP \
 --location $REGION_NAME \
 --name $VNET_NAME \
 --address-prefixes 10.30.0.0/16 \
 --subnet-name $SUBNET_NAME \
 --subnet-prefix 10.30.0.0/24

 # 서브넷 ID를 저장하고 Bash 변수에 저장
 SUBNET_ID=$(az network vnet subnet show \
 --resource-group $RESOURCE_GROUP \
 --vnet-name $VNET_NAME \
 --name $SUBNET_NAME \
 --query id -o tsv)

# * AKS 클러스터 배포
 # 1. AKS 클러스터 이름 변수 지정 후, 생성
 AKS_CLUSTER_NAME=serahpaks
 az aks create --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --network-plugin azure --generate-ssh-keys

 # 2. AKS 자격 증명 획득
 az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME

 # 3. Node 확인
 kubectl get nodes

# * AKS 네임스페이스 생성
 # 1. 네임스페이스 생성
 kubectl create namespace ratingsapp
 # 2. 현재 네임스페이스 확인
 kubectl get namespace

# * Azure Container Registry 작업을 사용하여 컨테이너 이미지 빌드
 # Fruit Smoothies 평가 앱은 프런트 엔드 웹 사이트와 RESTful API 웹 서비스에 각각 1 개의 컨테이너 이미지를 사용합니다.
 # 개발 팀은 로컬 Docker 도구를 사용하여 웹 사이트 및 API 웹 서비스에 대한 컨테이너 이미지를 빌드합니다. 세 번째 컨테이너는 데이터베이스 게시자가 제공하는 문서 데이터베이스를 배포하는 데 사용되며 ACR 에는 데이터베이스 컨테이너가 저장되지 않습니다. Azure Container Registry 를 사용하여 표준 Dockerfile 로 이러한 컨테이너를 빌드하고 빌드 지침을 제공할 수 있습니다. Azure Container Registry 를 사용하면 다단계 빌드를 포함하여 모든 Dockerfile 을 현재 환경에서 다시 사용할 수 있습니다.
 # 먼저 Node.js 웹 프레임워크인 Express 를 사용하여 빌드된 ratings-api 이미지를 빌드하고 레지스트리에 푸시합니다. 소스 코드 는 GitHub 에 있으며, Node.js Alpine 컨테이너 이미지를 기반으로 이미지를 빌드하는 Dockerfile 이 이미 포함되어 있습니다. 여기에서 리포지토리를 복제한 다음, 포함된 Dockerfile 을 사용하여 Docker 이미지를 빌드하고 레지스트리에 푸시합니다.

 # 1. acr 생성 및 연동
 # acr 생성
 ACR_NAME=serahpacr
 az acr create --name $ACR_NAME --resource-group $RESOURCE_GROUP --sku basic

 # aks에 acr 연동
 az aks update --name $ACR_NAME --resource-group $RESOURCE_GROUP--attach-acr serahpacr

 # 2. az acr build 실행 (ratings-api)
 이 명령은 Dockerfile 을 사용하여 컨테이너 이미지를 빌드합니다. 그런 다음 결과 이미지를 컨테이너 레지스트리로 푸시합니다. 소스 코드는 GitHub 에 있습니다.
 git clone https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-api.git
 cd mslearn-aks-workshop-ratings-api
 az acr build --resource-group $RESOURCE_GROUP --registry $ACR_NAME --image ratings-api:v1 .

 # 3. az acr build 실행 (ratings-web / 이 명령은 Dockerfile 을 사용하여 컨테이너 이미지를 빌드합니다. 그런 다음 결과 이미지를 컨테이너 레지스트리로 푸시합니다. 소스 코드는 GitHub 에 있습니다.)
 cd ~
 git clone https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-web.git
 cd mslearn-aks-workshop-ratings-web
 az acr build --resource-group $RESOURCE_GROUP --registry $ACR_NAME --image ratings-web:v1 .

 # 4. acr 확인
 az acr repository list --name $ACR_NAME --output table
------------------------------------------------------------------------------------------------------
# * Helm 차트를 이용해 MongoDB 배포
 # Helm 은 Kubernetes 용 어플리케이션 패키지 관리자이며 차트를 사용해 어플리케이션과 서비스를 쉽게 배포할 수 있는 방법을 제공합니다.이 연습에서는 Helm 을 사용하여 AKS(Azure Kubernetes Service) 클러스터로 MongoDB 를 배포합니다. 

 # 1. bitnami 리포지토리를 사용하도록 Helm 클라이언트 구성
 helm repo add bitnami https://charts.bitnami.com/bitnami

 * 참고 : helm search repo 명령을 통해 설치할 차트 확인 가능
 helm search repo bitnami
 * Helm Hub : https://hub.helm.sh/

 # 2. 이전에 생성한 ratingsapp 네임스페이스를 지정하여 MongoDB 인스턴스 설치
 # 매개 변수에 set 항목에 사용자 이름, 암호, 데이터베이스 이름을 설정할 수 있습니다.
 helm install ratings bitnami/mongodb --namespace ratingsapp --set auth.username=testid,auth.password=testpw,auth.database=ratingsdb

 * 참고 : 삭제하려면 다음 명령어 사용
 helm uninstall ratings --namespace ratingsapp

 # 3. 설치가 완료된 후 나오는 MongoDB DNS Name을 기록해둡니다.
 ratings-mongodb.ratingsapp.svc.cluster.local

 # 4. service와 pod가 정상적으로 Running중인지 확인합니다.
 kubectl get svc --namespace ratingsapp
 kubectl get pod --namespace ratingsapp

 # Kubernetes Secret 생성
  # MongoDB 세부 정보를 보관할 Kubernetes Secret을 생성합니다. (namespace : ratingsapp)
  # Kubernetes 에서는 secret 을 사용해 암호와 같이 민감한 정보를 저장하고 관리할 수 있습니다. 이 정보를 secret 에 저장하면 Pod 정의 또는 컨테이너 이미지에 하드 코딩할 때보다 안전하고 유연합니다.

 # 1. Secret 생성
  # 다음 명령을 사용하여 ratingsapp 네임스페이스에 mongosecret 라는 비밀을 만듭니다. Kubernetes 비밀은 여러 항목을 보유할 수 있으며 키로 인덱싱됩니다. 이 경우에서는 비밀이 MONGOCONNECTION 이라는 키 하나만 포함합니다. 값은 이전 단계에서 생성된 연결 문자열입니다.
 kubectl create secret generic mongosecret --namespace ratingsapp -- from-literal=MONGOCONNECTION="mongodb://testid:testpw@ratings-mongodb.ratingsapp:27017/ratingsdb"

 # 2. 다음 명령을 실행하여 secret 유효성 검사
 kubectl describe secret mongosecret --namespace ratingsapp

 # 3. MongoDB 구성 완료
 
------------------------------------------------------------------------------------------------------
# ratings-api deploy 배포
 # 웹 프런트 엔드가 데이터베이스와 통신할 수 있도록 하는 RESTful API 인 ratings-api 를 배포합니다. ratings API 는 Express 프레임워크를 사용하여 작성된 Node.js 어플리케이션으로, MongoDB 데이터베이스에서 항목 및 해당 평가를 검색하고 저장합니다. RESTful API 에 대한 Kubernetes deployment 매니페스트 파일을 만들고, Kubernetes 서비스를 만들어 네트워크를 통해 RESTful API 를 노출합니다.
 
 # 1. ratings-api deployment 배포
 kubectl apply -f ratings-api-deployment.yaml -n ratingsapp

 # 확인
 kubectl get pods -n ratingsapp
 kubectl describe pods ratings-api-xxx -n ratingsapp
 kubectl get deploy ratings-api --namespace ratingsapp

 # 2. ratings-api service 생성
 kubectl apply -f ratings-api-service.yaml -n ratingsapp

 # 확인
 kubectl get svc ratings-api --n ratingsapp -w

 # 엔드포인트 확인
 kubectl get endpoints ratings-api --namespace ratingsapp
------------------------------------------------------------------------------------------------------
# ratings-web 배포 
 # 웹 프런트 엔드인 ratings-web 을 배포합니다. 평가 웹 프런트 엔드는 Node.js 어플리케이션입니다. 

 # 1. ratings-web deployment 배포
 kubectl apply -f ratings-web-deployment.yaml -n ratingsapp

 # 확인
 kubectl get pods -n ratingsapp
 kubectl get pods --namespace ratingsapp -l app=ratings-web -w
 kubectl get deploy ratings-web --namespace ratingsapp

 # 2. ratings-web 서비스 생성
 kubectl apply -f ratings-web-service.yaml -n ratingsapp

 # 확인
 kubectl get service ratings-web -n ratingsapp -w

 # 접속
 http://<External IP>
------------------------------------------------------------------------------------------------------
# Kubernetes ingresss controller 배포
 # Kubernetes ingresss controller는 L7 부하 분산 장치 기능을 제공하는 소프트웨어입니다. 해당 기능에는 역방향 프록시, 구성 가능한 트래픽 라우팅 및 Kubernetes 서비스에 대한 TLS 구성이 포함됩니다. ingresss controller를 설치하고 부하 분산 장치를 대체하도록 구성합니다. 
 # 여기서는 NGINX 를 사용하여 기본 Kubernetes ingresss controller를 배포합니다. 그런 다음, 해당 수신을 사용하도록 프런트엔드 서비스를 구성합니다. Helm 차트를 사용하여 ingresss controller 설치를 진행합니다

 # 1. ingress에 대한 네임스페이스 생성
 kubectl create namespace ingress

 # 2. helm 리포지토리 추가
 helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
 helm repo update

 # 3. Nginx Ingress Contrller 설치
 helm install ingress-nginx/ingress-nginx --generate-name --namespace ingress --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz --set controller.service.externalTrafficPolicy=Local

 # 확인
 kubectl get svc --namespace ingress
 확인 후 Ingress Controller의 External IP를 기록해둡니다.

 # 4. ClusterIP 를 사용하도록 ratings-web 서비스를 재구성
 # 텍스트 편집기로 ratings-web-service.yaml 파일을 열고, type 을 ClusterIP 로 변경합니다.
 vi ratings-web-service.yaml 

 #  배포된 서비스에서는 type 의 값을 업데이트할 수 없어, 먼저 서비스를 삭제하고 다시 배포합니다.
 kubectl delete --namespace ratingsapp -f ratings-web-service.yaml
 kubectl apply --namespace ratingsapp -f ratings-web-service.yaml
------------------------------------------------------------------------------------------------------
# SSL/TLS 적용
 # HTTPS 연결을 적용하도록 cert-manager 및 Let's Encrypt용 ClusterIssuer 리소스를 배포합니다.

 # 1. cert-manager 배포
 # cert-manager 는 클라우드 네이티브 환경에서 인증서 관리를 자동화하도록 해주는 Kubernetes 인증서 관리 컨트롤러입니다. 인증서 관리자는 Let's Encrypt, HashiCorp Vault, Venafi, 간단한 서명 키 쌍 또는 자체 서명된 인증서를 비롯한 다양한 소스를 지원합니다. cert-manager 를 사용하여 웹 사이트의 인증서가 유효하고 최신 상태인지 확인하고, 인증서가 만료되기 전에 구성된 시간에 인증서를 갱신합니다.

 # cert-manager 네임스페이스 생성
 kubectl create namespace cert-manager

 # Jetstack Helm 리포지토리 추가
 helm repo add jetstack https://charts.jetstack.io
 helm repo update

 # cert-manager 배포
 helm install cert-manager jetstack/cert-manager --namespace cert-manager --version v1.13.3 --set installCRDs=true

 # cert-manager 배포 확인
 kubectl get pods --namespace cert-manager

 # 2. Cluster Issuer 배포
 # 인증서 발급 서비스에 대한 인터페이스의 역할을 하는 ClusterIssuer 리소스를 배포합니다.

 # cluster issuer 배포
 kubectl apply -f cluster-issuer.yaml --namespace ratingsapp
 
 # 3. Kubernetes 수신기에서 SSL/TLS를 사용하도록 구성합니다.
 kubectl apply -f ratings-web-ingress.yaml --namespace ratingsapp 
 
 # 4. 인증서가 발급되었는지 확인합니다. 10분 이상 시간이 소요됩니다.
  # Message 값이 Certificate is up to date and has not expired 로 변경되어야 합니다.
 kubectl describe cert ratings-web-cert --namespace ratingsapp

 # 5. SSL이 정상적으로 적용되었는지 테스트합니다.
  # 브라우저에서 https:// 와 함께, 앞의 결과에 보이는 DNS Name 을 입력합니다.
 https://www.bbiyak.shop/#/
