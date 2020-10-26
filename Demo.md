
## 환경 설명

### 작업하기 편리하기 위해서 public subnet에 EC2를 위치시킨거지 istio의 통신을 
private IP로된 G/W로 한다.



```
~/Dev/istio-lab  kubectl config current-context                                                          
arn:aws:eks:ap-northeast-2:090952272468:cluster/eksworkshop-eksctl

 ~/Dev/istio-lab  istioctl version                                                                       
client version: 1.4.9
control plane version: 1.4.9
data plane version: 1.4.9 (8 proxies)


```

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