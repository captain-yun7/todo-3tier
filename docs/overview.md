# todo-3tier 개요

웹 · API · DB 3티어 todo 앱을 VM에 세 방식(직접 설치 / Compose / Kubernetes)으로 배포해 비교하는 실습. 전체 절차는 [specs.md](specs.md).

## 스택
- web: nginx 1.29, 정적 HTML/JS. `/api/`만 api로 프록시
- api: Node 22, Express 5, CommonJS
- db: PostgreSQL 17 (VM 직접 설치는 Ubuntu 패키지 16)

## 구조
- `api/`, `web/`: 앱 코드와 Dockerfile
- `compose.yaml`: 배포 ② — 비밀번호는 `.env`의 `DB_PASSWORD`
- `k8s/`: 배포 ③ — kind 클러스터, Traefik values, 매니페스트. 비밀번호는 Secret `todo-secret`
