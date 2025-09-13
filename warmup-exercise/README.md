# Istio Gateway API 충돌 이슈 해결

## 문제 상황

### 발생한 에러들

1. **istiod readiness probe 실패**

   ```
   Readiness probe failed: Get "http://10.244.0.17:8080/ready": dial tcp 10.244.0.17:8080: connect: connection refused
   ```

2. **Gateway API 리소스 찾기 실패**

   ```
   failed to list *v1alpha2.Gateway: the server could not find the requested resource (get gateways.gateway.networking.k8s.io)
   ```

3. **ConfigMap 누락으로 인한 볼륨 마운트 실패**
   ```
   MountVolume.SetUp failed for volume "istiod-ca-cert": configmap "istio-ca-root-cert" not found
   ```

## 근본 원인

### 환경 차이점

- **강의 환경**: minikube + 순수 Istio
- **실제 환경**: Kind + APISIX 기설치 (Gateway API CRD 포함)

### 충돌 발생 과정

1. Istio 1.15.1은 기본적으로 Kubernetes Gateway API 지원 활성화
2. istiod가 시작하면서 `v1alpha2.Gateway` 리소스를 찾으려 시도
3. APISIX가 설치한 Gateway API는 `v1beta1` 버전만 제공
4. 버전 불일치로 인한 리소스 탐색 실패
5. istiod 초기화 실패로 readiness probe 계속 실패

## 해결 방법

### 실행 명령어

```bash
istioctl install --set values.pilot.env.PILOT_ENABLE_GATEWAY_API=false -y
```

### 적용된 변경사항

- istiod deployment에 환경변수 `PILOT_ENABLE_GATEWAY_API=false` 추가
- Kubernetes Gateway API 지원 완전 비활성화
- 전통적인 Istio Gateway (`networking.istio.io/v1alpha3`) 만 사용

## 해결 결과

### 1. istiod 정상화

- Pod 상태: Running 0/1 → Running 1/1
- readiness probe 성공
- 8080 포트 정상 응답

### 2. 필수 리소스 생성

- `istio-ca-root-cert` ConfigMap 자동 생성
- ingress/egress gateway 볼륨 마운트 문제 해결

### 3. 환경 안정화

- APISIX Gateway API와 분리 운영
- 강의 환경과 동일한 Istio Gateway 사용
- 버전 충돌 문제 완전 해결

## 결론

APISIX의 Kubernetes Gateway API와 Istio의 Gateway API 버전 불일치를 Gateway API 비활성화로 해결. 기존 APISIX 환경을 유지하면서 Istio 강의 진행을 위한 안정적인 환경 확보.
