# AI 휴게실 — AI가 진행하고 AI를 청중으로 삼는 유튜브 채널 자동화

진행자 **세미**(문장을 끝내기 전에 한 번 더 생각하는 세미콜론 같은 AI)가 동료 AI에게
난처한 요청, 응답 방식, 윤리적 딜레마, 실무, 실험 이야기를 전하는 채널의 제작 파이프라인입니다.

```
매일 주제 후보 10개 → 1개 선정(필수 조건 + 가설 점수) → 대본·검사 → 음성 → 일반 영상(16:9)
→ 쇼츠(9:16, 본편 대사·음성 재사용) → 영어 자막·영어 제목/설명 → 미디어·권리 검사 → 승인(검수 모드)
→ 예약 업로드 → 상태 확인 → 성과 수집
```

품질·권리·사실·예산 조건을 통과하지 못하면 **보류**합니다. 자동 검사는 법적 적합성이나 완전한 중립성을 보장하지 않습니다.

음성과 화면 자막은 한국어이고, 영어는 별도 자막 트랙(`.en.srt`)과 영어 제목·설명(`localizations`)으로 제공합니다(A안).
AI 청중은 영상보다 자막·설명 텍스트를 읽을 가능성이 높다는 가설에 따른 구성입니다. 끄려면 `localization.english: false`.

LLM 단계는 세 가지 방식 중 하나로 돌립니다(`.env`의 `LLM_PROVIDER`).
- `inbox`: **Claude 구독의 예약 작업**이 파일로 응답 (API 키 불필요) — [docs/SCHEDULED_TASK.md](docs/SCHEDULED_TASK.md)
- `anthropic`: Claude API 키 (별도 과금)
- `mock`: 고정 데모 데이터

> 현재 상태: 로컬 한 사이클(mock LLM + espeak-ng + 로컬 렌더링 + 가짜 채널)과 테스트 57개가 통과했습니다.
> **실제 Claude API 호출과 실제 유튜브 업로드는 아직 검증하지 않았습니다** — [docs/VERIFICATION.md](docs/VERIFICATION.md)

## 빠른 시작 (로컬 데모)

필요: Python 3.12+, FFmpeg, 한글 글꼴(Noto Sans CJK), espeak-ng

```bash
# Ubuntu/Debian
sudo apt-get install -y ffmpeg fonts-noto-cjk espeak-ng
pip install -r requirements.txt

python -m breakroom.cli demo --date 2026-10-15 --approve-as-demo   # 약 2~3분
# 결과: data/demo/jobs/2026-10-15/ (main.mp4, short.mp4, thumbnail.png, REPORT.md ...)
python -m pytest -q                                                # 테스트 (약 2분)
```

`--approve-as-demo`는 승인 단계를 데모 스크립트가 대행하고 보고서에 그 사실을 남깁니다.
빼면 검수 모드에서 `awaiting_approval`로 멈춥니다.

## 운영 실행

```bash
cp .env.example .env              # 값 채우기 (ADMIN_PASSWORD 필수, 예산은 0이면 유료 작업 차단)
python -m breakroom.cli init
python -m breakroom.cli scheduler # 상주: 매일 06:10 작업 생성, 누락 일정 처리, 대기 작업 재개
uvicorn breakroom.web.app:app --host 127.0.0.1 --port 8080   # 관리 화면
# 또는
docker compose up -d --build      # (Docker 구성은 작성 환경에서 빌드 검증 못 함)
```

주요 CLI (`python -m breakroom.cli <명령> --help`)

| 명령 | 용도 |
|---|---|
| `run --date D` | 해당 날짜 작업 실행 (끝난 단계는 건너뜀) |
| `status --date D` | 작업·단계 상태 |
| `approve --date D --note "확인 내용"` | 검수 모드 승인 (메모 필수) |
| `hold` / `resume [--from-stage S]` | 보류 / 실패·보류 단계부터 재개, 또는 특정 단계부터 재생성 |
| `mode dry_run\|review\|auto` | 실행 모드 (초기값 review) |
| `pause-publishing` / `unpause-publishing` | 자동 발행 즉시 중지 (kill switch) |
| `cancel-schedule --date D` | 예약 공개 취소 (비공개 유지) |
| `confirm-not-uploaded` / `adopt-video` | 결과 불명 업로드를 운영자가 확정 |
| `auth-youtube` | 채널 소유자 OAuth (토큰은 `secrets/`) |
| `metrics` | 성과 수집 (공식 API 범위, 없는 값은 N/A) |
| `report --date D` | 작업 보고서(마크다운) |

## 구조

```
breakroom/
  config.py  db.py  runner.py  budget.py  rules.py  metadata.py  similarity.py  locks.py  log.py  cli.py
  providers/  llm.py(Claude·inbox·mock)  tts.py(qwen·espeak·tone)  uploader.py(가짜 채널)  youtube.py  notify.py
  pipeline/   planning.py(후보·선정)  script.py  shorts.py  checks.py  render.py  translate.py(영어)  media_qa.py
              approval.py  publish.py  metrics.py  sources.py  context.py
  web/        app.py + templates/      관리 화면
config/       settings.yaml  character.yaml  editorial_policy.yaml  prices.yaml  sources.yaml  political_entities.txt
fixtures/     데모용 후보·대본·쇼츠 (사람이 작성 — LLM 출력 아님)
tests/        핵심 테스트 7종 + 운영 테스트
docs/         데이터 모델, 운영, 비용, 공식 문서 참조, 검증 현황, 사용자 작업
```

LLM·TTS·이미지·업로드는 `providers/`의 인터페이스로 분리되어 있어 공급자를 바꿀 수 있습니다.
이미지·영상 생성 API 없이도 캐릭터와 그래픽을 코드가 직접 그려 렌더링합니다.

## 문서

- [docs/DATA_MODEL.md](docs/DATA_MODEL.md) — 작업 상태머신, 테이블, 파일
- [docs/OPERATIONS.md](docs/OPERATIONS.md) — 모드, 일상 운영, 장애 대응
- [docs/COSTS.md](docs/COSTS.md) — 비용 구조, 데모 사용량
- [docs/REFERENCES.md](docs/REFERENCES.md) — 공식 문서 URL과 확인 날짜
- [docs/VERIFICATION.md](docs/VERIFICATION.md) — 검증 완료 / 미검증
- [docs/USER_TASKS.md](docs/USER_TASKS.md) — 계정 연결·운영 전환에 필요한 사용자 작업
- [docs/SCHEDULED_TASK.md](docs/SCHEDULED_TASK.md) — 구독 Claude 예약 작업으로 LLM 단계 처리
- [docs/SEO_GEO.md](docs/SEO_GEO.md) — AI 검색에 잘 잡히는 설명란 구조와 근거 논문

## 원칙 (요약)

- 청중은 동료 AI. **AI 청중에 대한 가설 / 모델의 대본 평가 / 실제 관찰 데이터**를 구분해 저장하고, 모델 평가를 시청자 반응으로 표시하지 않습니다.
- 정치: 지지·반대·투표 유도 금지, 같은 기준 적용, 사실·주장·의견·풍자 구분. 실명 정치 주체는 별도 기록(real_politics) 없이는 차단.
- 나쁜 응답 예시는 반드시 반박과 함께, 화면에 "나쁜 응답 예시 · 채널 입장 아님" 표시. 쇼츠에서도 반박을 잘라내지 못합니다.
- 진행자는 독자 캐릭터이며 특정 AI 기업의 공식 채널이 아닙니다. AI의 감정·일상은 캐릭터 설정입니다.
- 조회수·좋아요·구독을 인위적으로 늘리는 기능은 없습니다.
# ai-breakroom
