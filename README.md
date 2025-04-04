<img src="https://capsule-render.vercel.app/api?type=waving&color=00C3FF&height=150&section=header" width="1000" />

<div align="center">
<h1 style="font-size: 36px;">📡 EKS & Prometheus 연동 모니터링</h1>
</div>
</br>



## 🙆🏻‍♂️ 팀원

#### 팀명 : 우리FISA 4기 클라우드 엔지니어링 Gimme-Jjkkumi조


|<img src="https://avatars.githubusercontent.com/u/150774446?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/165532198?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/74342019?v=4" width="150" height="150"/>|<img src="https://avatars.githubusercontent.com/u/107902336?v=4" width="150" height="150"/>|
|:-:|:-:|:-:|:-:|
|김예진<br/>[@yeejkim](https://github.com/yeejkim)|김소연<br/>[@ssoyeonni](https://github.com/ssoyeonni)|나원호<br/>[@CooolRyan](https://github.com/CooolRyan)|김대연<br/>[@dyoun12](https://github.com/dyoun12)|

---

# 🌱 프로젝트 개요: EKS & Prometheus 연동 모니터링

- 본 프로젝트는 AWS EKS 클러스터의 메트릭 정보를 외부 Prometheus 인스턴스에서 수집 및 시각화하기 위해 구성한 과정입니다.
- 비용 최적화를 위해 Prometheus는 자체 서버에 설치하고, EKS는 필요한 Exporter를 노출하여 외부에서 접근 가능한 구조로 설계했습니다.


<br>

## 🎯 주요 목표

- ✅ AWS EKS에서 노드 및 파드 수준의 리소스 메트릭 수집
- ✅ 외부(로컬 서버)의 Prometheus 인스턴스에서 수집
- ✅ Cloud 비용 최소화를 고려한 Exporter 중심 아키텍처 설계

<br>

## ⚙️ 구성 요소

| 구성 요소             | 위치            | 설명                                                                        |
|----------------------|------------------|------------------------------------------------------------------------------|
| **EKS 클러스터**      | AWS              | 모니터링 대상 Kubernetes 클러스터                                           |
| **Node Exporter**     | EKS 노드         | EC2 인스턴스의 CPU, Memory, Disk 등 리소스 사용량을 Prometheus에 전달        |
| **Kube State Metrics**| EKS 내부         | 쿠버네티스 리소스 상태 (파드, 디플로이먼트 등) 메트릭 수집                   |
| **Prometheus**        | 로컬 서버 or VM  | 외부 메트릭 수집기. EKS Exporter에서 전달되는 메트릭을 수집하고 쿼리 지원     |
| **Helm**              | 클러스터 제어 PC | Prometheus Stack 및 Exporter 배포 자동화 및 설정 관리                        |

<br>

## 💡 모니터링 구성 시 비용 최적화 고려사항 (DevOps 관점)

본 프로젝트에서는 AWS 환경 내 모니터링 서버를 설계함에 있어, **데이터 전송량 기반의 비용 분석을 선행적으로 수행**하여, DevOps 관점에서 운영 비용 절감을 고려하였습니다.  
아래는 exporter 수와 시간당 발생하는 전송량, 예상 비용에 대한 시뮬레이션 결과는 다음과 같습니다.

### **EC2 기반 Prometheus 비용**

| 인스턴스      | 온디맨드 비용 시간당 | vCPU | Ram  | 스토리지   | 네트워크 성능  |
| --------- | ----------- | ---- | ---- | ------ | -------- |
| t3a.small | USD 0.0234  | 2    | 2GiB | EBS 전용 | 최대 5기가비트 |

### **Data Transfer 비용**

| 유형         | 1G당 비용    |
| ---------- | --------- |
| EC2 -> 인터넷 | USD 0.126 |
| 같은 VPC 내   | USD 0     |

### **비용 역전 지점**
> `t3a.small` 인스턴스 시간당 온디맨드 비용 = 시간당 190Mb 의 패킷이 외부로 송신

| 구성 요소             | 인터벌         | 시간당 전송량  | 비고               |
| ----------------- | ----------- | -------- | ---------------- |
| `mysqld-exporter` | 5초 (720회/h) | 20.18 MB | 회당 28,030 바이트 기준 |
| Grafana 대시보드 1개   | 1분 (60회/h)  | 3.56 MB  | 회당 59,267 바이트 기준 |

본 분석은 다음과 같은 조건에서 산정되었습니다:
- Region/VPC/EKS Cluster 수: 1
- Exporter 전송 주기: 5초
- Grafana Pull Interval: 1분

또한 시간당 전송량을 측정하기 위해서 port를 기준으로 수신된 패킷의 사이즈를 측정하여 1회 스크래핑 당 수신한 데이터량을 계산, 1시간동안 스크래핑 횟수를 고려했습니다.

### **계산 식**
#### $(20.18 * N) > 190 + (3.56 * N)$

> 결과적으로, 위 식을 만족하는 N의 최소값은 8로 exporter가 8개 이상일 경우 시간당 데이터 교환 비용이 EC2 온디맨드 비용을 초과하게 되며, 이때부터는 로컬에서 Prometheus 서버를 운영하는 것 대신 EC2 인스턴스를 기반으로 Prometheus를 운영하는 것이 더 경제적으로 볼 수 있습니다.

이는 단순한 리소스 스펙 선정이 아닌, 데이터 흐름에 따른 비용 최적화와 DevOps 운영 전략을 일부 경험할 수 있었습니다.

<br>

## #️⃣ 실습 과정

### 1️⃣ Prometheus 설치 
- Kubernetes에서 Helm(패키지 매니저)를 통해 Promethues 설치 진행

1. Promethues Repository 추가
   ```bash
   $ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   $ helm repo update
    ```

2. Promethues 설치
   ```bash
   # Promethues 설치
   $ helm install prometheus prometheus-community/kube-prometheus-stack -n monitoring

   # 설치 확인
   $ kubectl get all -n monitoring
    ```
   ![image](https://github.com/user-attachments/assets/9cffc7cb-55ad-4356-a092-616674b18d95)


3. Grafana 비밀번호 확인
   ```bash
   $ kubectl --namespace monitoring get secrets prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

    --- 출력 ---
    prom-operator
    ```

4. Grafana Instacne 접속 위한 포트포워딩
   ```bash
    # Grafana 파드 주소값 확인
   $ export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus" -oname)

    # 3000번 포트 포트포워딩 진행
    $ kubectl --namespace monitoring port-forward $POD_NAME 3000
   
    ```
    ![image](https://github.com/user-attachments/assets/0121fdea-32a1-4df6-a070-885365037f99)


5. localhost:3000 접속 (admin 로그인)
     ![image](https://github.com/user-attachments/assets/7d43c222-f96d-4b1a-ae91-5c8896fec78e)




### 2️⃣ AWS CLI 설치 

```bash
# AWS CLI 파일 다운로드
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# 다운로드한 zip 파일 압축 풀기
$ unzip awscliv2.zip

# AWS CLI 파일 설치 실행
$ sudo ./aws/install

# 설치 확인
# 개인 키값 입력
$ aws configure
```




### 3️⃣ EKS 설치 

```bash
# 최신 버전의 eksctl(EKS 클러스터 생성 도구)을 GitHub에서 다운로드하고 압축 해제
$ curl --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# 압축 해제된 eksctl 실행 파일을 시스템 경로(/usr/local/bin)로 이동 → 터미널 어디서나 실행 가능
$ sudo mv -v /tmp/eksctl /usr/local/bin

# eksctl이 정상적으로 설치되었는지 버전 확인
$ eksctl version

# 이름: ce-05-myeks, 버전: 1.26, 리전: 서울 클러스터 생성
$ eksctl create cluster --name ce-05-myeks --version 1.26 --region ap-northeast-2

# 현재 사용 중인 Kubernetes context 확인
$ kubectl config current-context

#  EKS 클러스터 목록 확인
$ aws eks list-clusters --region ap-northeast-2

# 현재 연결된 클러스터의 노드 확인
$ kubectl get nodes
```



### 4️⃣ EKS와 로컬의 Prometheus 연동

1. EKS에 Prometheus Exporter 설치
   
   - 로컬 서버의 Prometheus가 EKS의 Exporter를 통해 리소스 수집할 수 있도록 설정


    ```bash
    # Helm에 Prometheus 관련 차트 저장소를 추가
    $ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    
    # 저장소 목록 업데이트
    $ helm repo update
    
    # Kubernetes 리소스들의 상태 정보를 Prometheus가 수집할 수 있도록 도와주는 Exporter 설치
    $ helm install ksm prometheus-community/kube-state-metrics -n monitoring --create-namespace
    
    # 각 노드(서버)의 CPU, 메모리, 디스크 사용량 등의 시스템 정보를 Prometheus에 전달해주는 Exporter 설치
    $ helm install node-exporter prometheus-community/prometheus-node-exporter -n monitoring
    
    ```

2. Exporter 설치 확인
   ```bash
    # monitoring 네임스페이스에 존재하는 서비스 목록 확인
    $ kubectl get svc -n monitoring
    
    # LoadBalancer로 외부 접근이 가능하도록 변경
    $ kubectl patch svc ksm-kube-state-metrics -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
    $ kubectl patch svc node-exporter-prometheus-node-exporter -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
    
    # TYPE이 LoadBalancer로 바뀌었는지 확인
    $ kubectl get svc -n monitoring
    ```

  <details>
    <summary>변경 결과</summary>

  - ClusterIP인 경우
    
      ```bash
      ubuntu@myserver01:~$ kubectl get svc -n monitoring
      NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
      ksm-kube-state-metrics                   ClusterIP   10.100.203.241   <none>        8080/TCP   83s
      node-exporter-prometheus-node-exporter   ClusterIP   10.100.232.85    <none>        9100/TCP   79s
      ```

  - LoadBalancer 변경 결과
    - 외부 접근이 가능한 EXTERNAL-IP가 생성된 것 확인 가능
    ```bash
    ubuntu@myserver01:~$ kubectl get svc -n monitoring
    NAME                                     TYPE           CLUSTER-IP       EXTERNAL-IP                                                                   PORT(S)          AGE
    ksm-kube-state-metrics                   LoadBalancer   10.100.203.241   ad266ec3c214e4a18a85b9394ccc508e-743863240.ap-northeast-2.elb.amazonaws.com   8080:31208/TCP   3m50s
    node-exporter-prometheus-node-exporter   LoadBalancer   10.100.232.85    a0b331d93a2a14b6a8796e2f342209e6-337083271.ap-northeast-2.elb.amazonaws.com   9100:30446/TCP   3m46s
    ```
  </details>


### 5️⃣ EKS와 Prometheus 연동

1. Context 확인 및 변경
  
    🧭 Context란? <br>
  
    - Kubernetes에서 **어떤 클러스터에 연결할지, 어떤 사용자로 접근할지, 어떤 네임스페이스를 기본으로 사용할지** 정의한 설정
    
    - `kubectl` 명령은 현재 활성화된 context를 기준으로 실행
    - 멀티 클러스터 환경에서는 context를 전환하여 다양한 클러스터 관리 가능
  
  ---
  
  
  - 위 과정까지 EKS에서 작업했기 때문에, 로컬에 위치한 Prometheus를 설정하기 위해 context 변경 필요

   ```bash
    # 현재 내 context 확인
    $ kubectl config current-context
    
    # 사용 가능한 context 목록 확인
    $ kubectl config get-contexts
    
    # context 변경
    $ kubectl config use-context <context-name>
   ```




2. `prometheus-values.yaml` 파일 수정
  - EKS의 메트릭을 수집할 수 있도록 exporter의 external IP 설정

  ```bash
  # 현재 Prometheus 설정을 export해서 prometheus-values.yaml 파일 만들기
  $ helm get values prometheus -n monitoring -o yaml > prometheus-values.yaml
  
  # prometheus-values.yaml 파일에 추가할 설정
  # targets는 로드밸런스 EXTERNAL-IP 주소 사용
  prometheus:
    prometheusSpec:
      additionalScrapeConfigs:
        - job_name: 'eks-kube-state-metrics'
          static_configs:
            - targets:
                - ad266ec3c214e4a18a85b9394ccc508e-743863240.ap-northeast-2.elb.amazonaws.com:8080
        - job_name: 'eks-node-exporter'
          static_configs:
            - targets:
                - a0b331d93a2a14b6a8796e2f342209e6-337083271.ap-northeast-2.elb.amazonaws.com:9100
  
  # prometheus-values.yaml 파일을 기반으로 설정을 업데이트
  $ helm upgrade prometheus prometheus-community/kube-prometheus-stack \
    -n monitoring \
    -f prometheus-values.yaml
  ```


3. Prometheus 포트포워딩 & 서버 접속
   - 윈도우 서버에서 접속 가능하도록 포트포워딩 진행

    ![image](https://github.com/user-attachments/assets/27e8c585-dfae-4cca-95fd-b4a9848b8fd6)

  - 윈도우 서버에서 확인 <br>
    `http://localhost:9090/targets`

    <img src="https://github.com/user-attachments/assets/d3c24769-e05c-4171-989f-9986c026a0c9" width="700">

    - EKS의 항목들이 보이면 성공
      - eks-kube-state-metrics
      - eks-node-exporter

### *️⃣ 리소스 자원 종료
- EKS 사용 후 리소스 비용 관리를 위한 자원 종료 

  ```bash
  $ eksctl delete cluster --name ce-05-myeks --region ap-northeast-2
  ```



## 🔗 Reference

- [AWS EKS 워크숍 가이드 (공식)](https://catalog.us-east-1.prod.workshops.aws/workshops/46236689-b414-4db8-b5fc-8d2954f2d94a/ko-KR/eks/10-install)
- [AWS CLI 설치 가이드 (공식 문서)](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/getting-started-install.html)

