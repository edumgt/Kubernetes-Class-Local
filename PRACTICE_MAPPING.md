# 00~10 실습 ↔ Lecture 매핑 (비교 결과)

`00~10` 실습의 README 주제(아키텍처/Pod/ReplicaSet/Deployment/Service/YAML)와
`lecture01~15`의 이론/실습 문서 주제를 비교하여 아래처럼 통합했습니다.

| 기존 실습 폴더 | 통합 위치 | 매핑 근거 |
|---|---|---|
| `00-Docker-Images` | `lecture01/practice/00-Docker-Images` | 환경 사전 준비/이미지 빌드가 실습 시작 단계와 일치 |
| `01-Kubernetes-Architecture` | `lecture08/practice/01-Kubernetes-Architecture` | Node/Pod 동작 이해 전 아키텍처 기초 보강 |
| `02-PODs-with-kubectl` | `lecture08/practice/02-PODs-with-kubectl` | Pod 생성/조회/삭제/Service 노출이 lecture08 핵심 주제와 일치 |
| `03-ReplicaSets-with-kubectl` | `lecture09/practice/03-ReplicaSets-with-kubectl` | ReplicaSet + selector/service 연계가 labels/selectors 학습과 직결 |
| `04-Deployments-with-kubectl` | `lecture10/practice/04-Deployments-with-kubectl` | Deployment 운영(스케일/업데이트/롤백)이 HPA 전제 학습으로 적합 |
| `05-Services-with-kubectl` | `lecture13/practice/05-Services-with-kubectl` | Service 라우팅 이해가 Ingress 디버깅 실습의 선행 요소 |
| `06-YAML-Basics` | `lecture11/practice/06-YAML-Basics` | 매니페스트 문법/구조가 lecture11의 Manifest 주제와 일치 |
| `07-PODs-with-YAML` | `lecture11/practice/07-PODs-with-YAML` | YAML 기반 Pod 정의 실습 |
| `08-ReplicaSets-with-YAML` | `lecture11/practice/08-ReplicaSets-with-YAML` | YAML 기반 ReplicaSet 정의 실습 |
| `09-Deployments-with-YAML` | `lecture11/practice/09-Deployments-with-YAML` | YAML 기반 Deployment 정의 실습 |
| `10-Services-with-YAML` | `lecture11/practice/10-Services-with-YAML` | YAML 기반 Service 정의/적용/정리 흐름 |

## 반영 원칙
- 실습 폴더를 루트에서 분리하고, 강의별 `practice/` 하위로 이동
- 이론 문서와 실습 자료를 동일 강의 경로에서 접근 가능하게 구성
- 강의 인덱스는 `CURRICULUM.md`에 통합 링크로 반영
