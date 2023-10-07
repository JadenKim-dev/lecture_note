# Authorization 실습

먼저 다음의 구성 파일로 Role 객체를 생성한다.  
pod에 대한 조회 권한을 부여하는 Role이다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: r-01
  namespace: nm-01
rules:
  - apiGroups: [""]
    verbs: ["get", "list"]
    resources: ["pods"]
```

`apiGroups`에는 각 객체의 구성 파일 상단에서 apiVersion에 정의한 내용을 적어야 한다.  
단 pod와 같이 코어 API인 경우에는 apiVersion을 적지 않기 때문에, 위에서도 빈 문자열로 두었다.  
`verbs`에는 어떤 메서드에 대한 권한을 허용할지를 적어야 하는데, get, list, create, update, patch, watch, delete, deletecollection 중에 선택해서 적을 수 있다.  
`resources`에는 권한 제어 대상이 되는 리소스 종류를 적는다.

이제 다음의 구성 파일로 RoleBinding 객체를 생성한다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rb-01
  namespace: nm-01
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: r-01
subjects:
  - kind: ServiceAccount
    name: default
    namespace: nm-01
```
