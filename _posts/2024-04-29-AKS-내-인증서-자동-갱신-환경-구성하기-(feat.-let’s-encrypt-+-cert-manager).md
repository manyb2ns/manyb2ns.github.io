---
layout: post
date: 2024-04-29
title: "AKS 내 인증서 자동 갱신 환경 구성하기 (feat. let’s encrypt + cert-manager)"
tags: [TLS, cert-manager, let's encrypt, AKS, DevOps, ]
categories: [AKS, ]
---


AKS에 배포한 웹앱의 HTTPS 통신을 위해 인증서를 적용하려고 알아보던 중, 이전에 회사 동료분이 알려주셨던 Let’s Encrypt라는 무료 인증서 발급 사이트가 떠올라 사용하기로 했다.


자료를 찾다보니 cert-manager를 함께 사용해서 인증서를 자동 갱신하는 사례가 효율적이고 많이 사용되는 것으로 보여 본 게시물에서도 Let’s Encrypt + cert-manager 조합으로 AKS 웹앱의 인증서 자동 갱신 환경을 구성하려고 한다.


## 0. Overview


### 0.1 Let’s Encrypt? cert-manager?

- [Let’s Encrypt](https://letsencrypt.org/) : TLS 인증서를 무료로 제공하는 비영리 인증 기관(CA)
- cert-manager : k8s 환경에서 TLS 인증서 발급 요청 및 자동 갱신과 같은 관리 역할을 제공
	- Certificate
	- (Cluster) Issuer
	- CertificateRequest
	- Order
	- Challenge

### 0.2 간단한 아키텍처와 흐름


![0](/assets/img/2024-04-29-AKS-내-인증서-자동-갱신-환경-구성하기-(feat.-let’s-encrypt-+-cert-manager).md/0.png)_출처 : https://www.nginx.com/blog/automating-certificate-management-in-a-kubernetes-environment/_

1. cert-manager는 Let’s Encrypt(CA)로부터 인증서를 발급받기 위해 소유한 도메인의 DNS 영역에 특수 레코드를 생성
2. 레코드 등록을 마치고 Let’s Encrypt에게 자신의 도메인을 소유하고 있음을 인증하기 위한 Challenge를 요청
3. Let’s Encrypt는 Challenge 요청을 수신 후 특수 도메인을 확인
4. 해당 도메인에 대한 확인을 마치고 Let’s Encrypt는 cert-manager로 인증서를 공급
5. cert-manager는 공급받은 인증서를 AKS Secret 내 저장

## 1. Labs


### 1.0 준비 사항

- Azure CLI 설치 ≥ v2.40.0
	- [최신 버전(v2.59.0) 패키지 다운로드 링크](https://github.com/Azure/azure-cli/archive/refs/tags/azure-cli-2.59.0.tar.gz)
- kubectl 설치
- 외부 접근이 가능한 Azure DNS 영역 생성
	- 외부 도메인 구매 시, 도메인 네임서버를 Azure DNS 영역의 네임 서버로 교체 필요
- Azure Kubernetes Service(AKS) 클러스터 생성

### 1.1 환경 변수 초기화

- 본 게시글에선 k8s 매니페스트 작성 시 환경 변수를 활용할 예정이므로 개인의 테스트 환경에 맞춰 아래 변수 초기화 단계를 진행

{% raw %}
```bash
# Azure 테스트 환경
export AZURE_SUBSCRIPTION=<구독 명>
export AZURE_SUBSCRIPTION_ID=$(az account show --name $AZURE_SUBSCRIPTION --query 'id' -o tsv)
export AZURE_DEFAULTS_GROUP=<리소스 그룹 명>
export AZURE_DEFAULTS_LOCATION=<리전>
export DOMAIN_NAME=<Public DNS 도메인>

# AKS 관련
export CLUSTER=<AKS 클러스터 명>
export AZURE_LOADBALANCER_DNS_LABEL_NAME=<AKS Public IP의 DNS Label 임의 지정> # EX) demo-lb
export AKS_OIDC_ISSUER="$(az aks show -n demo-cluster -gAZURE_DEFAULTS_GROUP $ --query "oidcIssuerProfile.issuerUrl" -o tsv)" # 생략 가능할듯한데

# ETC
export EMAIL_ADDRESS=<이메일 주소>
```
{% endraw %}


### 1.2 cert-manager 설치


{% raw %}
```bash
# cert-manager 설치
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade cert-manager jetstack/cert-manager \
    --install \
    --create-namespace \
    --wait \
    --namespace cert-manager \
    --set installCRDs=true
    
# 정상 설치 여부 및 컴포넌트 확인
kubectl -n cert-manager get all # STATUS Running 여부

kubectl explain Certificate
kubectl explain CertificateRequest
kubectl explain Issuer
```
{% endraw %}


### 1.3 cert-manager의 Azure 인증/권한 구성


{% raw %}
```bash
# AKS 클러스터의 MI 인증 활성화
az extension add --name aks-preview
az aks update -g $AZURE_DEFAULTS_GROUP -n $CLUSTER \
--enable-oidc-issuer \
--enable-workload-identity \
--enable-managed-identity

# MI 생성
## cert-manager의 Azure DNS API 사용을 위한 인증 주체 생성
export USER_ASSIGNED_IDENTITY_NAME="demo-mi"
az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}"

## MI가 Azure DNS 레코드를 생성할 수 있는 RBAC 할당
export USER_ASSIGNED_IDENTITY_CLIENT_ID=$(az identity show --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -o tsv --resource-group $AZURE_DEFAULTS_GROUP)

az role assignment create \
    --role "DNS Zone Contributor" \
    --assignee $USER_ASSIGNED_IDENTITY_CLIENT_ID \
    --scope $(az network dns zone show --name $DOMAIN_NAME -o tsv --query id)

export SERVICE_ACCOUNT_NAME=cert-manager
export SERVICE_ACCOUNT_NAMESPACE=cert-manager
export SERVICE_ACCOUNT_ISSUER=$(az aks show --resource-group $AZURE_DEFAULTS_GROUP --name $CLUSTER --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Federation 생성
az identity federated-credential create \
  --name 'cert-manager' \
  -g $AZURE_DEFAULTS_GROUP \
  --identity-name $USER_ASSIGNED_IDENTITY_NAME \
  --issuer $SERVICE_ACCOUNT_ISSUER \
  --subject "system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}"

# cert-manager 라벨 구성
## values.yaml
podLabels:
  azure.workload.identity/use: "true"
serviceAccount:
  labels:
    azure.workload.identity/use: "true"

helm upgrade cert-manager jetstack/cert-manager \
    --namespace $SERVICE_ACCOUNT_NAMESPACE \
    --reuse-values \
    --values values.yaml
```
{% endraw %}


### 1.4 ClusterIssuer 및 Certificate 생성


{% raw %}
```bash
# clusterissuer-lets-encrypt-production.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $EMAIL_ADDRESS
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - dns01:
        azureDNS:
          resourceGroupName: $AZURE_DEFAULTS_GROUP
          subscriptionID: $AZURE_SUBSCRIPTION_ID
          hostedZoneName: $DOMAIN_NAME
          environment: AzurePublicCloud
          managedIdentity:
            clientID: $USER_ASSIGNED_IDENTITY_CLIENT_ID
            
envsubst < clusterissuer-lets-encrypt-production.yaml | kubectl apply -f  -

# certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: www
spec:
  secretName: www-tls
  privateKey:
    rotationPolicy: Always
  commonName: www.$DOMAIN_NAME
  dnsNames:
    - www.$DOMAIN_NAME
  usages:
    - digital signature
    - key encipherment
    - server auth
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
    
envsubst < certificate.yaml | kubectl apply -f -

# 생성된 각 오브젝트 상태 확인 (인증서 발급까지 약 1분 소요)
kubectl describe clusterissuer letsencrypt-production
cmctl status certificate www
cmctl inspect secret www-tls
```
{% endraw %}


### 1.5 샘플 웹앱 배포


{% raw %}
```bash
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloweb
  labels:
    app: hello
spec:
  selector:
    matchLabels:
      app: hello
      tier: web
  template:
    metadata:
      labels:
        app: hello
        tier: web
    spec:
      containers:
      - name: hello-app
        image: us-docker.pkg.dev/google-samples/containers/gke/hello-app-tls:1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8443
        volumeMounts:
          - name: tls
            mountPath: /etc/tls
            readOnly: true
        env:
          - name: TLS_CERT
            value: /etc/tls/tls.crt
          - name: TLS_KEY
            value: /etc/tls/tls.key
      volumes:
      - name: tls
        secret:
          secretName: www-tls
          
kubectl apply -f deployment.yaml

# service.yaml
apiVersion: v1
kind: Service
metadata:
    name: helloweb
    annotations:
        service.beta.kubernetes.io/azure-dns-label-name: $AZURE_LOADBALANCER_DNS_LABEL_NAME
spec:
    ports:
    - port: 443
      protocol: TCP
      targetPort: 8443
    selector:
        app: hello
        tier: web
    type: LoadBalancer
    
envsubst < service.yaml | kubectl apply -f -

# LoadBalancer 외부 IP와 매핑되는 CNAME 레코드 등록
az network dns record-set cname set-record \
    --zone-name $DOMAIN_NAME \
    --cname $AZURE_LOADBALANCER_DNS_LABEL_NAME.$AZURE_DEFAULTS_LOCATION.cloudapp.azure.com \
    --record-set-name www
    
# www.$DOMAIN_NAME 조회
dig www.$DOMAIN_NAME A
```
{% endraw %}


### 1.6 HTTPS 통신 및 인증서 확인

- `curl -v https://www.$DOMAIN_NAME`

	![1](/assets/img/2024-04-29-AKS-내-인증서-자동-갱신-환경-구성하기-(feat.-let’s-encrypt-+-cert-manager).md/1.png)

- 브라우저에서 https://www.<도메인 명> 접속 후 확인

	![2](/assets/img/2024-04-29-AKS-내-인증서-자동-갱신-환경-구성하기-(feat.-let’s-encrypt-+-cert-manager).md/2.png)

