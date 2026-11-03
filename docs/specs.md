# 3티어 앱 만들기와 배포 세 방식 — 전체 가이드

AI 코드 에이전트로 웹 · API · DB 3티어 todo 앱을 만들고, 같은 앱을 VM에 세 방식으로 배포합니다. 이 문서 하나만 있으면 어느 프로젝트에서든 처음부터 끝까지 따라 할 수 있게 썼습니다. 앱 코드와 설정 파일은 모두 문서 안에 있습니다.

| 단계 | 하는 일 |
|---|---|
| [0. 준비](#0-준비) | VM 접속, Docker · 에이전트 설치 |
| [1. 앱 만들기](#1-앱-만들기) | 에이전트로 3티어 todo 만들기, 기준 코드 |
| [2. 배포 ① VM에 직접 설치](#2-배포--vm에-직접-설치) | apt, systemd, nginx |
| [3. 배포 ② Compose](#3-배포--compose) | 컨테이너 세 개 |
| [4. 배포 ③ Kubernetes](#4-배포--kubernetes) | kind, Ingress, Secret, PVC |
| [5. 세 방식 비교](#5-세-방식-비교) | 같은 고장을 내고 결과 비교 |
| [6. 정리](#6-정리) | 지우기 |

**앱 구조** — 브라우저 → `web`(nginx, 80번) → `/api/`만 `api`(Node.js Express, 3000번)로 → `db`(PostgreSQL, 5432번)

**검증 환경** — Ubuntu 24.04, 4코어, 메모리 16GB, 디스크 80GB VM. 출력은 이 환경에서 실제로 나온 값입니다(2026-11). 버전 숫자 · ID · 시간은 달라도 됩니다.

---

## 0. 준비

### 0-1. VM에 접속하기

내 노트북의 `~/.ssh/config`에 VM을 등록합니다. `<VM 주소>`, `<사용자>`는 받은 값으로 바꿉니다.

```
Host myvm
  HostName <VM 주소>
  User <사용자>
```

```bash
ssh myvm
```

- VS Code를 쓰면 확장 **Remote - SSH** 설치 → 왼쪽 아래 `><` → **Connect to Host** → `myvm`. 이후 VS Code의 터미널이 VM 터미널입니다
- 이 문서의 명령은 **모두 VM 터미널**에서 칩니다

### 0-2. Docker 설치

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

**여기서 VM 접속을 끊고 다시 접속합니다.** 그래야 `sudo` 없이 `docker`를 쓸 수 있습니다.

```bash
docker run --rm hello-world | grep Hello
docker compose version
```

**이렇게 나오면 성공**

```
Hello from Docker!
Docker Compose version v5.6.1
```

### 0-3. AI 코드 에이전트 설치

둘 중 하나를 설치합니다. 계정이 필요합니다.

| | Claude Code | Codex |
|---|---|---|
| 설치 | `curl -fsSL https://claude.ai/install.sh \| bash` | `curl -fsSL https://chatgpt.com/codex/install.sh \| sh` |
| 확인 | `claude --version` | `codex --version` |
| 로그인 | `claude` 실행 → 로그인 화면에서 `c`로 주소 복사 → 노트북 브라우저에서 열기 → 나온 코드를 터미널에 붙여 넣기 | `codex login --device-auth` → 나온 주소를 노트북 브라우저에서 열고 코드 입력 |

VM에는 브라우저가 없어서 인증은 노트북 브라우저에서 합니다.

---

## 1. 앱 만들기

### 1-1. 작업 폴더와 규칙 파일

```bash
mkdir -p ~/todo && cd ~/todo && git init -q
printf 'node_modules\n.env\n' > .gitignore
cat > CLAUDE.md <<'EOF'
# todo
- Node 22, Express 5, CommonJS. TypeScript 쓰지 않음
- 구조: api/ (포트 3000), web/ (정적 파일 + nginx)
- DB는 PostgreSQL, 연결 정보는 환경변수 DB_HOST, DB_USER, DB_PASSWORD, DB_NAME
- 비밀번호를 코드에 쓰지 않음
- 요청한 기능만 만듦
EOF
cp CLAUDE.md AGENTS.md
```

`CLAUDE.md`는 Claude Code가, `AGENTS.md`는 Codex가 읽는 규칙 파일입니다. 매번 말하지 않아도 이 규칙대로 만듭니다.

### 1-2. 에이전트에게 요청하기

`~/todo`에서 `claude` 또는 `codex`를 실행하고, **계획부터** 보게 합니다(Claude Code는 `Shift+Tab`으로 plan 모드, Codex는 `/plan`).

```
todo 앱을 만들어 줘.
- api/: Express. GET /api/todos, POST /api/todos(title), DELETE /api/todos/:id, GET /api/health
- 시작할 때 todos 테이블이 없으면 만듦. DB가 늦게 뜰 수 있으니 재시도
- web/: index.html, app.js. 목록 보기, 추가, 삭제. API는 같은 주소의 /api로 호출
- 각 폴더에 Dockerfile. web은 nginx 이미지이고 /api/ 요청을 api:3000으로 넘김
```

- 계획에 요청하지 않은 기능(로그인 등)이 있으면 빼 달라고 합니다
- 명령 실행 승인을 물으면 **무슨 명령인지 읽고 설명할 수 있을 때만** 승인합니다
- 다 만들면 바로 실행하지 말고 물어봅니다: `추가 버튼을 누르면 DB까지 어떤 순서로 가는지 파일과 줄 번호로 설명해 줘`
- 새 대화를 열어 검토를 받습니다: `git diff를 읽고 버그와 보안 문제만 찾아 줘. 고치지는 말고 목록만 보여 줘`

### 1-3. 기준 코드

세 배포를 같은 코드로 비교하려면 아래 기준 코드를 씁니다. 에이전트가 만든 코드가 동작하지 않을 때도 이것으로 바꿉니다. 블록을 통째로 복사해 붙여 넣으면 파일이 만들어집니다.

```bash
mkdir -p ~/todo/api ~/todo/web && cd ~/todo

cat > api/package.json <<'EOF'
{
  "name": "todo-api",
  "version": "1.0.0",
  "private": true,
  "main": "app.js",
  "scripts": { "start": "node app.js" },
  "dependencies": { "express": "5.1.0", "pg": "8.16.3" }
}
EOF

cat > api/app.js <<'EOF'
const express = require('express');
const { Pool } = require('pg');

const pool = new Pool({
  host: process.env.DB_HOST || 'localhost',
  user: process.env.DB_USER || 'todo',
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME || 'todo',
  connectionTimeoutMillis: 3000,
});

// DB가 재시작되면 쉬고 있던 연결이 끊긴다. 로그만 남기고 계속 돈다(다음 요청 때 새로 연결)
pool.on('error', (err) => console.log(`database connection lost: ${err.message}`));

const app = express();
app.use(express.json());

app.get('/api/health', (req, res) => res.json({ ok: true }));

app.get('/api/todos', async (req, res) => {
  const { rows } = await pool.query('SELECT id, title FROM todos ORDER BY id');
  res.json(rows);
});

app.post('/api/todos', async (req, res) => {
  const title = String(req.body.title || '').trim().slice(0, 200);
  if (!title) return res.status(400).json({ error: 'title is required' });
  const { rows } = await pool.query('INSERT INTO todos (title) VALUES ($1) RETURNING id, title', [title]);
  res.status(201).json(rows[0]);
});

app.delete('/api/todos/:id', async (req, res) => {
  await pool.query('DELETE FROM todos WHERE id = $1', [Number(req.params.id)]);
  res.status(204).end();
});

// DB가 아직 준비 안 됐을 수 있어 몇 번 다시 시도한다
async function start() {
  for (let i = 1; i <= 15; i++) {
    try {
      await pool.query('CREATE TABLE IF NOT EXISTS todos (id SERIAL PRIMARY KEY, title TEXT NOT NULL)');
      console.log(`connected to database at ${pool.options.host}`);
      app.listen(3000, () => console.log('todo api listening on port 3000'));
      return;
    } catch (err) {
      console.log(`waiting for database at ${pool.options.host} (${i}/15): ${err.message}`);
      await new Promise((r) => setTimeout(r, 2000));
    }
  }
  console.log('could not connect to database');
  process.exit(1);
}

start();
EOF

cat > api/Dockerfile <<'EOF'
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
EOF
echo node_modules > api/.dockerignore

cat > web/index.html <<'EOF'
<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8">
  <title>todo</title>
  <style>
    body { font-family: sans-serif; max-width: 480px; margin: 40px auto; }
    li { margin: 6px 0; }
    button { margin-left: 8px; }
  </style>
</head>
<body>
  <h1>할 일</h1>
  <form id="add">
    <input id="title" placeholder="할 일을 입력" required>
    <button>추가</button>
  </form>
  <ul id="list"></ul>
  <p id="status"></p>
  <script src="app.js"></script>
</body>
</html>
EOF

cat > web/app.js <<'EOF'
const list = document.getElementById('list');
const status = document.getElementById('status');

async function load() {
  try {
    const res = await fetch('/api/todos');
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const todos = await res.json();
    list.innerHTML = '';
    for (const t of todos) {
      const li = document.createElement('li');
      li.textContent = t.title;
      const del = document.createElement('button');
      del.textContent = '삭제';
      del.onclick = async () => { await fetch(`/api/todos/${t.id}`, { method: 'DELETE' }); load(); };
      li.appendChild(del);
      list.appendChild(li);
    }
    status.textContent = `${todos.length}개`;
  } catch (err) {
    status.textContent = `api 연결 실패: ${err.message}`;
  }
}

document.getElementById('add').onsubmit = async (e) => {
  e.preventDefault();
  const input = document.getElementById('title');
  await fetch('/api/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title: input.value }),
  });
  input.value = '';
  load();
};

load();
EOF

cat > web/default.conf <<'EOF'
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    # /api/로 오는 요청은 api 컨테이너(3000번)로 넘긴다
    location /api/ {
        proxy_pass http://api:3000;
    }
}
EOF

cat > web/Dockerfile <<'EOF'
FROM nginx:1.29-alpine
COPY default.conf /etc/nginx/conf.d/default.conf
COPY index.html app.js /usr/share/nginx/html/
EOF

git add -A && git commit -qm "todo 3티어" && find api web -type f | sort
```

**이렇게 나오면 성공**

```
api/.dockerignore
api/Dockerfile
api/app.js
api/package.json
web/Dockerfile
web/app.js
web/default.conf
web/index.html
```

`git commit`에서 이름을 묻는 오류가 나면 `git config --global user.name "<이름>"`, `git config --global user.email "<이메일>"`을 먼저 칩니다.

---

## 2. 배포 ① VM에 직접 설치

컨테이너 없이 PostgreSQL · Node.js · nginx를 VM에 직접 설치하고 systemd로 띄웁니다.

### 2-1. 설치

```bash
sudo apt-get update
sudo apt-get install -y postgresql nginx nodejs npm
node -v; psql --version; nginx -v
```

**이렇게 나오면 성공**

```
v18.19.1
psql (PostgreSQL) 16.15 (Ubuntu 16.15-0ubuntu0.24.04.1)
nginx version: nginx/1.24.0 (Ubuntu)
```

Node가 22가 아니라 18입니다. **직접 설치하면 버전이 OS 패키지를 따라갑니다.**

### 2-2. DB 만들기

```bash
DB_PW=$(openssl rand -hex 12)
sudo -u postgres psql -c "CREATE USER todo PASSWORD '$DB_PW'"
sudo -u postgres psql -c "CREATE DATABASE todo OWNER todo"
```

**이렇게 나오면 성공**: `CREATE ROLE`, `CREATE DATABASE`. 비밀번호는 `openssl rand`로 만들어 변수에만 둡니다.

**2-3까지 같은 터미널에서 이어서 칩니다.** 접속이 끊기면 `$DB_PW`가 사라져 `api.env`에 빈 비밀번호가 들어갑니다. 끊겼다면 `DB_PW=$(openssl rand -hex 12)` 후 `sudo -u postgres psql -c "ALTER USER todo PASSWORD '$DB_PW'"`로 다시 맞춥니다.

### 2-3. api를 systemd 서비스로

```bash
sudo mkdir -p /opt/todo /etc/todo /var/www/todo
sudo cp -r ~/todo/api /opt/todo/api
cd /opt/todo/api && sudo npm install --omit=dev
printf 'DB_HOST=127.0.0.1\nDB_USER=todo\nDB_NAME=todo\nDB_PASSWORD=%s\n' "$DB_PW" | sudo tee /etc/todo/api.env >/dev/null
sudo chmod 600 /etc/todo/api.env
sudo tee /etc/systemd/system/todo-api.service >/dev/null <<'EOF'
[Unit]
Description=todo api
After=network.target postgresql.service

[Service]
User=www-data
WorkingDirectory=/opt/todo/api
EnvironmentFile=/etc/todo/api.env
ExecStart=/usr/bin/node app.js
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now todo-api
sleep 3; curl -s localhost:3000/api/health; echo
```

- `Restart=on-failure`: 프로세스가 비정상 종료되면 2초 뒤 다시 띄움
- `EnvironmentFile`: 비밀번호는 root만 읽을 수 있는 파일에 둠

**이렇게 나오면 성공**: `{"ok":true}`

### 2-4. nginx

```bash
sudo cp ~/todo/web/index.html ~/todo/web/app.js /var/www/todo/
sudo tee /etc/nginx/sites-available/todo >/dev/null <<'EOF'
server {
    listen 80 default_server;

    root /var/www/todo;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:3000;
    }
}
EOF
sudo ln -sf /etc/nginx/sites-available/todo /etc/nginx/sites-enabled/todo
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
sleep 1
curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"VM에서 추가"}' localhost/api/todos; echo
curl -s localhost/api/todos; echo
```

**이렇게 나오면 성공**

```
{"id":1,"title":"VM에서 추가"}
[{"id":1,"title":"VM에서 추가"}]
```

브라우저로 보려면 VS Code 아래쪽 **PORTS** 탭에서 80번을 추가하고 나온 주소를 엽니다.

<details><summary>이렇게 나오면?</summary>

| 화면 | 이유 | 대처 |
|---|---|---|
| `Job for nginx.service failed` | 80번을 다른 프로그램이 씀 | `sudo ss -ltnp \| grep :80`으로 찾아 끔 |
| nginx 404 페이지 | 설정을 다시 읽기 전에 요청함 | 1초 뒤 다시 |
| `/api/todos`가 502 | api가 꺼짐 | `sudo journalctl -u todo-api -n 20` |

</details>

### 2-5. 끄기

다음 방식도 80번을 쓰므로 끕니다.

```bash
sudo systemctl disable --now todo-api nginx
sudo systemctl stop postgresql@16-main
sudo systemctl disable postgresql
```

`sudo systemctl stop postgresql`만 치면 실제 DB(`postgresql@16-main`)는 꺼지지 않습니다.

---

## 3. 배포 ② Compose

같은 코드를 컨테이너 세 개로 띄웁니다.

### 3-1. compose.yaml과 비밀번호 파일

```bash
cd ~/todo
cat > compose.yaml <<'EOF'
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: todo
      POSTGRES_DB: todo
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: unless-stopped

  api:
    build: ./api
    image: todo-api:v1
    environment:
      DB_HOST: db
      DB_PASSWORD: ${DB_PASSWORD}
    depends_on:
      - db
    restart: unless-stopped

  web:
    build: ./web
    image: todo-web:v1
    ports:
      - "80:80"
    depends_on:
      - api
    restart: unless-stopped

volumes:
  db-data:
EOF
printf 'DB_PASSWORD=%s\n' "$(openssl rand -hex 12)" > .env
chmod 600 .env
```

`.env`는 1-1에서 `.gitignore`에 넣었으므로 Git에 올라가지 않습니다.

### 3-2. 띄우기

```bash
docker compose up -d --build
# DB 첫 초기화 동안 api가 재시도 중일 수 있어 health가 뜰 때까지 기다린다
for i in $(seq 1 30); do curl -sf localhost/api/health >/dev/null && break; sleep 1; done
docker compose ps --format "table {{.Service}}\t{{.Status}}\t{{.Ports}}"
curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"compose에서 추가"}' localhost/api/todos; echo
curl -s localhost/api/todos; echo
```

**이렇게 나오면 성공**

```
SERVICE   STATUS         PORTS
api       Up 3 seconds   3000/tcp
db        Up 3 seconds   5432/tcp
web       Up 3 seconds   0.0.0.0:80->80/tcp, [::]:80->80/tcp
{"id":1,"title":"compose에서 추가"}
[{"id":1,"title":"compose에서 추가"}]
```

- 밖으로 열린 포트(`0.0.0.0`)는 web 하나뿐입니다. api · db는 밖에서 바로 닿지 않습니다
- 2의 항목이 없습니다. DB가 다른 곳입니다

### 3-3. 내리기

```bash
docker compose down
```

`down`은 컨테이너만 지웁니다. 데이터는 볼륨 `db-data`에 남습니다. `down -v`를 붙이면 데이터까지 지웁니다.

---

## 4. 배포 ③ Kubernetes

같은 이미지를 kind(도커 안에 띄우는 쿠버네티스) 클러스터에 올립니다.

### 4-1. 도구 설치

```bash
cd /tmp
curl -fsSLo kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
curl -fsSLo kubectl https://dl.k8s.io/release/v1.37.0/bin/linux/amd64/kubectl
curl -fsSL https://get.helm.sh/helm-v4.3.0-linux-amd64.tar.gz | tar xz linux-amd64/helm
sudo install kind kubectl linux-amd64/helm /usr/local/bin/
kind version; kubectl version --client | head -1; helm version --short
```

### 4-2. 클러스터와 Ingress 컨트롤러

```bash
mkdir -p ~/todo/k8s && cd ~/todo/k8s
cat > kind.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: todo
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
EOF
cat > traefik-values.yaml <<'EOF'
service:
  spec:
    type: ClusterIP
ports:
  web:
    hostPort: 80
nodeSelector:
  ingress-ready: "true"
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
ingressClass:
  isDefaultClass: true
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 0
EOF
kind create cluster --config kind.yaml
helm upgrade --install traefik traefik --repo https://traefik.github.io/charts --version 41.6.0 \
  -n traefik --create-namespace -f traefik-values.yaml --wait
kubectl get nodes; kubectl get ingressclass
```

- `extraPortMappings`: VM의 80번을 kind 노드의 80번으로 이음
- Traefik: Ingress 규칙대로 요청을 Service로 보내는 컨트롤러

**이렇게 나오면 성공**: 노드가 `Ready`, 마지막에 `traefik` 줄이 보임.

### 4-3. 이미지 넣기

```bash
kind load docker-image todo-api:v1 todo-web:v1 --name todo
```

kind 노드는 VM의 도커와 이미지를 함께 쓰지 않습니다. 3에서 만든 이미지를 노드에 넣습니다.

### 4-4. 매니페스트

```bash
cd ~/todo/k8s
cat > db.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: postgres
          image: postgres:17-alpine
          env:
            - name: POSTGRES_USER
              value: todo
            - name: POSTGRES_DB
              value: todo
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: todo-secret
                  key: DB_PASSWORD
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: db-data
---
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  selector:
    app: db
  ports:
    - port: 5432
EOF
cat > api.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: todo-api:v1
          env:
            - name: DB_HOST
              value: db
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: todo-secret
                  key: DB_PASSWORD
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: api
spec:
  selector:
    app: api
  ports:
    - port: 3000
EOF
cat > web.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: todo-web:v1
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: todo
spec:
  rules:
    - host: todo.localhost
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
EOF
```

| 파일 | 리소스 | 하는 일 |
|---|---|---|
| `db.yaml` | PVC · Deployment · Service | DB 데이터를 파드 밖 디스크에 둠. `db` 이름으로 찾게 함. `Recreate`로 DB 파드 두 개가 같은 디스크를 동시에 쓰지 않게 함 |
| `api.yaml` | Deployment(2개) · Service | `readinessProbe`가 통과한 파드에만 요청을 보냄 |
| `web.yaml` | Deployment · Service · Ingress | `todo.localhost`로 온 요청을 web으로 |

### 4-5. Secret과 배포

```bash
cd ~/todo/k8s
kubectl create secret generic todo-secret --from-literal=DB_PASSWORD="$(openssl rand -hex 12)"
kubectl apply -f db.yaml -f api.yaml -f web.yaml
kubectl rollout status deploy/db deploy/api deploy/web --timeout=180s
kubectl get pods,ingress,pvc
curl -s -X POST -H 'Content-Type: application/json' -d '{"title":"k8s에서 추가"}' http://todo.localhost/api/todos; echo
curl -s http://todo.localhost/api/todos; echo
```

**이렇게 나오면 성공**: `api` 파드 2개 · `db` 1개 · `web` 1개가 `Running`, PVC가 `Bound`, 마지막 줄이 `[{"id":1,"title":"k8s에서 추가"}]`

- 요청 흐름: Ingress(Traefik) → Service `web` → nginx → Service `api` → 파드 → Service `db` → 파드
- `todo.localhost`는 따로 설정하지 않아도 VM 자신(127.0.0.1)을 가리킵니다

<details><summary>이렇게 나오면?</summary>

| 화면 | 이유 | 대처 |
|---|---|---|
| `ImagePullBackOff` | 4-3을 건너뜀 | 4-3 후 `kubectl rollout restart deploy/api deploy/web` |
| `api`가 계속 `0/1` | Secret을 다시 만들어 PVC 안의 옛 비밀번호와 다름 | `kubectl delete -f db.yaml -f api.yaml -f web.yaml`, `kubectl delete secret todo-secret` 후 4-5 다시 |
| `kind create cluster`가 80번 오류 | 2 · 3을 안 끔 | 2-5, 3-3 |

</details>

---

## 5. 세 방식 비교

같은 고장을 내고 결과를 봅니다. 지금 Kubernetes가 떠 있습니다.

### 5-1. Kubernetes

```bash
kubectl delete pod $(kubectl get pod -l app=api -o name | head -1 | cut -d/ -f2)
for i in $(seq 1 30); do curl -s -o /dev/null -w '%{http_code} ' http://todo.localhost/api/todos; sleep 0.1; done; echo
kubectl delete pod -l app=db
kubectl rollout status deploy/db --timeout=60s
sleep 3; curl -s http://todo.localhost/api/todos; echo
kubectl scale deploy/api --replicas=3
kubectl rollout status deploy/api --timeout=60s
kubectl get endpointslices -l kubernetes.io/service-name=api
```

**이렇게 나오면 성공**: `200`이 30개, DB 파드를 지워도 항목이 남음, `ENDPOINTS`에 IP 3개

Compose와 80번이 겹치므로 클러스터를 지웁니다.

```bash
kind delete cluster --name todo
```

### 5-2. Compose

```bash
cd ~/todo && docker compose up -d
sleep 3
sudo kill -9 $(docker inspect -f '{{.State.Pid}}' $(docker compose ps -q api))
sleep 3; curl -s -o /dev/null -w '%{http_code}\n' localhost/api/todos
docker compose ps api --format '{{.Status}}'
docker kill $(docker compose ps -q api)
sleep 5; docker compose ps -a api --format '{{.State}} {{.Status}}'
docker compose up -d api
docker compose up -d --scale api=3
docker compose restart web
sleep 2; docker compose ps api --format '{{.Name}} {{.Status}}'
docker compose down -v
```

**이렇게 나오면 성공**: 프로세스가 비정상 종료되면 다시 뜨고(`200`), 사람이 `docker kill`로 끄면 `exited`로 남음

### 5-3. VM 직접 설치

```bash
sudo systemctl start postgresql@16-main todo-api nginx
sleep 3
sudo kill -9 $(systemctl show -p MainPID --value todo-api); sleep 4
systemctl is-active todo-api
sudo systemctl stop postgresql@16-main
curl -s -o /dev/null -w '%{http_code}\n' localhost/api/todos
sudo systemctl start postgresql@16-main
sleep 2; curl -s -o /dev/null -w '%{http_code}\n' localhost/api/todos
```

**이렇게 나오면 성공**: `active`, DB를 끈 동안 `500`, 켜면 `200`

### 5-4. 실측 비교

| 시험 | ① VM 직접 | ② Compose | ③ Kubernetes |
|---|---|---|---|
| api 비정상 종료 | systemd가 2.6초 뒤 다시 띄움 | 1.4초 뒤 다시 뜸 | 파드 2개 중 1개를 지우는 동안 요청 30/30 성공 |
| 사람이 끔 | `systemctl stop` → 그대로 꺼짐 | `docker kill` → `Exited (137)`로 남음 | `kubectl delete pod` → 새 파드가 뜸 |
| DB 데이터 | 재부팅 뒤 남음 | `down` 뒤 남음, `down -v`면 지워짐 | DB 파드를 지워도 남음(PVC) |
| api IP가 바뀌면 | 해당 없음(127.0.0.1) | **502가 이어짐 → web 재시작으로 해결** | 해당 없음(Service IP 고정) |
| api 3개로 늘리기 | 포트를 나누고 nginx 설정을 손으로 | `--scale api=3` + web 재시작 | `kubectl scale` 한 줄, 등록도 자동 |
| 설정 위치 | `/etc` 아래 여러 곳 | `compose.yaml` 하나 | YAML 파일들 |

**Compose의 502**: nginx는 `api`라는 이름을 시작할 때 한 번만 IP로 바꿔 둡니다. api 컨테이너가 다른 IP로 다시 만들어지면 nginx는 옛 IP로 계속 보내 502가 납니다. Kubernetes는 nginx가 가리키는 Service IP가 바뀌지 않아 이 문제가 없습니다. `--scale` 뒤 web을 재시작한 것도 같은 이유입니다.

**코드 검토 소재**: DB를 끈 동안 나온 `500` 응답에는 `ECONNREFUSED 127.0.0.1:5432`와 서버 안의 파일 경로가 그대로 나옵니다. 기준 코드가 DB 오류를 처리하지 않기 때문입니다. 실제 서비스라면 고쳐야 할 부분입니다.

---

## 6. 정리

```bash
sudo systemctl disable --now todo-api nginx
sudo systemctl stop postgresql@16-main
sudo systemctl disable postgresql
cd ~/todo && docker compose down -v 2>/dev/null
kind delete cluster --name todo 2>/dev/null
sudo ss -ltn | grep -E ':(80|3000|5432) ' || echo 포트-비었음
```

**이렇게 나오면 성공**: 마지막 줄이 `포트-비었음`
