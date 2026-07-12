# Chapter 3. 첫 번째 배포 파이프라인

> 책: AI 시대에 개발자가 알아야 할 인프라 구성 배포 with 클로드코드  
> 읽은 범위: Chapter 3, 첫 번째 배포 파이프라인

## 한 줄 요약

코드를 Push하면 빌드부터 배포까지 자동으로 이어지는 파이프라인을 만들되, 명령형 배포의 한계를 넘어서 Git을 기준으로 상태를 관리하는 GitOps 방식으로 전환하는 것이 핵심이다.

---

## 이 장에서 다룬 내용

Chapter 3에서는 첫 번째 배포 파이프라인을 구성하면서, 단순한 자동 배포를 넘어 GitOps 방식이 왜 필요한지 설명한다.

핵심적으로 다룬 내용은 다음과 같다.

- 코드 Push 이후 빌드부터 배포까지 자동화되는 파이프라인
- 명령형 배포 방식이 가지는 구조적 문제
- 설정 드리프트와 기준 상태의 불명확함
- 배포 이력에서 “왜”가 사라지는 문제
- 선언형 사고로 전환해야 하는 이유
- Argo CD를 활용한 GitOps 구조
- CRD와 CR의 의미
- Server-side apply와 `--force-conflicts`
- `kubectl rollout undo`가 아니라 `git revert`로 롤백하는 방식
- GitHub Actions CI와 Argo CD를 연결하는 흐름
- 이미지 태그를 Git 커밋 SHA로 관리하는 방식

---

## 배포 파이프라인의 목표

이 장의 목표는 코드를 Push하면 빌드부터 배포까지 자동으로 이어지는 파이프라인을 만드는 것이다.

흐름을 단순화하면 다음과 같다.

```text
코드 변경
→ Git Push
→ CI에서 컨테이너 이미지 빌드
→ 이미지 레지스트리에 Push
→ Manifest의 이미지 태그 갱신
→ Git에 변경 사항 커밋
→ Argo CD가 변경 감지
→ Kubernetes 클러스터에 자동 배포
```

이 구조에서 중요한 점은 배포가 단순히 CI/CD 파이프라인에서 명령을 실행하는 방식으로 끝나지 않는다는 것이다.

최종적으로 어떤 상태가 되어야 하는지는 Git에 선언되고, 클러스터는 Git에 기록된 상태를 따라간다.

---

## 명령형 배포 방식의 구조적 문제

이 장에서는 기존 명령형 배포 방식이 가지는 세 가지 구조적 문제를 설명한다.

명령형 방식은 사람이 직접 명령을 실행해서 현재 상태를 바꾸는 방식이다.

예를 들면 다음과 같은 명령들이 있다.

```bash
kubectl edit configmap <CONFIGMAP_NAME>
kubectl set image deployment/<DEPLOYMENT_NAME> ...
kubectl rollout undo deployment/<DEPLOYMENT_NAME>
kubectl apply -f <MANIFEST_FILE>
```

이런 명령들은 빠르게 문제를 해결할 수 있다는 장점이 있지만, 운영 환경에서는 기준 상태와 이력이 흐려질 수 있다.

---

## 문제 1. 드리프트를 알 수 없다

첫 번째 문제는 설정 드리프트다.

설정 드리프트는 실제 클러스터의 상태와 Git에 저장된 상태가 달라지는 상황을 말한다.

예를 들어 긴급한 상황에서 누군가 클러스터에 직접 접속해 ConfigMap을 수정했다고 가정해보자.

```bash
kubectl edit configmap <CONFIGMAP_NAME>
```

그 후 Pod를 재시작해서 문제가 해결되었다.  
하지만 이 변경을 Git에 반영하지 않았다면, 클러스터의 실제 상태와 Git 저장소의 Manifest가 달라진다.

이 상태가 시간이 지나면 문제가 된다.

나중에 다른 사람이 Git 저장소의 ConfigMap과 클러스터 안의 ConfigMap이 다르다는 것을 발견했을 때, 어느 쪽이 진짜 기준인지 알기 어려워진다.

```text
Git에 있는 설정이 맞는가?
클러스터에 직접 수정된 설정이 맞는가?
긴급 수정이었는가?
실수였는가?
다시 Git에 반영해야 하는가?
```

이런 상황이 바로 설정 드리프트다.

### 생각한 점

이 문제는 실제 운영 환경에서도 충분히 발생할 수 있는 시나리오라고 느꼈다.

예를 들어 다음과 같은 방식으로 Git 기반 배포 흐름을 우회하면 드리프트가 생길 수 있다.

- Bastion 서버에서 직접 `kubectl` 명령 실행
- Kubernetes 관리 UI에서 직접 리소스 수정
- CI/CD 파이프라인을 거치지 않은 수동 변경
- 긴급 수정 후 Git 반영 누락

처음에는 빠른 대응처럼 보이지만, 시간이 지나면 “진짜 기준 상태가 어디인가”를 알 수 없게 된다.

GitOps는 이런 문제를 줄이기 위해 Git을 기준 상태로 삼고, 클러스터가 그 상태를 따라가도록 만든다.

---

## 문제 2. “왜”가 남지 않는다

두 번째 문제는 배포의 이유가 남지 않는다는 것이다.

예를 들어 이미지 태그를 잘못 입력해서 Pod가 `ImagePullBackOff` 상태에 빠졌다고 가정해보자.  
이때 다음 명령으로 직전 ReplicaSet으로 되돌릴 수 있다.

```bash
kubectl rollout undo deployment/<DEPLOYMENT_NAME>
```

이 명령은 빠르게 롤백할 수 있다는 장점이 있다.

하지만 시간이 지나면 다음 질문에 답하기 어려워진다.

```text
왜 이 배포를 했는가?
왜 롤백했는가?
어떤 변경이 문제였는가?
누가 어떤 의도로 배포했는가?
```

`kubectl rollout history`를 확인하더라도 `CHANGE-CAUSE`가 비어 있는 경우가 많고, revision 보관 개수에도 제한이 있다.  
배포가 여러 번 반복되면 과거 이력이 사라질 수도 있다.

결국 명령은 실행되었지만, 그 명령을 실행한 이유와 맥락이 남지 않는 문제가 생긴다.

---

## 문제 3. 리소스마다 기준이 다르다

세 번째 문제는 리소스마다 복원 기준이 다르다는 것이다.

Deployment처럼 rollout history가 있는 리소스는 어느 정도 이전 상태를 확인할 수 있다.  
하지만 `kubectl edit`로 수정한 ConfigMap, Secret, Service 같은 리소스는 동일한 방식으로 복원 근거를 찾기 어렵다.

즉, 어떤 리소스는 히스토리가 있고, 어떤 리소스는 히스토리가 없다.

운영 관점에서는 이 차이가 큰 문제가 된다.

```text
Deployment는 이전 revision으로 되돌릴 수 있다.
ConfigMap은 직접 수정했다면 변경 이유를 알기 어렵다.
Service나 Secret도 변경 이력이 명확하지 않을 수 있다.
```

리소스마다 기준과 복원 방식이 다르면 운영이 복잡해진다.

---

## 선언형 사고로 전환하기

이 세 가지 문제를 해결하는 첫 단계는 명령형 사고에서 선언형 사고로 전환하는 것이다.

명령형은 “무엇을 실행할 것인가”에 집중한다.

```text
이 ConfigMap을 수정해.
이 이미지를 바꿔.
이 Deployment를 롤백해.
```

반면 선언형은 “원하는 최종 상태가 무엇인가”에 집중한다.

```text
이 ConfigMap은 이런 값을 가져야 한다.
이 Deployment는 이 이미지 태그를 사용해야 한다.
이 Service는 이런 포트를 열어야 한다.
```

Kubernetes는 기본적으로 선언형 철학을 가진 플랫폼이다.

Manifest에 원하는 상태를 작성하고, Kubernetes는 실제 상태를 그 선언에 맞추려고 한다.

선언형 방식은 처음에는 불편해 보일 수 있다.  
매번 YAML을 수정해야 하고, Git에 커밋해야 하기 때문이다.

하지만 그 YAML이 기준 상태가 된다.

YAML을 Git에 커밋하면 자연스럽게 다음 정보가 남는다.

- 누가 변경했는가
- 언제 변경했는가
- 무엇을 변경했는가
- 왜 변경했는가
- 어떤 리뷰를 거쳤는가
- 문제가 생기면 어디로 되돌릴 수 있는가

즉, 선언형 사고는 단순히 YAML을 쓰는 방식이 아니라, 운영의 기준을 명확히 남기는 방식이다.

---

## GitOps와 Argo CD

GitOps는 Git에 적힌 YAML을 기준으로 삼고, 클러스터가 자동으로 그 상태를 따라오게 만드는 구조다.

이 구조를 구현하는 대표적인 도구 중 하나가 Argo CD다.

Argo CD는 Git 저장소에 선언된 Kubernetes Manifest를 감시하고, 실제 클러스터 상태와 비교한 뒤 필요한 경우 동기화한다.

Argo CD의 기본 동작 흐름은 다음과 같다.

```text
Git 저장소의 특정 경로 감시
→ Git 상태와 클러스터 상태 비교
→ 차이가 있으면 OutOfSync 상태로 판단
→ 필요하면 클러스터를 Git 상태로 동기화
→ 직접 수정된 리소스도 Git 기준 상태로 복구
```

책에서는 Argo CD가 주기적으로 Git과 클러스터 상태를 비교하고, 차이가 있으면 클러스터를 Git 상태에 맞춘다고 설명한다.

이 구조 덕분에 누군가 클러스터에서 직접 수정하더라도 Git에 선언된 상태와 다르다면 다시 원래 상태로 돌아갈 수 있다.

---

## Argo CD의 의미

Argo CD를 사용하면 배포의 기준이 명확해진다.

기존 방식에서는 CI/CD 파이프라인이 다음과 같이 직접 배포 명령을 실행할 수 있다.

```text
CI Pipeline
→ kubectl apply 또는 helm upgrade 실행
→ 클러스터 변경
```

반면 Argo CD 기반 GitOps에서는 흐름이 조금 다르다.

```text
CI Pipeline
→ 이미지 빌드
→ Manifest 변경
→ Git에 커밋
→ Argo CD가 Git 변경 감지
→ 클러스터 동기화
```

차이는 작아 보이지만 중요하다.

배포 명령의 주체가 CI/CD 파이프라인에서 GitOps 도구로 이동하고, 배포 기준은 Git이 된다.

즉, CI는 이미지를 만들고 Git을 갱신하는 역할에 집중하고, Argo CD는 Git에 선언된 상태를 클러스터에 반영하는 역할을 담당한다.

---

## CRD와 CR

이 장에서는 CRD와 CR도 함께 다룬다.

CRD는 Custom Resource Definition의 약자로, Kubernetes에 새로운 리소스 종류를 추가하는 설계도라고 볼 수 있다.

CR은 Custom Resource의 약자로, CRD를 바탕으로 만든 실제 리소스 객체다.

비유하면 다음과 같다.

```text
CRD = 새로운 리소스 타입을 정의하는 설계도
CR  = 그 설계도로 만든 실제 리소스
```

예를 들어 Argo CD의 `Application`은 Kubernetes에 기본으로 존재하는 리소스가 아니다.  
Argo CD가 CRD를 통해 `Application`이라는 리소스 타입을 추가하고, 사용자는 그 타입의 실제 리소스를 만들어 애플리케이션 배포를 정의한다.

Gateway API의 `Gateway` 같은 리소스도 같은 방식으로 이해할 수 있다.

CRD를 사용하면 Kubernetes의 기본 리소스가 아니더라도 YAML, `kubectl`, GitOps 흐름으로 동일하게 다룰 수 있다.

이 점이 중요하다.

도구마다 별도의 설정 방식만 제공하는 것이 아니라, Kubernetes 리소스처럼 선언하고 관리할 수 있게 된다.

대부분의 경우 CRD는 컨트롤러와 함께 동작한다.  
컨트롤러는 사용자가 선언한 CR을 보고 실제 상태를 그에 맞추는 역할을 한다.

이 구조가 흔히 말하는 Operator 패턴의 기반이 된다.

---

## Server-side apply와 `--force-conflicts`

Argo CD 같은 도구를 설치할 때 다음과 같은 옵션을 사용하는 경우가 있다.

```bash
kubectl apply --server-side=true --force-conflicts -f <MANIFEST_FILE>
```

여기서 `--server-side=true`는 apply 병합 처리를 클라이언트가 아니라 Kubernetes API 서버 쪽에서 수행하도록 하는 옵션이다.

일반적인 client-side apply는 마지막 적용 상태를 애노테이션에 저장하는 방식 때문에 큰 CRD를 다룰 때 애노테이션 크기 제한에 걸릴 수 있다.

Argo CD의 CRD처럼 크기가 큰 리소스는 이 제한을 넘을 수 있기 때문에 server-side apply를 사용하는 것이다.

`--force-conflicts`는 server-side apply 과정에서 필드 소유권 충돌이 발생했을 때 이를 강제로 해결하기 위한 옵션이다.

이 조합은 Argo CD뿐만 아니라 CRD가 큰 다른 도구를 설치할 때도 자주 등장할 수 있다.

예를 들면 다음과 같은 도구들이 있다.

- Istio
- cert-manager
- Prometheus Operator
- Gateway API
- Argo CD

정리하면 다음과 같다.

| 옵션 | 의미 |
|---|---|
| `--server-side=true` | 병합 처리를 Kubernetes API 서버에서 수행 |
| `--force-conflicts` | 필드 소유권 충돌이 있을 때 강제로 적용 |
| 사용하는 이유 | 큰 CRD나 복잡한 리소스를 안정적으로 적용하기 위해 |

---

## Argo CD 관련 명령어

실습에서는 Argo CD CLI와 `kubectl`을 사용해 애플리케이션 상태를 확인한다.

공개용 예시는 다음과 같이 일반화할 수 있다.

```bash
argocd login localhost:8443 \
  --username admin \
  --password <PASSWORD> \
  --insecure
```

로컬 포트포워딩을 통해 Argo CD API 서버에 접속하고, admin 계정으로 로그인하는 명령이다.

```bash
kubectl --context <KUBE_CONTEXT> get application -n argocd
```

Argo CD의 `Application` 리소스를 확인한다.

```bash
argocd app get <APP_NAME>
```

특정 Argo CD 애플리케이션의 상태를 확인한다.

```bash
argocd repo list
```

Argo CD에 등록된 Git 저장소 목록을 확인한다.

이런 명령들은 GitOps가 실제로 어떤 저장소와 애플리케이션을 기준으로 동작하는지 확인할 때 유용하다.

---

## GitOps에서의 롤백

GitOps가 왜 유용한지는 롤백할 때 특히 잘 드러난다.

명령형 방식에서는 문제가 생겼을 때 다음과 같은 명령을 사용할 수 있다.

```bash
kubectl rollout undo deployment/<DEPLOYMENT_NAME>
```

이 명령은 빠르게 Deployment를 이전 ReplicaSet으로 되돌릴 수 있다.

하지만 GitOps에서는 기준이 클러스터의 revision이 아니라 Git이다.

따라서 롤백도 Git을 기준으로 수행한다.

```bash
git revert <COMMIT_SHA>
git push
```

그러면 되돌리는 변경 사항이 새 커밋으로 Git에 기록되고, Argo CD가 이를 감지해 클러스터 상태를 다시 동기화한다.

```text
문제 있는 커밋 발생
→ git revert로 반대 변경 커밋 생성
→ Git에 push
→ Argo CD가 변경 감지
→ 클러스터가 되돌려진 선언 상태로 동기화
```

### 생각한 점

`kubectl rollout undo`가 아니라 `git revert`를 이용해 롤백한다는 점이 선언형 사고를 잘 보여준다고 느꼈다.

기존 배포 방식에서도 Manifest나 Helm Chart를 Git으로 관리하고 있다면, Git 이력을 기준으로 롤백할 수 있는지 확인해볼 필요가 있다.

또한 Argo CD 같은 GitOps 도구를 도입했을 때 얻을 수 있는 효과도 함께 고민해볼 수 있다.

단순히 “배포가 자동으로 된다”가 아니라, 다음과 같은 질문에 답할 수 있는지가 중요하다.

- 롤백이 Git 이력으로 남는가?
- 누가 어떤 변경을 되돌렸는지 확인할 수 있는가?
- 되돌린 이유를 커밋 메시지와 Pull Request에 남길 수 있는가?
- 클러스터 상태가 Git 상태와 계속 일치하는가?

---

## `git revert`가 중요한 이유

`git revert`는 기존 커밋을 지우는 명령이 아니다.

기존 커밋은 그대로 보존하고, 그 변경을 반대로 적용하는 새 커밋을 추가한다.

예를 들어 A라는 커밋에서 이미지 태그를 잘못 변경했다면, `git revert`는 A 커밋을 삭제하지 않는다.  
대신 A 커밋의 변경 내용을 되돌리는 새로운 커밋 B를 만든다.

```text
기존 이력:
A -- B -- C

C를 revert한 이후:
A -- B -- C -- D

D = C의 변경을 되돌리는 새 커밋
```

이 방식은 협업 환경에서 안전하다.

그 이유는 다음과 같다.

- 원본 커밋이 보존된다.
- 롤백 자체가 새 커밋으로 기록된다.
- 누가 언제 무엇을 되돌렸는지 남는다.
- 되돌린 커밋을 다시 되돌릴 수도 있다.
- 이미 공유된 Git 히스토리를 재작성하지 않는다.

반면 `reset`이나 `rebase`는 Git 히스토리를 다시 쓰는 방식이다.  
개인 브랜치에서는 사용할 수 있지만, 이미 공유된 브랜치나 GitOps 기준 브랜치에서는 위험할 수 있다.

GitOps에서는 Git 히스토리가 곧 운영 이력에 가까워지기 때문에, 이력을 지우는 방식보다 이력을 보존하는 방식이 더 적합하다.

---

## GitOps 파이프라인 구성

이 장에서 구성하는 GitOps 파이프라인은 크게 세 부분으로 볼 수 있다.

```text
1. Argo CD
2. GitHub Actions CI
3. CI와 Argo CD를 연결하는 Manifest 갱신 흐름
```

---

## 1. Argo CD

Argo CD는 Git Push만으로 배포가 반영되는 선언적 배포 체계를 만든다.

Argo CD에서는 `Application` 리소스를 통해 다음 정보를 연결한다.

- Git 저장소
- Manifest 경로
- 배포 대상 클러스터
- 배포 대상 namespace
- 동기화 정책

`auto-sync`를 사용하면 Git에 변경이 생겼을 때 자동으로 클러스터에 반영할 수 있다.

`selfHeal`을 사용하면 누군가 클러스터에서 직접 리소스를 수정했을 때, Git에 선언된 상태로 다시 되돌릴 수 있다.

이 구조가 설정 드리프트를 줄이는 핵심이다.

```text
Git 상태와 클러스터 상태가 다름
→ Argo CD가 차이 감지
→ Git 기준 상태로 복구
```

---

## 2. GitHub Actions CI

GitHub Actions CI는 컨테이너 이미지를 빌드하고 이미지 레지스트리에 Push하는 역할을 한다.

예를 들어 다음과 같은 일을 수행할 수 있다.

```text
코드 checkout
→ 클라우드 인증
→ Docker 인증 설정
→ docker build
→ docker push
→ 이미지 태그 생성
```

책에서는 이미지 태그로 Git 커밋 SHA의 앞 7자리를 사용한다.

예를 들면 다음과 같다.

```text
commit sha: abcdef1234567890...
image tag : abcdef1
```

이 방식은 `latest` 태그보다 추적 가능성이 좋다.

`latest`는 시간이 지나면 어떤 코드로 만든 이미지인지 알기 어려워질 수 있다.  
반면 커밋 SHA 기반 태그는 이미지와 소스 코드의 연결 관계를 명확히 남길 수 있다.

---

## 3. CI와 Argo CD 연결

CI와 Argo CD는 직접적으로 배포 명령을 주고받지 않는다.

대신 CI가 이미지를 빌드한 뒤, Kubernetes Manifest의 이미지 태그를 새 SHA 태그로 변경하고 그 변경을 Git에 커밋한다.

그 후 Argo CD가 해당 커밋을 감지해 배포한다.

```text
GitHub Actions
→ 새 이미지 빌드
→ 이미지 레지스트리에 Push
→ Manifest의 이미지 태그 변경
→ Git에 커밋 및 Push
→ Argo CD가 변경 감지
→ 클러스터에 배포
```

책에서는 `sed`를 사용해 Manifest의 이미지 태그를 교체하는 방식이 나온다.

이 방식의 핵심은 사람이 직접 Manifest의 이미지 태그를 바꾸는 작업을 줄이는 것이다.

사람이 직접 태그를 수정하다 보면 다음과 같은 실수가 생길 수 있다.

- 이미지 태그 변경을 깜빡함
- 코드만 Push하고 배포 파이프라인 실행을 잊음
- 잘못된 태그를 입력함
- 빌드된 이미지와 Manifest가 가리키는 이미지가 달라짐

CI가 이미지 태그를 갱신하고 Git에 커밋하면, 이런 휴먼 에러를 줄일 수 있다.

### 생각한 점

이미지 태그를 바꾸는 것을 잊거나, 코드를 Push했지만 배포 파이프라인을 실행하지 않아 문제가 생기는 상황은 충분히 발생할 수 있다.

GitOps 파이프라인을 촘촘하게 구성하면 이런 휴먼 에러를 줄일 수 있을 것 같다.

중요한 것은 단순히 자동화하는 것이 아니라, 다음 흐름이 끊기지 않도록 만드는 것이다.

```text
코드 변경
→ 이미지 빌드
→ 이미지 태그 생성
→ Manifest 갱신
→ Git 기록
→ 배포 반영
```

이 중 하나라도 수동으로 남아 있으면 실수가 생길 수 있다.  
따라서 GitOps 파이프라인을 설계할 때는 “어디까지 자동화할 것인가”와 함께 “어떤 지점에서 사람이 검토할 것인가”를 함께 고민해야 한다.

---

## 명령형 배포와 GitOps 배포 비교

이 장을 읽고 명령형 배포와 GitOps 배포의 차이를 다음과 같이 정리할 수 있었다.

| 항목 | 명령형 배포 | GitOps 배포 |
|---|---|---|
| 기준 상태 | 현재 클러스터 상태 또는 실행한 명령 | Git에 선언된 상태 |
| 변경 방식 | `kubectl`, `helm` 등 명령 직접 실행 | Manifest 변경 후 Git 커밋 |
| 배포 이력 | 도구나 리소스별로 다름 | Git 이력으로 남음 |
| 변경 이유 | 남기기 어려움 | 커밋 메시지, PR, 리뷰에 남길 수 있음 |
| 드리프트 감지 | 별도 관리 필요 | Git과 클러스터 상태 비교 가능 |
| 롤백 | `kubectl rollout undo` 등 | `git revert` |
| 감사 가능성 | 제한적 | Git 기반으로 추적 가능 |
| 휴먼 에러 | 수동 작업이 많을수록 증가 | 자동화와 선언형 관리로 감소 가능 |

---

## 내가 이해한 방식

Chapter 3를 읽고 첫 번째 배포 파이프라인의 핵심을 다음과 같이 이해했다.

```text
CI/CD는 코드를 빌드하고 이미지를 만든다.
GitOps는 Git에 선언된 상태를 클러스터에 반영한다.
Argo CD는 Git과 클러스터 사이의 차이를 감지하고 동기화한다.
Rollback은 클러스터 명령이 아니라 Git 이력으로 수행한다.
```

즉, GitOps 파이프라인에서는 CI와 CD의 역할이 분리된다.

CI는 이미지를 만들고, Manifest를 갱신한다.  
CD는 Git에 선언된 Manifest를 기준으로 클러스터를 동기화한다.

이 구조에서 Git은 단순한 코드 저장소가 아니다.

Git은 다음 역할을 한다.

- 배포 기준 상태
- 변경 이력
- 롤백 기준
- 리뷰 대상
- 감사 기록
- 협업의 중심

이 점이 GitOps의 핵심이라고 느꼈다.

---

## 더 알아볼 것

이 장을 읽고 나서 더 확인해보고 싶은 내용은 다음과 같다.

- Argo CD의 `auto-sync`와 `selfHeal` 동작 방식
- Argo CD에서 드리프트가 감지되는 구체적인 조건
- Argo CD Application 리소스 작성 방식
- GitOps에서 `git revert` 기반 롤백 절차
- 기존 CI/CD 기반 배포 방식에서 GitOps 방식으로 전환할 때 고려할 점
- 이미지 태그를 커밋 SHA로 관리하는 방식의 장단점
- `sed`를 이용한 Manifest 태그 변경 방식의 한계
- Helm, Kustomize를 사용할 때 이미지 태그 갱신을 관리하는 방법
- CI가 Manifest를 직접 커밋하는 방식의 장단점
- Pull Request 기반 승인 절차와 GitOps 자동 동기화를 어떻게 함께 설계할지
- 직접 `kubectl` 접근을 제한하거나 감지하는 방법
- 설정 드리프트를 조직적으로 줄이는 방법

---

## 정리

Chapter 3에서 가장 크게 남은 키워드는 다음과 같다.

- 명령형 배포는 빠르지만 기준 상태와 이력이 흐려질 수 있다.
- 설정 드리프트는 Git 상태와 클러스터 상태가 달라지는 문제다.
- Kubernetes는 선언형 철학을 가진 플랫폼이다.
- GitOps는 Git에 선언된 상태를 기준으로 클러스터를 동기화한다.
- Argo CD는 Git과 클러스터 상태의 차이를 감지하고 자동으로 맞춘다.
- CRD는 Kubernetes에 새로운 리소스 종류를 추가하는 설계도다.
- 큰 CRD를 적용할 때 server-side apply가 필요할 수 있다.
- GitOps에서 롤백은 `kubectl rollout undo`보다 `git revert`가 더 자연스럽다.
- CI는 이미지를 만들고 Manifest를 갱신한다.
- Argo CD는 Git에 기록된 Manifest를 기준으로 배포한다.
- 커밋 SHA 기반 이미지 태그는 배포 추적성을 높인다.

이번 장은 단순히 “배포 자동화 파이프라인을 만든다”는 내용이 아니었다.

더 중요한 메시지는 배포의 기준을 어디에 둘 것인가였다.

명령형 방식에서는 사람이 실행한 명령과 현재 클러스터 상태가 중심이 된다.  
하지만 GitOps 방식에서는 Git에 선언된 상태가 중심이 된다.

이 차이가 드리프트 감지, 배포 이력, 롤백, 감사 가능성, 협업 방식까지 바꾼다.

결국 GitOps의 핵심은 다음 문장으로 정리할 수 있을 것 같다.

> 클러스터를 직접 고치는 것이 아니라,  
> Git에 원하는 상태를 선언하고  
> 클러스터가 그 상태를 따라오게 만든다.

AI 시대에는 이 구조가 더 중요해질 것 같다.

AI가 Manifest를 수정하고 배포 흐름을 도와줄 수 있게 되더라도, 그 결과가 Git에 기록되고 검토되고 되돌릴 수 있어야 안전하다.

따라서 앞으로의 배포 파이프라인은 단순히 빠른 자동화가 아니라,  
**기록 가능하고, 검토 가능하며, 되돌릴 수 있는 자동화**가 되어야 한다.
