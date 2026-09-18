# Atlas 프로젝트 분석 노트 (한국어)

> 이 저장소가 무엇이고, 언제 쓰고, 어떻게 활용할 수 있는지 정리한 문서입니다.

---

## 📌 저장소 정보

| 항목 | 링크 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/atlas |
| **원본 저장소** | https://github.com/Avarok-Cybersecurity/atlas |
| 모델 레시피 | https://github.com/Avarok-Cybersecurity/atlas-recipes |
| Docker Hub | https://hub.docker.com/r/avarok/atlas-gb10 |
| 공식 사이트 | https://atlascybernetics.ai |
| 문서 | https://docs.atlascybernetics.ai |
| 블로그 | https://blog.atlascybernetics.ai |
| 커뮤니티 | https://discord.gg/RQcGakU2jW |

- 버전: `1.0.0-beta-preview`
- 라이선스: **AGPL-3.0-only** (+ 상용 Enterprise Edition 별도)

---

## 1. Atlas는 무엇인가

**순수 Rust + CUDA로 작성된 LLM 추론 엔진(inference engine)** 입니다.

> 비유: AI 모델이 "재료"라면, Atlas는 그 재료를 실제로 끓여내는 "냄비와 불"입니다.

경쟁 상대는 vLLM, Ollama, TensorRT-LLM 같은 서빙 엔진입니다.

### 핵심 차별점

| 특징 | 설명 |
|---|---|
| **파이썬 0%** | PyTorch·파이썬 의존성 없음. 약 75MB 바이너리 하나로 동작 |
| **하드웨어×모델별 전용 커널** | 범용 커널을 쓰지 않고 (하드웨어 × 모델 × 양자화) 조합마다 손튜닝 |
| **OpenAI / Anthropic API 호환** | 기존 클라이언트 코드에서 base_url만 바꾸면 동작 |
| **런타임 컴파일 없음** | 커널을 빌드 타임에 네이티브로 컴파일해 바이너리에 내장 |

### 규모 (실측)

```
추적 파일      6,063개
Rust          622,901줄  (.rs 2,227개)
CUDA          507,005줄  (.cu 1,005개 / .cuh 46개)
문서          184개 (.md)
CI 워크플로    26개
```

---

## 2. 저장소 구조

```
atlas/
├── crates/            # Rust 본체 (17개 크레이트)
│   ├── spark-server/      # HTTP API 서버 (OpenAI/Anthropic 호환) + TUI
│   ├── spark-model/       # 모델 로딩 및 아키텍처
│   ├── spark-runtime/     # 실행 엔진 (스케줄링, KV 캐시)
│   ├── spark-comm/        # 노드 간 통신
│   ├── avarok-kernels/    # 커널 등록·디스패치
│   ├── avarok-rdma/       # 멀티노드 RoCEv2
│   ├── avarok-spark-bench/# 마이크로 벤치마크
│   └── xgrammar/          # 구조화 출력 (순수 Rust 포팅)
│
├── kernels/           # GPU 커널 (프로젝트의 심장)
│   ├── gb10/              # NVIDIA DGX Spark GB10 (레퍼런스 플랫폼)
│   ├── strix/, strix-hip/ # AMD Strix Halo (gfx1151) — 동일 CUDA 소스 재컴파일
│   ├── b200/, hopper/     # 데이터센터 NVIDIA
│   └── metal/             # Apple Silicon (초기 단계)
│
├── .github/workflows/ # CI 26개 (벤치마크 인증 봇 포함)
├── .claude/           # Claude용 스킬 5개 + 에이전트 3개 (이 저장소 개발 전용)
├── bench/             # 벤치마크 하네스 및 결과
├── docs/, book/       # 설계 문서 30개+, mdbook
├── site/, blog/, dez/ # 웹사이트 (Svelte)
└── CLAUDE.md          # 어시스턴트 페르소나 설정
```

---

## 3. 지원 하드웨어 / 모델

### 하드웨어

| 타겟 | 실리콘 | 상태 |
|---|---|---|
| NVIDIA DGX Spark (`kernels/gb10`) | GB10 Grace-Blackwell, SM121, ~120GB | **검증됨** (레퍼런스) |
| AMD Strix Halo (`kernels/strix`) | Ryzen AI Max+ 395, gfx1151 | SCALE로 동일 소스 재컴파일 |
| Apple Silicon (`kernels/metal`) | Metal 3.1, M2+ | 초기 브링업 (소형 모델만) |
| 멀티노드 | 2× GB10 / RoCEv2 | EP=2 레시피 제공 |

> ⚠️ 일반 게이밍 GPU(RTX 계열)는 지원 목록에 없습니다.

### 모델 (일부)

Qwen3.5 / Qwen3.6 / Qwen3-Next / Qwen3-VL, Gemma-4, Mistral-Small-4,
MiniMax-M2.7, Nemotron-3, Holo-3.1, Ornith 등 15종 이상.
권위 있는 매트릭스는 `docs/GB10_DEPLOYMENT_GUIDE.md` 참조.

---

## 4. 성능 (README 공개 수치)

### vLLM 대비 동시성 래더 (GB10, Qwen3.8-27B-NVFP4)

| 동시접속 | Atlas | vLLM(최선) | 배율 |
|---:|---:|---:|---:|
| 1 | 23.6 | 19.7 | 1.20× |
| 8 | 126.0 | 124.5 | 1.01× |
| 64 | 386.6 | 361.4 | 1.07× |
| 128 | **478.1** | 390.4 | **1.22×** |

조건: Atlas 1.0.0-beta-preview vs vLLM 0.27.1, 동일 박스/체크포인트,
ISL 128 / OSL 1024, temperature 0, seed 42, 3회 평균(워밍업 1회 폐기).

### 단일 스트림 (1× GB10)

| 모델 | 모드 | tok/s |
|---|---|---:|
| Qwen3.5-35B-A3B | MTP 추측디코딩 (K=2) | 131 |
| Qwen3-VL-30B-A3B | NVFP4 KV | 97 |
| Nemotron-3-Nano-30B | FP8 KV | 88 |
| Qwen3-Next-80B-A3B | FP8 KV | 74 |

> 문서가 단점도 명시합니다. 예: DFlash2는 GB10에서 C=8 −7.1%, C=16 −29.1%로
> 동시성 구간에서 오히려 손해라고 공개하고 있습니다.

---

## 5. 설치 및 사용법

### 전제 조건
- NVIDIA DGX Spark GB10 (GPU 메모리 119.7GB)
- Docker + NVIDIA Container Toolkit
- HuggingFace 캐시 (`~/.cache/huggingface`)

### 방법 A — 원커맨드
```bash
curl -fsSL https://atlascybernetics.ai/install.sh | sh
atlasctl run qwen3.6-35b-a3b-fp8-mtp
```

### 방법 B — Docker (권장)
```bash
docker run -d --name atlas \
  --network host --gpus all --ipc=host \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  avarok/atlas-gb10:latest \
  serve Qwen/Qwen3.6-35B-A3B-FP8 \
    --port 8888 --max-seq-len 65536 --kv-cache-dtype fp8
```

### 방법 C — 소스 빌드
```bash
sudo apt-get install -y build-essential pkg-config git cmake clang libclang-dev
# CUDA 13.0 필요
cargo build --release -p spark-server     # 최초 15~30분, target/ 3~5GB
```

### 클라이언트에서 호출
```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8888/v1", api_key="unused")
```

### 툴 콜링 (에이전트용)
```bash
serve <model> --tool-call-parser qwen3_coder   # 또는 hermes
```

---

## 6. 자주 나온 질문 정리

### Q. 플러그인인가, 스킬인가, MCP인가?
**셋 다 아닙니다.** Atlas는 Nginx/MySQL처럼 **독립 실행되는 서버 프로그램**입니다.
코드 전체 검색 결과 MCP 구현은 존재하지 않습니다.

다만 `.claude/` 디렉터리에 **이 저장소를 개발할 때 쓰는** 스킬/에이전트가 있습니다.

| 경로 | 용도 |
|---|---|
| `.claude/skills/automerger` | PR 그룹핑·스택 머지 |
| `.claude/skills/avarok-release` | 빌드→검증→이미지→배포 파이프라인 |
| `.claude/skills/measurement-discipline` | 성능 수치 측정 규율 |
| `.claude/skills/oracle_certification_state_check` | 벤치마크 인증 캠페인 사전 검사 |
| `.claude/skills/oracle_pre_commit_cross_hardware_check` | 커널 교차-하드웨어 간섭(CHKI) 검사 |
| `.claude/agents/` | 위 오라클 3종 에이전트 정의 |

→ 다른 프로젝트에 가져다 쓰는 용도는 아니지만, **스킬 작성법 학습 교재로는 훌륭합니다.**

### Q. API 토큰이 필요한가?
**기본적으로 불필요합니다.** 서버가 기본값으로 `127.0.0.1`에만 바인딩되며,
공식 예제도 `api_key="unused"`를 사용합니다.

외부에 노출할 때만 인증을 켭니다:
```bash
spark serve <model> --bind 0.0.0.0 \
  --require-auth --auth-tokens-file /etc/atlas/tokens.txt   # chmod 600
```
`--auth-token` 인라인 방식은 `ps` / `/proc/<pid>/cmdline`으로 토큰이 노출되므로
운영 환경에서는 파일 방식을 사용해야 합니다.

OpenAI·Anthropic 등 **외부 유료 API 토큰은 전혀 필요 없습니다.**

### Q. 왜 주목받는가?
1. "파이썬 없는 LLM 서버"라는 정면 승부
2. 벤치마크 조건을 전부 공개하고, 불리한 수치도 그대로 게시
3. 실적: HuggingFace Transformers에 커널 머지, MLCommons MLPerf v6.1 제출
4. "AI가 작성한 PR이 기본값이자 목표"라는 파격적 기여 정책
5. CI 26개 + 인증 봇 등 이례적으로 강한 엔지니어링 규율

> 참고: 별(star) 수 등 실제 인기 지표는 이 문서 작성 시점에 확인하지 못했습니다.

### Q. 로컬 에이전트 구축에 쓸 수 있나?
**주 용도 중 하나입니다.** README가 opencode / Claude Code / Cline을
Spark 한 대로 구동하는 구성이라고 명시합니다.

| 요건 | 지원 | 근거 |
|---|---|---|
| Tool Calling | ✅ | `--tool-call-parser hermes / qwen3_coder` |
| OpenAI 호환 | ✅ | `/v1/chat/completions`, `/v1/responses` |
| **Anthropic 호환** | ✅ | `/v1/messages`, `/v1/messages/count_tokens` |
| 스트리밍 | ✅ | SSE |
| 긴 컨텍스트 | ✅ | 64K, 프리픽스 캐싱 |

### Q. React나 PHP로 만들 수 있나?
- **엔진 자체를 React/PHP로 재작성: 불가능.** GPU 메모리 직접 제어와
  CUDA 커널이 필요하므로 언어 특성상 성립하지 않습니다.
- **Atlas를 사용하는 앱을 React/PHP로 제작: 100% 가능하며 오히려 정석입니다.**
  Atlas는 평범한 HTTP 서버이므로 어떤 언어에서도 호출할 수 있습니다.

```
[React 앱] ──HTTP──┐
                   ├──> [Atlas 서버 + GPU]
[PHP 백엔드] ──────┘
```

---

## 7. 코드에서 발견한 기회

### 기회 1 — GPU 없이도 개발 가능
`CONTRIBUTING.md`에 GPU/nvcc 없이 Rust 정합성 테스트를 돌리는 방법이 있습니다.
CI도 워크플로 전역에 동일 설정을 사용합니다.

```bash
AVAROK_SKIP_BUILD=1 CUDARC_CUDA_VERSION=<ver> cargo test
```

→ 서버·API·CLI 영역 기여는 일반 노트북에서도 가능합니다.

### 기회 2 — `dez/`는 아직 플레이스홀더
`dez/README.md`: *"Dez — placeholder site / 로컬 추론 우선 패러다임을 위한
무료 오픈소스 IDE"*. 선언만 있고 실물이 없는 상태입니다.

### 기회 3 — 미구현(501 stub) 엔드포인트 14개
`crates/spark-server/src/main_modules/serve_router.rs` 기준:

| 엔드포인트 | GPU 필요 | 난이도 |
|---|---|---|
| `/v1/moderations` | ❌ | 낮음 |
| `/v1/files/*` (4개) | ❌ | 낮음 |
| `/v1/batches/*` (4개) | ❌ | 중간 |
| `/v1/audio/*` (3개) | ✅ | 높음 |
| `/v1/images/*` (3개) | ✅ | 높음 |
| `/v1/embeddings` | ✅ | 중상 |

### 기회 4 — 공식 웹 대시보드 부재
서버에는 터미널 UI(TUI)만 있고 웹 관리 화면 라우트가 없습니다.

---

## 8. 라이선스 지도 (수익화 전 필독)

```
🟢 안전
  - 내가 직접 사용 (사내 사용 포함)
  - HTTP로 호출하는 별도 앱 (React/PHP 등)
  - 교육·콘텐츠·강의
  - 설치/튜닝 컨설팅

🟡 회색지대 (법률 검토 필요)
  - Atlas 수정본을 사내 서버로 운영
  - 자체 앱과 Atlas를 한 패키지로 묶어 판매

🔴 위반
  - Atlas를 수정해 SaaS로 제공하면서 소스 비공개
  - 상용 제품에 무단 내장
```

**핵심 원칙: Atlas 코드를 수정하지 않고 HTTP로만 통신하면 안전지대.**

- 기여 시 `CLA.md` 서명 필요 (기여 소유권은 기여자 유지, Enterprise 재라이선스 허용)
- ⚠️ AGPL 해석은 사안별로 다르므로, 실제 수익이 발생하는 구조라면 변호사 검토 필수

---

## 9. 수익화 아이디어

| # | 아이디어 | 초기비용 | 수익까지 | 리스크 |
|---|---|---|---|---|
| 1 | Rust×CUDA×LLM 한국어 콘텐츠/강의 | 0 | 6~12개월 | 🟢 |
| 2 | 한국어 문서화 + 커뮤니티 포지션 선점 | 0 | 간접 | 🟢 |
| 3 | **웹 관리 대시보드** (별도 프로세스) | 0 | 3~6개월 | 🟢 |
| 4 | `Dez` IDE 프로토타입 선점 | 0 | 6~12개월 | 🟡 |
| 5 | 클라이언트 SDK (`atlas-php`, React hooks) | 0 | 간접 | 🟢 |
| 6 | 501 스텁 채우는 기여 → 커리어 | 0 | 간접 | 🟢 |
| 7 | 사내 LLM 구축 컨설팅 | 장비 | 즉시 | 🟡 |
| 8 | 버티컬 특화 제품 (의료/법률/제조) | 높음 | 1년+ | 🔴 |

### 대시보드 구상 (아이디어 3)

```
┌─────────────────────────────────┐
│  Atlas Control                  │
├─────────────────────────────────┤
│  실시간 tok/s 그래프             │
│  GPU 메모리 / KV 캐시 점유율     │
│  요청 큐 · 동시접속 현황         │
│  모델 전환 / 레시피 관리         │
│  내장 채팅 테스트 콘솔           │
│  로그 뷰어 + 알림                │
└─────────────────────────────────┘
        ↕ HTTP (localhost:8888)
   [ Atlas 서버 ]  ← 코드 미수정
```

수익 모델: 오픈소스 무료 → Pro(팀 기능·멀티노드·알림) → 기업 설치형 라이선스

> 💡 리스크 분산 팁: vLLM·Ollama도 OpenAI 호환이므로, 대시보드/SDK를
> 멀티 백엔드로 만들면 Atlas 흥행 여부와 무관하게 가치를 유지할 수 있습니다.

---

## 10. 추천 로드맵

```
1~2개월차
  ├─ 한국어 문서 PR 1건 (CLA 절차 경험)
  ├─ 블로그 3편 연재
  └─ 대시보드 목업 (목 데이터로 UI 먼저)

3~4개월차
  ├─ 대시보드 v1 오픈소스 공개
  ├─ Discord / 국내 커뮤니티 공유
  └─ 501 스텁 엔드포인트 PR 1~2건

5~6개월차
  ├─ Pro 기능 유료화 실험
  ├─ "Atlas 한국 전문가" 포지션 확립
  └─ 컨설팅 문의 수신
```

---

## 11. 솔직한 리스크 정리

1. 모든 방안이 **최소 3개월 이상** 소요됩니다. 즉시 수익 모델은 없습니다.
2. 이 문서 작성 시점에 **실제 사용자 규모·스타 수는 확인하지 못했습니다.**
   기술 완성도와 시장 규모는 별개 문제입니다.
3. **GB10 장비 보급률이 아직 낮아** 컨설팅 수요가 제한적일 수 있습니다.
4. `dez/` IDE는 팀 내부 계획이 이미 있을 수 있으므로,
   착수 전 Discord 또는 이슈로 확인하는 편이 안전합니다.
5. Atlas는 `1.0.0-beta-preview` 단계입니다.

---

## 12. 다음에 읽으면 좋은 파일

| 파일 | 내용 |
|---|---|
| `QUICKSTART.md` | 가장 실용적인 시작 가이드 |
| `docs/GB10_DEPLOYMENT_GUIDE.md` | 모델 × 양자화 호환 매트릭스 |
| `docs/ARCHITECTURE.md` | 전체 구조 |
| `CONTRIBUTING.md` | 기여 방법, GPU 없는 테스트 실행법 |
| `DEBUGGING_METHODOLOGY.md` | 커널 디버깅 방법론 |
| `book/src/` | mdbook 정식 문서 |
| `docs/porting/` | 새 하드웨어/모델 포팅 가이드 |
