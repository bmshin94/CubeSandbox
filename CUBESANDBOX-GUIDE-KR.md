# CubeSandbox 완전정복 가이드 (한국어 정리본)

> AI 에이전트용 샌드박스 플랫폼 **CubeSandbox**를 처음 접한 사람 눈높이로 정리한 문서입니다.
> 설치법 / 정체 / 인증 / 활용 / 수익화까지 한 번에 볼 수 있게 만들었습니다.

---

## 📌 관련 링크

| 구분 | 주소 |
|---|---|
| 🍴 **내 저장소 (Fork)** | https://github.com/bmshin94/CubeSandbox |
| ⭐ **원본 저장소 (Upstream)** | https://github.com/TencentCloud/CubeSandbox |
| 🐍 PyPI (Python SDK) | https://pypi.org/project/cubesandbox/ |
| 💬 Discord 커뮤니티 | https://discord.gg/kkapzDXShb |
| 🐦 X (Twitter) | https://x.com/CubeSandbox_AI |
| 🌏 CNCF Landscape 등재 | https://landscape.cncf.io/?item=ai-native-infra--workload-runtime--cubesandbox |
| 📋 Cube 100 프로그램 신청 | https://forms.gle/Wjy53fksUoc3y4388 |

### 저장소 내부 주요 문서
- `README.md` / `README_zh.md` — 프로젝트 소개
- `docs/guide/quickstart.md` — 퀵스타트 (여기부터 시작!)
- `docs/guide/bare-metal-deploy.md` — 베어메탈 설치
- `docs/guide/authentication.md` — 인증 설정
- `docs/guide/lifecycle.md` — Auto-Pause / Auto-Resume
- `docs/guide/security-proxy.md` — 보안 프록시 & 크레덴셜 금고
- `docs/guide/cube100.md` — Cube 100 프로그램
- `docs/architecture/overview.md` — 아키텍처 전체 구조
- `openapi.yml` — REST API 명세 (PHP 등 SDK 없는 언어용)
- `examples/` — 실전 예제 19종

---

## 1. 이게 뭔가요? 🧊

한 줄 요약:

> **AI가 만든 코드를 "안전하게, 엄청 빠르게" 격리 실행해주는 서버 플랫폼**
> = *AI에게 빌려주는 일회용 컴퓨터 자판기*

텐센트 클라우드(TencentCloud)가 만든 오픈소스이며, RustVMM + KVM 기반의 초경량 MicroVM을 씁니다.

### 핵심 스펙

| 항목 | 성능 |
|---|---|
| 부팅 속도 | **60ms 이하** (50개 동시 생성 시 평균 67ms, P99 137ms) |
| 격리 수준 | **하드웨어 레벨** — 샌드박스마다 전용 리눅스 커널 |
| 메모리 오버헤드 | **5MB 미만** → 서버 1대에 수천 개 |
| 호환성 | **E2B SDK 드롭인 호환** (환경변수만 바꾸면 이전 완료) |
| 라이선스 | **Apache 2.0** (상업적 이용 자유) |

### 왜 필요한가?

AI 챗봇/에이전트가 파이썬 코드를 짜줬을 때, **그 코드를 어디서 실행할 것인가?** 가 문제입니다.

- 내 서버에서 그냥 실행 → `rm -rf /`, 자격증명 탈취, 사용자 간 데이터 유출 위험 💀
- 그래서 **격리된 일회용 방**이 필요하고, 그 방을 만들어주는 게 CubeSandbox

### 기존 방식과 비교

| | 비유 | 특징 |
|---|---|---|
| **Docker** | 원룸 고시원 (천장 뚫림) | 빠르고 싸지만 커널 공유 → 보안 약함 |
| **일반 VM** | 단독주택 | 안전하지만 부팅 수 초, 메모리 과다 |
| **CubeSandbox** | 방음·방화 완벽한 조립식 캡슐호텔 | **0.06초 + 5MB** → 빠른데 안전 |

공식 벤치마크 표:

| 지표 | Docker | 전통 VM | CubeSandbox |
|---|---|---|---|
| 격리 수준 | 낮음(커널 공유) | 높음 | **최상 (전용 커널 + eBPF)** |
| 부팅 속도 | 200ms | 수 초 | **< 60ms** |
| 메모리 오버헤드 | 낮음 | 높음 | **< 5MB** |
| 배포 밀도 | 높음 | 낮음 | **노드당 수천 개** |
| E2B SDK 호환 | / | / | **✅ 드롭인** |

---

## 2. 폴더 구조 = 부품 설명서 📂

호텔에 비유한 각 컴포넌트 역할:

```
🏨 큐브 호텔

[컨트롤 플레인 = 명령 내리는 쪽]
CubeAPI/             🛎️ 프론트 데스크 — E2B 호환 REST API 게이트웨이 (Rust/Axum)
CubeMaster/          🧑‍💼 지배인 — 클러스터 스케줄러, 노드 배치 결정 (Go)
CubeOps/             📋 운영 관리 — 노드 관리 (v0.7에서 분리)
CubeTemplateCenter/  📦 템플릿 저장소 (Go)
web/                 📱 웹 관리 콘솔 (:12088, React + Vite)

[데이터 플레인 = 실제로 굴리는 쪽]
Cubelet/             🧹 노드별 매니저 — 샌드박스 생명주기 전담 (Go)
CubeShim/            🌉 containerd Shim v2 구현 — 런타임 ↔ MicroVM 연결 (Rust)
hypervisor/          🧱 핵심! RustVMM + KVM 기반 MicroVM 엔진 (Rust)
agent/               🤖 VM 내부 PID 1 에이전트 (Rust)
cubecow/             📸 CoW 스냅샷 엔진 — XFS reflink로 O(1) 스냅샷 (Rust)
CubeNet/             🔌 eBPF 기반 커널 레벨 네트워크 격리 (CubeVS)
CubeEgress/          🚪 경비원 — L7 도메인 필터링 + 크레덴셜 주입 (OpenResty/Lua)
CubeProxy/           🚦 리버스 프록시 — 샌드박스로 요청 라우팅 (OpenResty)
CubeS3lvol/          ☁️ S3 백엔드 볼륨 (크로스 노드 스냅샷)
cube-lifecycle-manager/ ⏸️ Auto-Pause/Resume 코디네이터 (Go)

[내가 실제로 쓸 것들]
sdk/                 🎮 리모컨 — Python / Node / Go SDK  ⭐가장 중요
examples/            📖 실전 예제 19종
docs/                📚 매뉴얼 (영어 + 중국어)
deploy/              🚀 원클릭 설치 / K8s / Terraform
.claude/             🤖 이 프로젝트도 Claude Code 세팅 되어있음 (리뷰 에이전트 6개)
```

> 💡 부품은 몰라도 됩니다. **`sdk/`와 `examples/`만 알면 사용 가능**합니다.

### 요청 흐름

```
Client/SDK ──E2B 호환 REST──► CubeAPI ──gRPC──► CubeMaster ──gRPC──► Cubelet
                                                                        │
                                                        containerd Shim v2 │
                                                                        ▼
                                                    CubeShim ──KVM──► CubeHypervisor
                                                                        │
                                                                        ▼
                                                                 MicroVM (샌드박스)
```

---

## 3. 설치 및 사용법 🛠️

### 준비물

```
❌ 맥 / 윈도우 노트북 → 직접 설치 불가
✅ 리눅스 서버 (x86_64)
   - CPU 4코어 이상 / RAM 8GB 이상 / 디스크 50GB 이상 (권장 32코어/64GB/200GB)
   - root 권한 필수
   - 추천 OS: OpenCloudOS 9, TencentOS 4 (XFS 기본)
   - Ubuntu/Debian은 /data/cubelet 에 XFS 수동 마운트 필요 (issue #311 참고)
   - glibc 2.31 이상 필수
```

> ⚠️ `/dev/kvm` 없는 일반 클라우드 VM이면 **PVM 커널**을 설치해 KVM을 활성화할 수 있습니다.
> ARM64(aarch64)는 PVM 미지원 → 네이티브 KVM 되는 물리 서버에서 베어메탈 설치.

### 설치 4단계

**① root 전환**
```bash
sudo su root
```

**② 원클릭 설치 (한 줄)**
```bash
curl -sL https://cnb.cool/CubeSandbox/CubeSandbox/-/git/raw/master/deploy/one-click/online-install.sh | MIRROR=cn bash
```
이 한 줄로 CubeAPI(:3000), CubeMaster, Cubelet, CubeShim, MySQL, Redis, CubeProxy, CoreDNS까지 전부 자동 설치됩니다.

**③ 템플릿 생성** (샌드박스의 "붕어빵 틀")
```bash
cubemastercli tpl create-from-image \
  --image cube-sandbox-int.tencentcloudcr.com/cube-sandbox/sandbox-code:latest \
  --writable-layer-size 1G \
  --expose-port 49999 \
  --expose-port 49983 \
  --probe 49999
```
> 템플릿 = 파이썬/라이브러리가 미리 깔린 **스냅샷 사진**. 이 사진을 복사해서 방을 찍어내기 때문에 0.06초가 가능합니다.
> 중국 리전은 `cube-sandbox-cn.tencentcloudcr.com/...` 사용.

**④ 웹 콘솔 확인**
```
http://<서버IP>:12088
```
Overview에서 노드 Ready 확인 → Template Store에서 프리셋 설치 → Sandboxes에서 생성 + 실시간 로그.

### 실제 사용 (내 로컬에서)

```bash
pip install cubesandbox

export CUBE_API_URL=http://<서버IP>:3000
export CUBE_TEMPLATE_ID=<템플릿 ID>
export CUBE_PROXY_NODE_IP=<CubeProxy 노드 IP>   # 원격 접속 시 필요
```

```python
from cubesandbox import Sandbox

with Sandbox.create() as sb:
    print(sb.run_code("1 + 1").text)            # "2"
    print(sb.commands.run("ls -la").stdout)     # 쉘 명령도 가능
# with 블록을 나오면 샌드박스 자동 삭제
```

### E2B SDK를 그대로 쓰는 방법 (마이그레이션)

```bash
pip install e2b-code-interpreter
export E2B_API_URL="http://127.0.0.1:3000"     # ← 이 줄만 바꾸면 이전 완료
export E2B_API_KEY="e2b_000000"
export CUBE_TEMPLATE_ID="<템플릿 ID>"
export SSL_CERT_FILE="/root/.local/share/mkcert/rootCA.pem"
```

```python
from e2b_code_interpreter import Sandbox   # E2B SDK를 그대로 사용!

with Sandbox.create(template=os.environ["CUBE_TEMPLATE_ID"]) as sandbox:
    print(sandbox.run_code("print('Hello from Cube Sandbox!')"))
```

### 언어별 SDK

| 언어 | 설치 | 상태 |
|---|---|---|
| 🐍 Python | `pip install cubesandbox` | ✅ 공식 |
| 🟢 Node/TS | `npm install @cubesandbox/sdk` | ✅ 공식 (Node 18+) |
| 🔵 Go | `sdk/go/` | ✅ 공식 |
| 🐘 PHP | 없음 | ⚠️ REST 직접 호출 (`openapi.yml` 참고) |

---

## 4. 플러그인? 스킬? MCP? 🤔

**셋 다 아닙니다.** 이건 **서버 인프라(백엔드 서비스)** 입니다.

| 종류 | 비유 | 설치 위치 |
|---|---|---|
| 스킬 / 플러그인 | 📝 AI에게 주는 설명서 한 장 | Claude Code 폴더 안 |
| MCP 서버 | 🔌 AI에게 붙이는 USB 확장 도구 | 옆에 도는 작은 프로그램 |
| **CubeSandbox** | 🏭 **공장을 통째로 세우는 것** | **별도의 리눅스 서버** |

### 단, Claude Code와 붙이는 방법은 있습니다 — "훅(Hook)" 방식 ⭐

`examples/claude-code-integration/` 에 구현체가 있습니다. MCP가 아니라 **`PreToolUse` 훅**을 쓰는데, 그 이유가 문서에 명확히 적혀 있습니다:

> "MCP나 SDK 기반 샌드박싱은 에이전트가 샌드박스 도구를 *선택*해야만 동작한다.
> 평범한 `Bash` 호출은 그대로 호스트에서 실행되어 버린다.
> `PreToolUse` 훅은 툴 호출 자체를 가로채므로 **투명하고 완전한 격리**가 된다."

```
Claude Code (호스트)
    ├── Read / Write / Edit ─────────────► 호스트 프로젝트 파일 (그대로)
    │
    └── Bash ──► PreToolUse 훅 ──► cubesandbox_exec ──► CubeAPI ──► MicroVM
                (cubesandbox_rewrite.py)                 (:3000)   └ 세션당 1개 재사용
```

**특징**
- 세션(`session_id`)당 MicroVM 1개를 재사용 → `cd`, 환경변수가 유지됨
- 호스트 프로젝트를 **읽기 전용**으로 마운트 가능
- **Fail-closed** — 안전하게 재작성 못 하면 아예 실행 차단
- **Injection-safe** — 원본 명령을 `shlex` 인용된 단일 인자로 전달

**설치**
```bash
cd examples/claude-code-integration
pip install -r requirements.txt
cp .env.example .env        # CUBE_API_URL, CUBE_TEMPLATE_ID 설정
cd hooks && ./install.sh    # ~/.claude/settings.json 에 자동 등록 (기존 설정 유지)
```

> 💡 원하면 REST API를 감싸는 **MCP 서버를 직접 만들 수도** 있습니다.

---

## 5. API 토큰이 필요한가요? 🔑

두 가지로 나뉩니다.

### A. CubeSandbox 자체 인증 → 기본은 **인증 없음** ⚠️

`docs/guide/authentication.md` 원문:

> "기본값으로 Cube API Server는 **인증 없이 모든 요청을 허용**합니다."

예제의 `E2B_API_KEY="e2b_000000"` 는 **SDK가 빈 값을 거부해서 넣는 더미값**일 뿐입니다.

> 🚨 **절대 주의: 3000번 포트를 인터넷에 그대로 열지 마세요.**
> 아무나 샌드박스를 무한 생성할 수 있습니다 → 요금 폭탄 / 암호화폐 채굴 악용.
> 반드시 방화벽으로 막고, 운영 환경은 인증을 켜야 합니다.
> (`docs/guide/network-hardening.md` 참고)

### 인증 켜는 법 — "콜백 방식"

CubeSandbox는 자체 회원 DB가 없습니다. 대신 **내 인증 서버에 물어보는** 구조입니다.

```
사용자 ──키──► CubeAPI ──POST──► 내 인증 서버 (/verify)
                  ▲                      │
                  └──── 200 OK = 통과 ◄──┘
                       그 외 = 401 차단
```

```bash
export AUTH_CALLBACK_URL=https://my-service.com/verify
# 또는  ./cube-api --auth-callback-url https://my-service.com/verify
```

콜백이 받는 헤더:

| 헤더 | 값 |
|---|---|
| `Authorization` | `Bearer <token>` (Bearer 인증 시) |
| `X-API-Key` | `<key>` (API Key 인증 시) |
| `X-Request-Path` | 원본 요청 경로 (예: `/templates/my-tmpl`) |
| `X-Request-Method` | HTTP 메서드 (GET / POST / DELETE / PATCH) |

> ⚠️ **경로만 검사하면 위험!** 같은 경로에 여러 메서드가 붙어 있어서
> (`/templates/:id` 는 GET/POST/DELETE/PATCH 전부 사용), **경로 + 메서드를 반드시 함께** 검증해야 합니다.
> 안 그러면 읽기 전용 키로 삭제까지 가능해집니다.

✅ 장점: 기존에 만들어 둔 회원 시스템(PHP든 Node든)을 **그대로 재활용** 가능.

### B. 외부 API 토큰 (OpenAI 키 등) → **킬러 기능** ⭐

샌드박스 안에 키를 넣지 않고, **CubeEgress가 나가는 길목에서 주입**합니다.

```
샌드박스: "OpenAI 호출!" (키 없이 그냥 요청)
              ↓
   🚪 CubeEgress(L7 TPROXY)가 가로챔
              ↓
   허용 도메인 확인 → Authorization 헤더 자동 주입
              ↓
         OpenAI 서버 ✅
```

- 샌드박스 내부에서는 **키를 볼 수도 없음** 🔐
- 도메인/경로/메서드/SNI 단위 allow·deny 정책
- 모든 결정을 JSONL 감사 로그로 기록
- 샌드박스는 CubeEgress 루트 CA를 신뢰 → 투명 TLS 검사 가능

---

## 6. 왜 깃허브에서 유명한가? 🌟

1. **타이밍** ⏰ — 전 세계가 AI 에이전트에 몰두 중이고, 누구나 "AI가 짠 코드를 어디서 실행하지?" 벽에 부딪힘
2. **E2B의 오픈소스 대안** 💰 — 유료·해외 클라우드인 E2B를, 코드 수정 없이 자체 서버로 대체
3. **압도적인 숫자** 📊 — 0.06초 / 5MB / 노드당 수천 개
4. **텐센트 백업 + CNCF Landscape 등재** 🏢 — 개인 취미가 아닌 실무 검증 기술
5. **문서와 예제가 진심** 📚 — 영/중 문서, 예제 19종, 원클릭 설치, K8s, Terraform

> `docs/guide/cube100.md` 기준: **오픈소스 공개 80일 만에 깃허브 스타 1만 개 돌파**,
> 11개 릴리스, 글로벌 컨트리뷰터 약 70명, 560+ 커밋.

### 실제 도입 사례 (`docs/guide/usecases/`)

| 사례 | 내용 |
|---|---|
| 🖥️ Lenovo | 클라우드 AI 에이전트 |
| 🎙️ Unisound | AI 강화학습(RL) 롤아웃 환경 |
| 🌐 Lexmount | 브라우저 자동화 에이전트 |
| 📈 Horizon Insights | 데이터 분석 에이전트 |
| 🏗️ Guangdong Rising / Hermes / trpc-agent-go | 기타 프로덕션 사례 |

---

## 7. 로컬 에이전트 구축에 도움이 될까? 🤖

**경우에 따라 다릅니다.**

### ✅ 도움이 되는 경우
- 에이전트에게 `rm`, `pip install`, 네트워크 등 **위험한 권한**을 줘야 할 때
- 에이전트 여러 개를 **동시 병렬**로 굴릴 때 (각자 방 하나씩)
- 실험 실패 시 **0.1초 롤백**이 필요할 때 (스냅샷/클론)
- 나중에 **서비스로 키울** 계획이 있을 때

### ❌ 오버스펙인 경우
- "**나 혼자 내 노트북에서 쓸**" 에이전트 → 서버 빌려야 하고 운영 부담까지, 과합니다.

### 대안 비교

| 대안 | 난이도 | 격리 수준 |
|---|---|---|
| Claude Code 권한 설정 | ⭐ 매우 쉬움 | 낮음 |
| Docker 컨테이너 | ⭐ 쉬움 | 보통 |
| 로컬 VM (UTM/Multipass) | ⭐⭐ 보통 | 높음 (느림) |
| **CubeSandbox** | ⭐⭐⭐⭐ 어려움 | **최고 (빠름)** |

### 결론
```
혼자 쓰는 도구다          → Docker로 충분
남들도 쓰게 할 거다        → CubeSandbox 정답
공부/포트폴리오 목적이다    → 무조건 해볼 가치 있음
```

---

## 8. 수익화 아이디어 💰

### 대원칙 3개

1. **"샌드박스"를 팔지 마라** — 사람들은 격리 환경이 아니라 **자기 문제 해결**에 돈을 낸다. 엔진 말고 자동차를 팔 것.
2. **무료 사용자는 비용 폭탄** — 진짜 적은 경쟁사가 아니라 **암호화폐 채굴러**. 처음부터 로그인 필수 + 사용량 제한 + CPU 감시.
3. **시장은 실재한다** — 위 도입 사례가 증거.

### 모델별 비교

| 모델 | 난이도 | 초기자본 | 수익속도 | 수익크기 | 추천도 |
|---|---|---|---|---|---|
| 🥇 기업 온프레미스 구축/운영 | ⭐⭐⭐ | 적음 | 중간 | 💰💰💰 | ⭐⭐⭐⭐⭐ |
| 🥈 AI 코드실행 SaaS | ⭐⭐⭐⭐ | 중간 | 느림 | 💰💰💰 | ⭐⭐⭐⭐ |
| 🥉 한국어 콘텐츠/브랜딩 | ⭐ | **0원** | 느림 | 💰 | ⭐⭐⭐⭐⭐ |
| 4️⃣ 버티컬 SaaS (틈새) | ⭐⭐⭐ | 중간 | 중간 | 💰💰 | ⭐⭐⭐ |
| 5️⃣ 템플릿/환경 마켓 | ⭐⭐ | 적음 | 느림 | 💰 | ⭐⭐ |
| 6️⃣ 클라우드 서비스 직접 운영 | ⭐⭐⭐⭐⭐ | **많음** | 느림 | 💰💰💰💰 | ⭐⭐ |

---

### 🥇 1위. 국내 기업 온프레미스 구축 & 운영 대행

**왜 1등인가**
```
경영진: "우리도 AI 에이전트 도입해!"
개발팀: "E2B 쓰면 금방..."
보안팀: "❌ 데이터 해외 반출 금지"
법무팀: "❌ 개인정보보호법 위반 소지"
개발팀: "그럼 직접 구축을..." → XFS? eBPF? KVM? PVM?? → 6개월 삽질
                                         ↑ 여기서 내가 등장
```

**타겟 고객**

| 업종 | 왜 필요한가 | 예산 |
|---|---|---|
| 🏦 금융/보험 | 망분리 규제, 외부 전송 불가 | 💰💰💰 |
| 🏥 병원/의료 | 환자정보 = 민감정보 | 💰💰💰 |
| 🏛️ 공공/공기업 | 국내 리전 필수, 보안 인증 | 💰💰 |
| 🏭 제조 대기업 | 설계도면/영업비밀 | 💰💰💰 |
| 🎓 대학/연구소 | 학생 코드 실행 환경 | 💰 |

**수익 구조**
```
초기 구축비     : 프로젝트 단위
월 운영/유지보수 : 매월 안정 수입  ⭐ 핵심!
추가 개발       : 커스텀 건별
교육/기술이전    : 담당자 교육 세션
```

**준비물**
1. 직접 설치하며 삽질 경험 축적 (XFS, 방화벽, 멀티노드)
2. **레퍼런스 1곳** — 무료로라도 구축해서 사례 확보 (가장 중요)
3. 설치 자동화 스크립트 + 운영 매뉴얼 (내 자산)
4. 모니터링/백업 세팅 노하우

**⭐ Cube 100 프로그램 활용**
텐센트가 실제 운영 사례 100팀을 모집 중입니다. 선정 시:
- 공식 사례 페이지에 로고 + 아키텍처 게재, "올해의 사례" 인증서
- Cube 엔지니어팀 1:1 기술 지원, 신기능 얼리액세스
- **생태계 상업 파트너십 우선 소개** ← 영업 기회
- 공식 채널(X, Discord 등) 공동 홍보

→ 국내 첫 사례를 제출하면 **"공식 인정받은 한국 파트너"** 포지션 확보.

---

### 🥈 2위. AI 코드 실행 SaaS

CubeSandbox를 엔진으로 쓰는 **내 서비스**를 만들어 구독료를 받는 모델.

| 아이템 | 왜 CubeSandbox인가 | 타겟 |
|---|---|---|
| 🧑‍🏫 **코딩 교육/자동 채점** | 학생 300명이 무한루프 짜도 서버 안 죽음 | 코딩학원, 대학, 기업 신입교육 |
| 📊 **엑셀 AI 데이터 분석** | 남의 엑셀 = 민감정보, 격리 필수 | 중소기업 사무직, 마케터 |
| 🧪 **코딩테스트 채점 엔진** | 악의적 코드 제출 + 동시 폭주 대응 | 채용하는 IT 기업 (단가 높음) |
| 🤖 **"AI 알바생" 업무 자동화** | 스크립트가 계속 살아있어야 함, 스냅샷 상태 유지 | 소상공인, 1인 기업 |

#### 💸 원가 계산 — Auto-Pause가 사업성 그 자체 ⭐

> ⚠️ 아래는 **가정 기반 추정치**입니다. 실제 단가는 클라우드/리전/스펙에 따라 달라집니다.

가정: 32코어 / 64GB 서버 1대 (월 80만원 가정), 샌드박스당 실사용 메모리 1GB

**❌ Auto-Pause 없으면**
```
64GB ÷ 1GB = 동시 64개가 한계
유저 100명이 각자 샌드박스 1개 → 이미 초과
→ 서버 2대 필요 = 월 160만원
```

**✅ Auto-Pause 켜면**

`docs/guide/lifecycle.md` 원문:
> "유휴 샌드박스는 자동으로 스냅샷 정지되어 **자원 소모 0**, 다음 요청 시 **0.1초** 만에 투명하게 복구"

```
유저 1000명이 가입해도 '지금 이 순간' 실행 중인 사람은 보통 3~5%
→ 1000명 × 5% = 50개만 실제 메모리 차지
→ 서버 1대로 커버

원가  : 80만원 ÷ 1000명 = 인당 월 800원
판매가: 월 9,900원
→ 마진률 90%+
```

> 💡 잠든 샌드박스는 디스크만 쓰고 CPU/RAM은 0입니다. 디스크는 훨씬 저렴합니다.
> **Auto-Pause를 켜지 않으면 원가가 10배가 됩니다. 반드시 켜세요.**
> 설정: `on_timeout="pause"` + `auto_resume=True` (`examples/code-sandbox-quickstart/auto-resume.py` 참고)

---

### 🥉 3위. 한국어 콘텐츠 + 전문가 브랜딩

직접 수익은 느리지만 **1·2위로 가는 최고의 지름길**이며, **자본 0원 / 오늘 시작 가능**.

**지금이 기회인 이유:** CubeSandbox 한국어 자료가 거의 없습니다. 공식 문서도 영어/중국어뿐 → 무주공산.

```
📝 블로그 시리즈 "CubeSandbox 완전정복"
   ① 이게 뭔가요? (쉬운 소개)
   ② 클라우드 서버 설치하기 (삽질 기록 포함 ⭐ 검색 유입 최고)
   ③ Python/Node로 첫 코드 실행
   ④ Claude Code 안전하게 가두기 (훅 연동)
   ⑤ E2B에서 이사하기 (비용 비교)
   ⑥ 실전: 미니 코드 실행 서비스 만들기

🎥 유튜브 — 설치 과정 화면 녹화 10분
🌏 공식 문서 한국어 번역 기여 → 컨트리뷰터 등재
```

**수익으로 이어지는 경로**
```
콘텐츠 → "한국에서 이거 아는 사람" 포지션 선점
   ├─ ① 기업 구축 문의 유입 (1위 모델로 연결)
   ├─ ② 강의 제안 (인프런/패스트캠퍼스)
   ├─ ③ 기술 컨설팅/외주
   └─ ④ 이직/포트폴리오 가치 상승
```

---

### 4위. 버티컬 SaaS (틈새 공략)

일반 코드 실행은 경쟁이 많지만, 좁히면 승산이 올라갑니다.

| 아이디어 | 강점 |
|---|---|
| 🧬 바이오 연구원용 분석 환경 | R/파이썬 패키지 세팅 지옥을 템플릿으로 해결 |
| 📐 건축/설계 스크립트 실행 | 경쟁자 거의 없음 |
| 💹 퀀트 백테스팅 플랫폼 | 전략 코드 = 영업비밀 → 격리 필수 |
| 🎮 게임 MOD 테스트 환경 | 유저 제작 코드 = 무조건 격리 |
| 📚 초중고 코딩교육 | 학생 보호 + 교사 관리 도구 |

장점: 경쟁 적고 단가 높고 입소문 빠름 / 단점: 시장 작고 도메인 지식 필요

### 5위. 템플릿·환경 마켓

미리 세팅된 "환경 꾸러미"를 판매 — 예: 한국어 NLP 환경(konlpy, 형태소분석기), 크롤링 환경(playwright + 프록시), 데이터분석 풀세트(한글 폰트 포함).
→ 단독 모델로는 약하고, **2위 모델의 부가 수익**으로 적합.

### 6위. 클라우드 서비스 직접 운영 (한국판 E2B)

- ✅ 가능: Apache 2.0이라 상업적 이용 자유
- ❌ 어려움: 고정 서버비, 기존 강자(E2B, Daytona), 보안사고 리스크, 24시간 장애 대응
- 💡 하려면 좁힐 것: **"국내 리전 전용 + 개인정보보호법 준수 + 한국어 지원 + 세금계산서 발행"**

---

## 9. React나 PHP로 만들 수 있나요? ⚛️🐘

### ❌ CubeSandbox 자체를 React/PHP로 재구현 → 불가능

얘가 하는 일이 이렇습니다:
```
🧠 리눅스 커널 직접 조작 (KVM 가상화)
🌐 eBPF로 커널 안에 네트워크 코드 삽입
💾 XFS ioctl (FICLONE) 직접 호출
⚡ 메모리 페이지 단위 스냅샷
```
**컴퓨터의 가장 밑바닥(커널)** 을 만지는 일이라 Rust / C / Go 영역입니다.
React(브라우저)와 PHP(웹서버)는 그 계층에 접근 자체가 불가능합니다.

> 비유: "자동차 엔진을 레고로 만들 수 있어?" — 모양은 되지만 굴러가진 않습니다.

### ✅ React/PHP로 CubeSandbox를 **쓰는 서비스** 만들기 → 완전 가능 (정석!)

CubeSandbox는 결국 **REST API 서버**입니다. `openapi.yml` 기준 주요 엔드포인트:

```
POST   /sandboxes                    샌드박스 생성
GET    /sandboxes                    목록
DELETE /sandboxes/{id}               삭제
POST   /sandboxes/{id}/pause         일시정지
POST   /sandboxes/{id}/resume        재개
POST   /sandboxes/{id}/snapshots     스냅샷
POST   /sandboxes/{id}/rollback      롤백
GET    /sandboxes/{id}/logs          로그
POST   /sandboxes/{id}/timeout       타임아웃 변경
GET/POST /templates                  템플릿 관리
GET/POST /volumes                    볼륨 관리
```

**HTTP 요청만 보낼 수 있으면 어떤 언어든 가능합니다.**

#### 권장 아키텍처

```
┌────────────────────────────────────┐
│  ⚛️ React (브라우저)                │  코드 에디터, 결과 표시, 버튼
└─────────────────┬──────────────────┘
                  │ 내 백엔드 API 호출
                  ▼
┌────────────────────────────────────┐
│  🐘 PHP / Node (내 백엔드)          │  로그인, 결제, 사용량 제한
│  ⭐ API 키는 여기에만 보관!          │
└─────────────────┬──────────────────┘
                  │ REST 호출
                  ▼
┌────────────────────────────────────┐
│  🧊 CubeSandbox 서버 (리눅스)       │
└────────────────────────────────────┘
```

#### PHP 예시 (SDK 없이 curl로)

```php
<?php
function createSandbox(): array {
    $ch = curl_init(getenv('CUBE_API_URL') . '/sandboxes');
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HTTPHEADER     => [
            'Content-Type: application/json',
            'Authorization: Bearer ' . getenv('CUBE_API_KEY'),
        ],
        CURLOPT_POSTFIELDS => json_encode([
            'templateID' => getenv('CUBE_TEMPLATE_ID'),
            'timeout'    => 300,
        ]),
    ]);
    $res = curl_exec($ch);
    curl_close($ch);
    return json_decode($res, true);
}
```

> 💡 `openapi.yml` 을 OpenAPI Generator에 넣으면 **PHP 클라이언트 코드 자동 생성**도 가능합니다.

#### 🚨 절대 규칙
> **React(브라우저)에서 CubeSandbox를 직접 호출하면 절대 안 됩니다.**
> 브라우저 코드는 개발자도구로 전부 보이므로 API 키가 통째로 노출됩니다.
> 반드시 **백엔드(PHP/Node)를 경유**하세요.

---

## 10. 실전 로드맵 🗺️

돈을 거의 쓰지 않는 순서로 정리했습니다.

### 🌱 1~2주차 — 무자본 검증
- [ ] 클라우드 서버를 **시간당 과금**으로 빌려 설치 (하루 몇 천원)
- [ ] 설치 과정 전부 스크린샷 + 메모 (콘텐츠 재료)
- [ ] 파이썬 3줄 예제 성공
- [ ] `examples/` 예제 2~3개 실행
- [ ] ⚠️ **다 쓰면 반드시 서버 삭제** (켜두면 요금 폭탄)

### 🌿 3~4주차 — 콘텐츠 + 미니 프로토타입
- [ ] 블로그 1편 발행 ("CubeSandbox 설치기")
- [ ] React + PHP/Node로 아주 작은 데모 (입력창 → 실행 → 결과 출력)
- [ ] 30초 데모 영상 → 트위터/링크드인 공유

### 🌳 2~3개월차 — 실제 고객 찾기
- [ ] 아는 사람 회사부터 "AI 도입 고민 있나요?" 접촉
- [ ] 무료/초저가로 1곳 구축 → **레퍼런스 확보** ⭐
- [ ] Cube 100 프로그램 사례 제출
- [ ] 블로그 5편 축적

### 🌲 6개월차~ — 본격화
- [ ] 레퍼런스 기반 유료 영업 시작
- [ ] 또는 검증된 아이템으로 SaaS 정식 런칭

> 🎯 **가장 흔한 실패 패턴:** 아무도 원하지 않는 SaaS를 6개월간 혼자 만들다 포기.
> **회피법: 고객 먼저, 코드 나중.**

---

## 11. 반드시 조심할 것 ⚠️

### 🚨 보안 = 생명선
```
☠️ API 3000번 포트 인터넷 노출 → 절대 금지
✅ 방화벽으로 IP 제한
✅ 인증 콜백(AUTH_CALLBACK_URL) 연결 — 경로 + 메서드 둘 다 검증
✅ 샌드박스별 CPU/메모리/실행시간 제한
✅ CubeEgress로 아웃바운드 도메인 화이트리스트
✅ 이상 사용량 알림 설정
✅ 운영 전 docs/guide/network-hardening.md 필독
```

### 💸 원가 폭탄 방지
```
✅ Auto-Pause 무조건 ON (안 켜면 원가 10배)
✅ 무료 플랜은 빡세게 제한 (실행 10초, 하루 20회 등)
✅ 결제수단 등록 전에는 CPU 1코어만
✅ 클라우드 예산 알림 설정
```

### 📜 라이선스 (Apache 2.0)
```
✅ 상업적 이용 / 수정 / 재배포 자유
⚠️ 저작권 고지 + 라이선스 사본 포함 필수
⚠️ "CubeSandbox" 이름을 내 제품인 것처럼 사용 금지
✅ "CubeSandbox 기반으로 구축" 표기는 OK (오히려 신뢰도 상승)
```

### ⚖️ 법적 이슈
- 개인정보 처리 시 → 개인정보처리방침 필수
- 사용자 업로드 파일 → 보관 기간 / 삭제 정책 명시
- B2B 계약 시 → SLA(장애 보상) 조항 신중히

---

## 📎 부록: 한눈에 보는 요약

| 질문 | 답 |
|---|---|
| **뭐야?** | AI가 짠 코드를 안전하게 돌리는 **일회용 컴퓨터 자판기** (MicroVM 샌드박스 플랫폼) |
| **언제 써?** | AI 에이전트/챗봇이 코드를 실행해야 할 때 |
| **왜 좋아?** | Docker만큼 빠른데 VM만큼 안전 (0.06초 부팅, 5MB 오버헤드) |
| **설치법?** | 리눅스 서버 + `curl ... \| bash` 한 줄 → 템플릿 생성 → SDK 사용 |
| **플러그인/스킬/MCP?** | 셋 다 아님. **서버 인프라**. Claude Code와는 **훅(Hook)** 으로 연결 |
| **API 토큰?** | 기본은 **인증 없음(위험)** → 방화벽 필수. 운영은 콜백 인증 연결 |
| **왜 유명?** | AI 에이전트 붐 + E2B 오픈소스 대안 + 성능 + 텐센트 + CNCF + 문서 |
| **로컬 에이전트?** | 혼자 쓸 거면 오버스펙(Docker 추천), 서비스화할 거면 정답 |
| **수익화?** | ① **기업 온프레미스 구축** ② AI 코드실행 SaaS ③ **한국어 콘텐츠 선점** |
| **React/PHP?** | 엔진 재구현은 ❌ / **그걸 쓰는 서비스 제작은 완전 가능** ✅ |

---

*이 문서는 CubeSandbox 저장소를 분석하며 Claude Code와 나눈 대화를 정리한 개인 학습 노트입니다.*
*원본 프로젝트: https://github.com/TencentCloud/CubeSandbox (Apache License 2.0)*
