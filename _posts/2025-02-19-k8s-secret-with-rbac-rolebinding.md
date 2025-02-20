---
title: 쿠버네티스 시크릿에 권한 설정 하기
categories: kubernetes
tags:
  - kubernetes
  - k8s
  - secret
published: true
---

[이전 글](https://jinwoongkim.net/kubernetes/k8s-secret/)에서 쿠버네티스에서 시크릿을 만들고 사용하는 방법에 대해서 다루어 보았다.

다만, 이름이 무색하게 이렇게 시크릿을 만들면 누구나 조회 수정이 가능하다.

**즉, 시크릿이 전혀 시크릿(?)하지 않은 상황 ㅎㅎ**

이럴때 ETCD에서 시크릿을 암호화하는 등 보안정책을 적용 할 수도 있지만, 보다 단순하게
쿠버네티스의 RBAC
과 Rolebinding을 이용해서 특정 네임스페이스만 시크릿을 조회 수정하게 할 수 있다.

참고로 RBAC이란? 쿠버네티스에서... 리소스에 권한 설정을 하는 룰이라고 것이고
이렇게 만든 루을 Rolebinding을 통해 적용한다고 생각하면 된다.

### **`dp` 네임스페이스 사용자에게 `log-analyzer` Secret 조회 & 수정 권한 부여**하는 방법

### 1. Role로 권한 정의

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dp
  name: secret-editor
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["log-analyzer"]  # 특정 Secret만 적용
    verbs: ["get", "list", "watch", "update", "patch", "delete"]
```

### 2. RoleBinding으로 권한 부여
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: dp
  name: secret-editor-binding
subjects:
  - kind: User
    name: target-user  # 권한을 부여할 사용자 (변경 가능)
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: secret-editor  # 위에서 만든 Role을 참조
  apiGroup: rbac.authorization.k8s.io
```


### **RBAC 개념 비교 테이블**

| 개념                     | 적용 범위     | 권한 정의        | 대상 지정                     | 네임스페이스 제한        |
| ---------------------- | --------- | ------------ | ------------------------- | ---------------- |
| **RBAC**               | 클러스터 전체   | 접근 제어 시스템    | X (직접 적용 안 됨)             | X                |
| **Role**               | 특정 네임스페이스 | 리소스 접근 권한 정의 | X                         | ✅ 네임스페이스 한정      |
| **RoleBinding**        | 특정 네임스페이스 | X            | ✅ Role을 사용자/그룹에 연결        | ✅ 네임스페이스 한정      |
| **ClusterRole**        | 클러스터 전체   | 리소스 접근 권한 정의 | X                         | ❌ (모든 네임스페이스 포함) |
| **ClusterRoleBinding** | 클러스터 전체   | X            | ✅ ClusterRole을 사용자/그룹에 연결 | ❌ (모든 네임스페이스 포함) |

/* Binding이 필요한 이유? 하나의 롤을 만들고 여러 네임스페이스에 바인딩 가능

	Role은 권한 정의
	RoleBinding은 권한 적용
/* ClusterRole 은 클러스터 전체 적용이라, 네임스페이스와 무관한 리소스(노드, PV, SC)도 관리 가능



`kubectl get secret`을 `abc` 네임스페이스의 사용자 권한으로 실행
kubectl get secret -n dp --as=system:serviceaccount:abc:default











---
업무상 쿠버네티스(K8s)를 많이 쓰는데 `secret` 을 직접 만들어 써본적은 없었다.

이번에 만들일이 생겨서 `secret` 을 어떻게 만들고, 사용하는지 정리해봤다.

사용자가 `kubectl` 혹은 k8s `secret`을 만들 수 있는 K8s 툴 환경이 구성되어있다고 가정하였다.
{: .notice--warning}

### K8s Secret 이란?
K8s의 문서를 보면 아래와 같이 서술되어 있다.

>시크릿은 암호, 토큰 또는 키와 같은 소량의 중요한 데이터를 포함하는 오브젝트이다. 이를 사용하지 않으면 중요한 정보가 [파드](https://kubernetes.io/ko/docs/concepts/workloads/pods/) 명세나 [컨테이너 이미지](https://kubernetes.io/ko/docs/reference/glossary/?all=true#term-image)에 포함될 수 있다. 시크릿을 사용한다는 것은 사용자의 기밀 데이터를 애플리케이션 코드에 넣을 필요가 없음을 뜻한다.

[출처](https://kubernetes.io/ko/docs/concepts/configuration/secret/)

그런데 시크릿이 완벽한 솔루션은 아니다. 결국 같은 네임스페이스를 쓰거나 팟에 들어올 수 있는 사람들은 어떻게든 시크릿을 볼 수 있기 때문이다.

하지만 익숙해지면 만드는게 어렵지도 않고, 최소한 코드나 이미지에는 민감한 정보가 없어야 하므로 사용하는 것이 좋다.

시크릿을 만든 다음에는, 팟에서 해당 시크릿을 마운트하거나 환경 변수로 불러와서 사용하면 된다. 본 글에서는 `1) 시크릿 만들기`, `2) 팟에서 시크릿 접근하기` 순서대로 설명하겠다.


## 1. 시크릿 만들기


**시크릿의 모든 값은 base64로 인코딩 되어야 한다.** 리눅스,맥 터미널에서는 아래 코드를 이용하면 쉽게 인코딩 할 수 있다. (윈도우는 파워쉘로 하면 된다. 그리고 구글링 해보면 온라인에서 할 수 있는 곳도 금방 찾을 수 있다.)


```bash
echo -n '인코딩할문자열' | base64
```

수행 예

```bash
jinwoong.kim $ echo -n '인코딩할문자열' | base64
7J247L2U65Sp7ZWg66y47J6Q7Je0
jinwoong.kim $
```

이렇게 시크릿에 넣을 값들을 base64 로 인코딩해주고 아래와 같이 시크릿 `json` 혹은 `yaml` 파일을 만들어 준다음에 `data` 항목에 키, 벨류 값을 넣어 주면 된다.

##### `secret.json`
```json
{
  "metadata": {
    "annotations": {},
    "name": "secret-name",
    "namespace": "default"
  },
  "apiVersion": "v1",
  "kind": "Secret",
  "data": {
    "인코딩할문자열": "7J247L2U65Sp7ZWg66y47J6Q7Je0",
    "mypassword": "bXlwYXNzd29yZA=="
  }
}
```

혹은
##### `secret.yaml`
```yaml
kind: Secret
apiVersion: v1
metadata:
  name: secret-name
  namespace: default
data:
  인코딩할문자열: 7J247L2U65Sp7ZWg66y47J6Q7Je0==
  mypassword: bXlwYXNzd29yZA==
type: Opaque
```

위와 같이 시크릿 파일을 작성 후, 아래처럼 `kubectl` 명령어를 실행하면 시크릿을 만들 수 있다.

```
jinwoong.kim $ kubectl create -f secret.json
```

생성 확인
```
jinwoong.kim $ kubectl get secret
NAME                                         TYPE                                  DATA   AGE
secret-name                                 Opaque                                2      2d
```

`Opaque`는 시크릿 타입을 지정하지 않았을때 지정되는 기본 시크릿 타입이다. 시크릿 타입에 대한 더 자세한 정보는 [여기](https://kubernetes.io/ko/docs/concepts/configuration/secret/#secret-types)에서 확인 할 수 있다.
{: .notice--info}

### 2. 팟에서 시크릿 접근하기

자, 이제 시크릿을 만들었으니, 팟에서 시크릿을 접근해보자. 팟에서 시크릿을 접근하는 방법은 `볼륨 마운팅`, `환경 변수`, 총 두 가지가 있다. 하나씩 알아보자.
##### pod.yaml
```yaml
apiVersion: apps/v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: nginx
```

팟을 생성하기 위한 아주 기본적인 `pod.yaml` 을 예로 설명하겠다.

##### 방법1) 볼륨 마운트
**pod_with_mounted_secret.yaml**
```yaml
apiVersion: apps/v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: nginx
    volumeMounts:
    - name: secret
      mountPath: "/etc/secret"
      readOnly: true
  volumes:
  - name: secret
    secret:
      secretName: secret-name
      optional: false
```

위의 `pod.yaml`에 두 가지를 추가해주면 된다. 먼저, `spec.volumes` 에 시크릿을 위한 볼륨을 생성해주고, 그 다음에 `spec.containers.volumeMounts` 에서 해당 볼륨을  `"/etc/secret"` 밑에 마운트 해주면 된다  (당연히 마운트 위치는 본인 마음이다.). 이렇게 하면 마운트 한 위치에 시크릿 값들이 파일로 마운팅 되는데, 이때  파일들을 읽어보면 시크릿 키의 값이 나온다. 

```
jinwoong.kim $ cd /etc/secret
jinwoong.kim:/etc/secret $ ls
인코딩할문자열  mypassword

jinwoong.kim:/etc/secret $ cat 인코딩할문자열
7J247L2U65Sp7ZWg66y47J6Q7Je0

jinwoong.kim:/etc/secret $ cat mypassword
bXlwYXNzd29yZA
```

##### 방법2) pod_with_env_secret.yaml
```yaml
apiVersion: apps/v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: nginx
    env:
    - name: mypassword
        valueFrom:
          secretKeyRef:
            name: secret-name
            key: mypassword

```

또 다른 방법으로는 위와 같이 환경 변수로 시크릿을 가져 오는 방법이다. 이를 위해선, 위와 같이 `spec.containers.env` 에  가져 올 시크릿의 키를 환경 변수로 셋팅해주면 된다.

팟에서 환경변수(키)에 접근해보면 아래와 같이 값이 출력 되는 것을 볼 수 있다.

```
jinwoong.kim $ echo $인코딩할문자열
7J247L2U65Sp7ZWg66y47J6Q7Je0

jinwoong.kim $ echo $mypassword
bXlwYXNzd29yZA
```

이렇게 위의 두 가지 방법 중 본인이 선호하는 방식으로 시크릿을 접근 하면 된다.