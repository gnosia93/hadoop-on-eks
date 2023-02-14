### 1. ec2 키페어 생성 ###

https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/create-key-pairs.html 를 참고하여 aws-kp 라는 이름의 키페이를 생성한다.

### 2.kubectl 설치하기 ###

```
$ curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.7/2022-10-31/bin/darwin/amd64/kubectl
$ chmod +x ./kubectl
$ mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
$ kubectl version --short --client
```

### 3. eksctl 설치하기 ###

```
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
$ brew upgrade eksctl && { brew link --overwrite eksctl; } || { brew tap weaveworks/tap; brew install weaveworks/tap/eksctl; }
$ eksctl version
0.127.0
```


### 4. eks 클러스터 생성 ###

cluster.yaml 파일을 생성한다.
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: hadoop-on-eks
  region: ap-northeast-2

nodeGroups:
  - name: hadoop-ng-x86 
    instanceType: m6i.2xlarge
    minSize: 2
    maxSize: 5
    desiredCapacity: 3
    volumeSize: 80
    ssh: # use existing EC2 key
      publicKeyName: aws-kp
```

아래 명령어를 이용하여 새로운 클러스터를 하나 생성한다.
```
$ eksctl create cluster -f cluster.yaml

...
2023-02-04 20:35:44 [✔]  EKS cluster "hadoop-on-eks" in "ap-northeast-2" region is ready
```
VPC 를 비롯한 서브넷, 시큐리티 그룹 등과 같은 AWS 리소스가 자동으로 생성된다.

[참고] 클러스터 삭제 명령어
```
$ eksctl delete cluster -f cluster.yaml
```

### 5. 클러스터 생성 확인 ###
```
$ kubectl config get-contexts
CURRENT   NAME                                             CLUSTER                                 AUTHINFO                                         NAMESPACE
          docker-desktop                                   docker-desktop                          docker-desktop
*         name-of-account@spark-on-eks.ap-northeast-2.eksctl.io   spark-on-eks.ap-northeast-2.eksctl.io  

$ kubectl get nodes
NAME                                                STATUS   ROLES    AGE     VERSION
ip-192-168-22-60.ap-northeast-2.compute.internal    Ready    <none>   3m15s   v1.24.9-eks-49d8fe8
ip-192-168-33-80.ap-northeast-2.compute.internal    Ready    <none>   4m9s    v1.24.9-eks-49d8fe8
ip-192-168-35-245.ap-northeast-2.compute.internal   Ready    <none>   3m17s   v1.24.9-eks-49d8fe8
```

### 6. 성능 메트릭 및 로그 수집 ###
* https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html

성능 메트릭 및 켄테이너 로그를 수집하기 위해서는 아래와 두 단계에 걸쳐 설정 작업이 필요하다 

- EC2 서비스 메뉴에서 EKS 워커 노드 하나를 선택한 다음, attach 된 IAM Role 을 찾아 CloudWatchAgentServerPolicy 정책을 추가한다.  
- cloudwatch 및 flient bit 에이전트를 워커로드에 데몬셋으로 설치한다.  

클러스터, 노드 및 파드 별로 메트릭 정보와 로그 정보를 수지하기 위해서 cloudwatch 에이전트와 C 언어로 강량화 구현된 fluent bit 에이전트를 파드 형태로 설치할 것이다.  
```
$ ClusterName=hadoop-on-eks
RegionName=ap-northeast-2
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights\
/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring\
/quickstart/cwagent-fluent-bit-quickstart.yaml | sed 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' | kubectl apply -f - 

$ % kubectl get namespace amazon-cloudwatch
NAME                STATUS   AGE
amazon-cloudwatch   Active   6h52m

$ kubectl get pods -n amazon-cloudwatch

NAME                     READY   STATUS    RESTARTS   AGE
cloudwatch-agent-85fcl   1/1     Running   0          6h46m
cloudwatch-agent-dsqn6   1/1     Running   0          6h46m
cloudwatch-agent-gdsgc   1/1     Running   0          6h46m
cloudwatch-agent-kbnl8   1/1     Running   0          6h46m
cloudwatch-agent-mmbxd   1/1     Running   0          6h46m
cloudwatch-agent-z4tnc   1/1     Running   0          6h46m
fluent-bit-46pfl         1/1     Running   0          6h46m
fluent-bit-6tcnp         1/1     Running   0          6h45m
fluent-bit-hfbfs         1/1     Running   0          6h46m
fluent-bit-qvshq         1/1     Running   0          6h45m
fluent-bit-v67m2         1/1     Running   0          6h45m
fluent-bit-vg2x5         1/1     Running   0          6h46m

$ kubectl describe pod fluent-bit-46pfl -n amazon-cloudwatch

$ kubectl describe configmaps fluent-bit-cluster-info -n amazon-cloudwatch
```

cloudwatch 의 container insight 메뉴에서 메트릭을 확인한다. 
![](https://github.com/gnosia93/spark-on-eks/blob/main/images/eks-container-insight.png)

cloudwatch 의 logs 하단의 log groups 메뉴에서 어플리케이션 로그를 확인한다. 
![](https://github.com/gnosia93/spark-on-eks/blob/main/images/eks-log-groups.png)



## 참고자료 ##
* https://eksctl.io/usage/creating-and-managing-clusters/
* https://stackoverflow.com/questions/64059038/the-results-of-aws-eks-list-nodegroups-and-eksctl-get-nodegroups-are-inconsi
* https://findstar.pe.kr/2022/08/21/which-instance-type-is-right-for-EKS/
* https://lannex.github.io/blog/2020/EFK-Fluentd-Fluentbit/
