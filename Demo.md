AWSome Builder Program 발표 시나리오
========

## 0. Requirement 정리

- Cloud Bursting
1) Autoscaling  
1.1 HPA   
1.2 Cluster Autoscaler

- Application Logging

- GPU 활용

- Security 

### 0.1 목차



## 1. Architecture (환경설명)   
-----


- On-Prem, EKS  
1.1 Network   
1.1.1 Peering Connection
- Istio  
1.1 OnPrem에는 민감 정보와 부하가 적게 발생하는 서비스만 두고 나머지 서비스는 클라우드로 옮긴다.

- 시나리오 1  
부하 발생시켜서 On-Prem에 CloudWatch Agent 가 깔렸다고 가정하고 (*설치 방법 설명* ) 부하가 발생하면, CloudWatch가 Metrics 정보를 가져와서 Alarm을 생성하고 SNS를 발송하면 특정 토픽이 발생했다는 Trigger를 이용해 Lambda가 실행되어 Traffic을 On-Prem이 아닌 Cloud쪽으로 돌리게 된다.   

그렇지만 민감정보가 있는 서비스는 남겨 놓고, 부하를 많이 차지하는 서비스만 work load만 클라우드로 옮긴다. 

- 시나리오 2   
부하 발생시 HPA CA 시나리오

- 시나리오 3  
어플리케이션의 로깅은 여러가지 선택지가 있다. 기존에 On-Prem에서 사용중인 EFK를  1) 파드로 구성 2) Managed ES 3) Cloudwatch log 4) S3 에 저장해서 Athena로 분석하는 방법이 있다. 

오늘은 주로 ES 프로비져닝 방법과 CloudWatch에 모인 로그를 보여드리고, Athena로 쿼리 하는 방법을 보여드리겠다.

- 시나리오 4
AI/ML 서비스 
1) GPU Provisioning
2) Sagemaker 


## Scenario : 1. Cloud Bursting
1.1 환경 설명   
1.1.1 k8s 클러스터 설명
 - 작업하기 편리하기 위해서 public subnet에 EC2를 위치시킨거지 istio의 통신을 private IP로된 G/W로 한다.   

1.1.2 istio 설정 설명  
 - istio 버전 1.4.9
 - control plane  
 - Envoy Proxy 가 auto-inject 되도록 설정

2.1 흐름 설멍


### 시연
1.1 웹페이지 보여주기
 - hip 한 아이템들을 모아서 파는 싸이트
 - 전체 서비스 보여주기
 - 환율변경, 추천, 광고

1.2 클라우드 콘솔
 - EKS 콘솔
 - 생성 방법 설명   
   1. 클러스터 IP Range 
   2. Control Plane Logging - Managed 이기 때문에 직접 접근 불가  
 - 클러스터 생성
 - Node Group 생성

 > EKS의 여러개의 노드 그룹을 가질 수 있으며 하나의 노드 그룹은 1) 동일한 인스턴스 타입, 2) 같은 AMI, 3) 같은 EKS node IAM Role 이어야 하면 하나의 ASG이 할당된다. 
 > 각 노드 그룹에는 각각 다양한 OS와 인스턴스 타입의 그룹이 올 수 있다. 
 > 오토스케일링은 각각의 ASG Policy에 따라 늘어나며 늘어난 Resource의 Scheduling은 k8s 스케줄러가 한다.  

 1.3 VPC  
 - VPC Peering 상황
  1. Requester VPC Accepter VPC
  2. Route Tables

1.4 Pod Deploy 환경 보여주기
 - 

 - Question : Kiali의 Multi Cluster 지원 여부
```shell
# 설정 : Account : 090952272468
export ACCOUNT_ID=090952272468
export MAIN=arn:aws:eks:ap-northeast-2:090952272468:cluster/AWS-Cluster
export REMOTE=arn:aws:eks:ap-northeast-2:090952272468:cluster/Octank-Cluster

# $MAIN $REMOTE
$ echo "Primary Cluster : $MAIN,  Secondary Cluster : $REMOTE"


#istio version check 
$ istioctl version                    

# pod deployment 확인 : kubeopsview 적용 검토 필요
$ kubectl get pod -n hipster-app --context $MAIN

$ kubectl get pod -n hipster-app --context $REMOTE

# 2개의 Ingress Gateway
$ kubectl get gateway --all-namespaces --context $MAIN

#Proxy 상황
istioctl proxy-status --context $MAIN
```

1.5 kiali 로 환경 보여주기  
 - Service Graph  

```shell
#별도의 shell에서 
 $ istioctl dashboard kiali --context $MAIN

```
  1. Graph
  2. Service
  3. App Metric
  4. 

1.6 CPU 부하 발생

  1. EC2 Console 2개 
    - load.py
    - htop 

  2. CloudWatch Alarm 설정

  3. lambda 설정 설명

```shell
# 현황 
 $ kubectl describe  vs recommend-split -n hipster-app --context $MAIN 
# 또는 kiali 의 서비스 참조
```

## Scenario : 2. Auto-scaling 

1. HPA
Kubelet 과 통합된 cAdvisor가 각 컨테이너의 메트릭 정보를 수집해서 메트릭 서버(k8s Addon)에 전달하고 API 서버를 통해서 System metrics를 기반으로 HPA 가 스케일을 판단하게되고 Deployment 의 Replicas 를 변경하게 된다. 

```shell
kubectl describe hpa

Reference:                                             Deployment/php-apache
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (1m) / 50%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       1 current / 1 desired
```
참조 : https://blog.naver.com/alice_k106/221521978267

  - kube-ops-view 상에서 시연
  - 샘플 앱 -> 부하발생 (https://www.eksworkshop.com/beginner/080_scaling/test_hpa/)



2. Cluster Autoscaler

- 클러스터 오토스케일러는 클라우드 벤더마다 조금씩 다른데 AWS 는 AutoScalingGroup 연결되어 있다. 
- Scheduler 가 할당하지 못한 자원이 발생할때 즉 pending Resource 가 발생할때 실행된다.

- 실연은 예제를 그대로 하되 

###   Question
 - 오토스케일러가 ASG에게 명령하나? Desired Capacity를 바꾸라고?
 - 원래는 ASG Policy가 비어있다. 

 3. Spot Instance 
  - 노드 그룹 추가 
  - 위의 예시에서 처럼 부하발생
  - ToDo : Node Selector 예시 추가


 4. Gpu Node Group은   
   - Console 에서 Node Group을 추가할 수 있는 화면 시연




> node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,
k8s.io/cluster-autoscaler/eksworkshop-eksctl
Node 그룹을 정의할 수 있다. 
여기에 Spot Instance
GPU Type을 정의할 수 있다.
궁금중 여러개를 모아놨을때 AutoScaling은 어떻게 반응하는가?


>- Spot Instance
2분전에 발생 
Spot Instances are sent an interruption notice two minutes ahead to gracefully wrap up things
예제는 그냥 Spot 에 올려놓는것만 있다. 여러 노드 타입을 노드 그룹으로 묶는 방법은?


## Secnario : 3 Logging & Monitoring & Tracing

- Logging
 1. 컨트롤 플레인정보은 AWS가 관리하기 때문에 Cloudwatch 로 로그를 본다.
 2. 데이터 플레인은 FluentD 나 FluentBit을 통해서 로그를 수집(Processing) 1) EFK 2) S3, Athena 3) Cloudwatch (데모)

- Monitoring
 1. Prometheus + Grafana 조합이 가능하나
 2. CloudWatch Insight를 CPU, Memory, disk and Network, container restart failure 소개한다. 

 - Tracing
 1. Istio의 Jaeger가 가능하며 (예시 BookInfo)
 2. X-ray 시연

 - 메트릭도 SideCar가 pod의 네임스페이스를 공유하기 때문에 서비스가 올라가 있는 놈들의 정보를 그대로 끌어다가 보여준다.



## Scenario 4 GPU Sagemaker


- 
- 프로비저닝이 가능하다. (예제 올리기)  - 1시간
- kubectl 로 명령을 내려서 실행하며, Traingjob을 실행하고 종료되면 리소스는 반환된다.

- Oregon
kubectl apply -f train.yaml
kubectl describe trainingjob xgboost-mnist




참조영상 : https://www.youtube.com/watch?v=6sogVHw9jZ4

exec role : Inho-sagemaker-exec-role
OIDC role : inho-sagemaker-role






Event Engine
export AWS_DEFAULT_REGION=us-east-2
export AWS_ACCESS_KEY_ID=ASIAXF7JLH2NSJAALWYW
export AWS_SECRET_ACCESS_KEY=e2wxf2zO59JTGyT+NljOw52OalLUITueDbpWpdEP
export AWS_SESSION_TOKEN=IQoJb3JpZ2luX2VjELH//////////wEaCXVzLWVhc3QtMSJHMEUCIQCJzZq0/DkBsBX2JjaUi1IjXVpkDrBzfMd890SH9x5aEAIgMYftwrl5h4YIrtXp7A80perZOIesWIUYHRTps61iiBYqoAII+f//////////ARABGgw0OTM4NzM3NDE0NjciDE6yaRI34l+pSav7cCr0ARLmU5KZGOh4iIpHQ0u8maX5rHZPdjPSgqb1yjHXep9y2CZuVIiHPqaxXriwJxgmTaQM3JPZkhRPmFLEwVEOuCsePycA5fzj7Y2QNYBH4Pez+z9gR9TnOt4/lB1lgK31rQ4Ccg0/HMH8UhwF4nLIAdlKcY+pOhBRyvK+P2jilosEcdYc68Az1tOuN1fJwJUzv31xJytBpBkRQvToFtrFwW+WimFc12DpkPj8RQNvZy7iUtMSRt1/4N8/sX+91byOcq127rZD1LSODOEQWzbI51om6Fo8tHki86NH+cVusNOWyZSA/R3ii38M2TuYJbpiy7gIva4w1Mjd/AU6nQEElpkp7VRdFLSoR3TTaoAgzh0az+q1q1sKkafmvDnig8+5yuNbv6AvOWx3qUehq9x2TteijpxhWqDp1F+iOf2L5Cny+ITP8C9Q7sGlxZM4uiUkfbYoBmoQRIhF3BotH1FFFaKeYDYFkRswICjJmq75XDaTBp5H92yBTHKJe5Xp490jZqsO1Y7N0Pp51TMNlTpNtXtKHQRN2mXFgoo3
``` shell
## sagemaker operator가 떠 있고 

lab-admin:~/environment/sagemaker $ kubectl get pod -n sagemaker-k8s-operator-system
NAME                                                         READY   STATUS    RESTARTS   AGE
sagemaker-k8s-operator-controller-manager-5f6b7bb74f-rcjq7   2/2     Running   0          84m

## custom Resource Type 으로 구동중이다. 
lab-admin:~/environment/sagemaker $ kubectl get crd | grep sagemaker
batchtransformjobs.sagemaker.aws.amazon.com           2020-10-26T13:23:37Z
endpointconfigs.sagemaker.aws.amazon.com              2020-10-26T13:23:38Z
hostingautoscalingpolicies.sagemaker.aws.amazon.com   2020-10-26T13:23:38Z
hostingdeployments.sagemaker.aws.amazon.com           2020-10-26T13:23:38Z
hyperparametertuningjobs.sagemaker.aws.amazon.com     2020-10-26T13:23:38Z
models.sagemaker.aws.amazon.com                       2020-10-26T13:23:38Z
processingjobs.sagemaker.aws.amazon.com               2020-10-26T13:23:38Z
trainingjobs.sagemaker.aws.amazon.com                 2020-10-26T13:23:38Z
```


alpha.eksctl.io/cluster-name=eksworkshop-eksctl,alpha.eksctl.io/nodegroup-name=nodegroup,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=m5.large,beta.kubernetes.io/os=linux,eks.amazonaws.com/nodegroup-image=ami-025592e84db381916,eks.amazonaws.com/nodegroup=nodegroup,eks.amazonaws.com/sourceLaunchTemplateId=lt-0afdffaf71e32ea49,eks.amazonaws.com/sourceLaunchTemplateVersion=1,failure-domain.beta.kubernetes.io/region=ap-northeast-2,failure-domain.beta.kubernetes.io/zone=ap-northeast-2a,kubernetes.io/arch=amd64,kubernetes.io/hostname=ip-192-168-23-156.ap-northeast-2.compute.internal,kubernetes.io/os=linux,node.kubernetes.io/instance-type=m5.large,topology.kubernetes.io/region=ap-northeast-2,topology.kubernetes.io/zone=ap-northeast-2a


### Deploy LoadGen
```
kubectl apply -f istio-workshop-labs/multi-network/remote/loadgenerator.yaml -n hipster-app --context $REMOTE 
```


### Secondary Ingress 설치
```
kubectl apply -f second-ingress.yaml

```


### Istio Diagnose
```
istioctl x describe pod <podname> -n hipster-app