# HTTPX2 저장소 분석 및 활용 정리 (한국어)

이 문서는 `httpx2` 저장소를 전수조사하여 **무엇을 하는 프로젝트인지, 언제 쓰는지,
어떻게 활용·수익화할 수 있는지**를 한국어로 정리한 기록입니다.

## 저장소 주소

| 구분 | 주소 |
| --- | --- |
| 이 저장소 (포크) | https://github.com/bmshin94/httpx2 |
| 업스트림 (원본) | https://github.com/pydantic/httpx2 |
| 원조 프로젝트 | https://github.com/encode/httpx |
| 공식 문서 | https://pydantic.dev/docs/httpx2/ |
| PyPI | https://pypi.org/project/httpx2/ |
| 라이선스 | BSD-3-Clause (`LICENSE.md`) |

분석 시점 버전: **2.13.0** (2026-09-14 릴리스), 기준 커밋 `66d76cb`.

---

## 1. 한 줄 요약

`httpx2`는 **파이썬용 차세대 HTTP 클라이언트 라이브러리**입니다.
널리 쓰이던 `httpx 0.28.1`을 **Pydantic 팀이 포크하여 이어받은 공식 후속 프로젝트**이며,
이 저장소는 그것을 `bmshin94` 계정으로 다시 포크한 사본입니다.

## 2. 왜 포크되었나

- 원작자(`@lovelydinosaur`, Tom Christie)의 `httpx`는 최근 유지보수 활동이 크게 줄었습니다.
- 그런데 `httpx`는 FastAPI, Starlette, OpenAI SDK, MCP SDK 등
  **파이썬 생태계의 핵심 경로**에 깔려 있어 보안 패치 공백이 치명적입니다.
- Pydantic(Pydantic Services Inc.)이 `httpx2` 이름으로 유지보수를 인수했습니다.
- 즉 `httpx2` = **살아있는 httpx**. 공개 API는 사실상 동일합니다.

## 3. 폴더 전수조사 결과

| 경로 | 내용 |
| --- | --- |
| `src/httpx2/httpx2/` | 상위 레벨 클라이언트. `_client.py`(Client/AsyncClient), `_models.py`, `_urls.py`+`_urlparse.py`(WHATWG URL 파서), `_auth.py`, `_content.py`, `_multipart.py`, `_decoders.py`(gzip/brotli/zstd), `_config.py`(Timeout/Limits/Proxy/SSL), `_status_codes.py`, `_main.py`(CLI), `_sse.py`(SSE 내장), `websockets/`(WebSocket 내장), `_transports/`(default/asgi/wsgi/mock), `_alias.py`(마이그레이션 브릿지) |
| `src/httpcore2/` | 하위 레벨 전송 엔진(별도 패키지). `_async/`가 실제 구현(connection, connection_pool, http11, http2, http_proxy, socks_proxy), `_sync/`는 스크립트로 자동 생성된 동기 버전, `_backends/`(anyio/trio/sync/mock) |
| `tests/` | `httpx2`/`httpcore2` 각각 async·sync 테스트. `whatwg.json`으로 URL 표준 적합성 검증. 커버리지 100% 미달이면 CI 실패 |
| `docs/` | 35개 문서. `quickstart`, `async`, `http2`, `sse`, `websockets`, `migration`, `compatibility`, `troubleshooting`, `advanced/`(clients·proxies·ssl·timeouts·authentication·event-hooks·transports·resource-limits·emscripten) |
| `benchmark/` | 자체 `asyncio.Protocol` 서버 기반 처리량 벤치마크. `aiohttp` 비교, `us/req`·`retention` 지표, pyinstrument 프로파일링 |
| `scripts/` | `install`·`check`·`test`·`coverage`·`lint`·`build`·`docs`·`benchmark`, 그리고 async 코드를 sync 코드로 자동 변환하는 `unasync.py` |
| `.github/` | Python 3.10~3.15 매트릭스 CI, zizmor(워크플로 보안 스캔), dependabot, PyPI 배포 워크플로 |
| `redirects/`, `wrangler.toml` | Cloudflare Workers로 구 문서 URL 리다이렉트 |
| `AGENTS.md`, `CLAUDE.md` | 이 포크에서 추가된 AI 협업 지침 및 페르소나 설정 (PR #1 `feat/claude-guide`) |

소스 규모: `src` 기준 약 18,555줄.

## 4. 핵심 기능

### httpx에서 그대로 이어받은 것

- `requests` 호환 API (`get`/`post`/`put`/`patch`/`delete`/`head`/`options`)
- 동기 + 비동기 양쪽 (`Client` / `AsyncClient`)
- HTTP/1.1 + HTTP/2
- 연결 풀링, 쿠키 세션, 리다이렉트, 프록시(HTTP/SOCKS), 전 구간 타임아웃
- ASGI/WSGI 앱에 네트워크 없이 직접 요청 (FastAPI 테스트의 근간)
- 전면 타입 어노테이션(`py.typed`), mypy strict, 커버리지 100%

### httpx2에서 새로 추가된 것

- **SSE 내장**: `client.sse()` → `httpx-sse` 불필요
- **WebSocket 내장**: `client.websocket()` (`httpx2[ws]`) → `httpx-ws` 불필요
- **`QUERY` 메서드** 지원
- **`truststore`로 OS 신뢰 저장소 사용** (certifi 대신) → 사내 프록시/인증서 문제 완화
- **`Origin` 값 객체** 및 `URL.origin`
- **Pyodide/WebAssembly 지원** (`httpx2-jsfetch`)
- **`alias_httpx()`**: `import httpx`를 프로세스 전체에서 `httpx2`로 치환
- Python 3.10+ 필수, `HTTPXDeprecationWarning`이 기본 노출

## 5. 어떨 때 쓰는가

1. 외부 API 호출 (결제, 소셜 로그인, LLM API 등)
2. 비동기 대량 요청 (수백~수천 URL 동시 수집)
3. FastAPI/Starlette 테스트 (`ASGITransport`)
4. LLM 스트리밍 응답 수신 (SSE)
5. 실시간 양방향 통신 (WebSocket)
6. 마이크로서비스 간 통신 (HTTP/2 멀티플렉싱)
7. CLI에서 API 테스트 (`httpx2` 명령, curl 대체)

## 6. 설치 및 사용법

### 설치

```shell
pip install httpx2
pip install 'httpx2[http2]'
pip install 'httpx2[cli]'
pip install 'httpx2[ws]'
pip install 'httpx2[brotli,cli,http2,socks,ws,zstd]'
```

### 기본 사용

```python
import httpx2

r = httpx2.get("https://api.github.com/users/bmshin94")
r.raise_for_status()
print(r.status_code, r.json())

# 실무에서는 Client 재사용 (커넥션 재사용 = 성능 향상)
with httpx2.Client(
    base_url="https://api.github.com",
    headers={"Authorization": "Bearer TOKEN"},
    timeout=httpx2.Timeout(10.0, connect=5.0),
    limits=httpx2.Limits(max_connections=100, max_keepalive_connections=20),
    follow_redirects=True,
    http2=True,
) as client:
    r = client.get("/users/bmshin94")
```

### 비동기

```python
import asyncio, httpx2

async def main():
    async with httpx2.AsyncClient() as client:
        results = await asyncio.gather(*(client.get(f"/item/{i}") for i in range(100)))
```

### SSE (LLM 스트리밍)

```python
async with httpx2.AsyncClient() as client:
    async with client.sse(url, method="POST", json={"stream": True}) as source:
        async for event in source:
            print(event.data, end="", flush=True)
```

### WebSocket

```python
with httpx2.Client() as client:
    with client.websocket("wss://example.com/ws") as ws:
        ws.send_text("hello")
        print(ws.receive_text())
```

### CLI

```shell
httpx2 https://api.github.com
httpx2 -m POST https://httpbin.org/post --json '{"a":1}'
httpx2 -h "Authorization" "Bearer TOKEN" --http2 -v https://example.com
httpx2 --download out.zip https://example.com/big.zip
```

### 이 저장소를 직접 개발할 때

```shell
scripts/install     # uv sync --frozen
scripts/check       # ruff format + mypy strict + ruff check + unasync 검증
scripts/test        # pytest + coverage
scripts/coverage    # 커버리지 100% 미달 시 실패
scripts/lint        # 자동 포맷 + 수정 + unasync 생성
scripts/docs        # 문서 로컬 서버
scripts/build       # 패키지 + 문서 빌드
scripts/benchmark --quick
```

주의: `src/httpcore2/httpcore2/_sync/`는 직접 수정하지 않습니다.
`_async/`를 수정한 뒤 `scripts/lint`를 실행하면 `scripts/unasync.py`가 동기 버전을 생성합니다.
CI는 `--check` 모드로 불일치를 검사합니다.

## 7. 플러그인? 스킬? MCP?

**셋 다 아닙니다. 순수한 파이썬 라이브러리(PyPI 패키지)입니다.**

| 구분 | 정체 | 실행 주체 |
| --- | --- | --- |
| 라이브러리 | 코드에서 `import`해 쓰는 부품 | 내 파이썬 프로그램 |
| 플러그인 | Claude Code 등에 기능을 끼우는 확장 꾸러미 | Claude Code CLI |
| 스킬 | AI에게 주는 작업 설명서(마크다운) | AI 모델 |
| MCP | AI가 외부 시스템에 접근하는 표준 프로토콜/서버 | 별도 서버 프로세스 |

다만 연결 고리가 있습니다. `docs/migration.md`에 따르면
**MCP Python SDK가 v2에서 `httpx` → `httpx2`로 전환**했습니다.
즉 `httpx2`는 MCP가 아니지만 **MCP 서버의 핵심 부품**입니다.

## 8. API 토큰이 필요한가

`httpx2` 자체는 **토큰이 전혀 필요 없습니다.** 가입·로그인도 없습니다.
토큰은 **호출 대상 API**가 요구할 때 쓰며, `httpx2`는 그것을 실어 보내는 역할입니다.

```python
import os, httpx2

client = httpx2.Client(headers={"Authorization": f"Bearer {os.environ['API_KEY']}"})

httpx2.get(url, auth=httpx2.BasicAuth("user", "pass"))
httpx2.get(url, auth=httpx2.DigestAuth("user", "pass"))
httpx2.get(url, auth=httpx2.NetRCAuth())
```

토큰 자동 갱신은 `httpx2.Auth`를 상속해 `auth_flow()`에서 401 시 재시도로 구현합니다.

보안 주의사항:

- 토큰 하드코딩 금지. 환경변수 / `.env` 사용 (`.env`는 `.gitignore`에 추가)
- CLI `--verbose`는 Authorization 헤더를 화면에 노출함
- 리다이렉트로 다른 도메인에 갈 때 `httpx2`가 인증 헤더를 자동 제거함 (토큰 유출 방지)

## 9. AI 에이전트 구축에 도움이 되는가 — 결론: 매우 그렇다

1. **LLM 스트리밍 = SSE 내장** (OpenAI/Anthropic/Gemini 전부 SSE 사용)
2. **멀티 에이전트 병렬 호출** (`AsyncClient` 하나를 공유해 커넥션 풀 공유)
3. **툴 호출(Tool Use) 실행 레이어** — 모든 툴은 결국 HTTP 요청
4. **전 구간 타임아웃** — 에이전트 멈춤 방지 (`connect` 짧게, `read` 길게)
5. **Realtime/음성 에이전트 = WebSocket**
6. **비용/토큰 추적 = `event_hooks`** (응답 훅에서 사용량 로깅)
7. **에이전트 테스트 = `MockTransport`** (비용 0, 결정적 테스트)
8. **HTTP/2 멀티플렉싱**으로 동일 호스트 다중 요청 레이턴시 감소
9. **MCP Python SDK가 이미 `httpx2` 기반**

## 10. React나 PHP로 만들 수 있는가

### 포팅은 비현실적

- 소스 18,555줄 + 테스트 포함 3만 줄 이상
- HTTP/2(`h2`), SOCKS, TLS 트러스트 스토어, WHATWG URL 파서 전부 재구현 필요
- `anyio`/`trio`/`asyncio` 동시성 추상화에 대응 개념이 없음
- React는 UI 라이브러리이며, 브라우저 샌드박스에서는 커넥션 풀·HTTP/2 설정·
  인증서 제어·프록시 설정이 불가능

### 각 언어의 대응품은 이미 존재

| 기능 | Python (`httpx2`) | JS/TS | PHP |
| --- | --- | --- | --- |
| 기본 요청 | `httpx2.get()` | `fetch`, `axios` | `Guzzle`, Symfony HttpClient |
| 비동기/병렬 | `asyncio.gather` | `Promise.all` | Guzzle Promises, Amp, Swoole |
| SSE | `client.sse()` | `EventSource` | Guzzle 스트림 + 직접 파싱 |
| WebSocket | `client.websocket()` | `WebSocket` | Ratchet, ReactPHP, Swoole |
| HTTP/2 | `http2=True` | 런타임 자동 | Guzzle + cURL(nghttp2) |
| 테스트 모킹 | `MockTransport` | `msw`, `nock` | Guzzle MockHandler |

### 권장 조합

```
[React 프론트엔드]   화면 / 스트리밍 UI
        ↕  fetch + EventSource(SSE)
[FastAPI 백엔드]     비즈니스 로직
        ↕  httpx2 (비동기 + SSE + 커넥션 풀)
[외부 API / LLM]
```

이유: API 키를 프론트에 노출하지 않음, 캐싱·재시도·레이트리밋·비용추적을 백엔드에서 중앙 관리.
PHP를 쓴다면 PHP를 API 게이트웨이로 두고 LLM·크롤링 워커만 Python + `httpx2`로 분리하는 하이브리드.

## 11. 유튜브 강의 제작 가능성 — 가능

- 라이선스: BSD-3-Clause이므로 코드 인용·설명·상업적 사용 모두 가능.
  설명란에 원 저작권/저장소 링크 표기, 로고·상표 직접 사용은 자제, 비공식 강의 명시 권장.
- 기회 요인: 한국어 자료가 거의 없음(선점 가능), `httpx` 유지보수 중단으로 마이그레이션 검색 수요 발생,
  AI/LLM 붐과 직결.

### 추천 커리큘럼 (10부작)

| 회차 | 제목 |
| --- | --- |
| 0 | `requests`는 이제 끝났나? 2026 파이썬 HTTP 클라이언트 지도 |
| 1 | `httpx2` 5분 입문 — requests 쓰던 사람 바로 적응 |
| 2 | 비동기 입문: API 100개 호출 10초 → 0.3초 실측 |
| 3 | Client 재사용 / 커넥션 풀 / Limits 실무 세팅 |
| 4 | 타임아웃 4종 완전정복 — 장애를 막는 설정 |
| 5 | SSE로 ChatGPT처럼 글자 흘리기 (실습) |
| 6 | WebSocket 실시간 채팅 만들기 |
| 7 | HTTP/2 + 프록시 + 트러스트 스토어 (사내망 인증서 문제 해결) |
| 8 | MockTransport / ASGITransport로 테스트 초고속화 |
| 9 | `httpx` → `httpx2` 마이그레이션 실전 + `alias_httpx()` |
| 10 | 오픈소스 설계 해부: 커버리지 100%, mypy strict, unasync |

심화: "httpx2 + FastAPI + React로 AI 챗봇 만들기" 3~5부작, 비동기 크롤러, LLM 비용 추적.
제작 팁: 저장소의 `benchmark/` 하네스 결과를 화면에 노출해 신뢰도 확보, 썸네일에 숫자 강조,
예제 저장소 별도 공개, 영어 자막으로 해외 유입.

## 12. 수익화 아이디어

핵심 전제: `httpx2`는 BSD 오픈소스이므로 **라이브러리 판매로는 수익이 나지 않습니다.**
수익은 **지식 / 시간 절약 / 운영 부담 대행**에서 발생합니다.

### 1) 교육 콘텐츠 — 난이도 낮음, 회수 빠름

| 채널 | 상품 | 가격대 |
| --- | --- | --- |
| 유튜브 | 무료 강의 + 애드센스/멤버십 | CPM 기반 |
| 인프런/클래스101 | 유료 강의 | 5만~15만원 |
| Udemy | 영어 강의 | $20~100 |
| Gumroad/리디 | 전자책(마이그레이션 실무 가이드) | 1.5만~3만원 |
| 뉴스레터 | 유료 구독 | 월 5천~1만원 |
| 기업 | 사내 교육 | 회당 100만~500만원 |

### 2) 마이그레이션 자동화 도구 + 컨설팅

무료 OSS CLI(`httpx2-migrate`)로 리드를 만들고 유료 서비스로 전환:

```shell
httpx2-migrate scan ./src        # httpx 사용처 전수 스캔 + 리포트
httpx2-migrate apply ./src       # import 자동 변환 + diff
httpx2-migrate check-deps        # 전이 의존성 중 httpx 사용 패키지 탐지
```

| 상품 | 가격 |
| --- | --- |
| 마이그레이션 진단 리포트 | 50만~150만원 |
| 실행 대행 (소규모) | 300만~800만원 |
| 실행 대행 (대규모) | 1,500만~5,000만원 |
| LTS 유지보수 계약 | 월 100만~500만원 |

판매 논리: `docs/migration.md`가 경고하는 "객체가 패키지 경계를 넘지 못하는 문제"
(`isinstance(client, httpx.Client)` 실패, `except` 미포착)는 모르고 진행하면 프로덕션 장애로 직결됩니다.

### 3) 오픈코어 확장 패키지

무료 OSS로 생태계 선점:

| 패키지 | 기능 |
| --- | --- |
| `httpx2-retry` | 지수 백오프 + 지터 + `Retry-After` 준수 |
| `httpx2-cache` | HTTP 캐시(RFC 9111), Redis/디스크 백엔드 |
| `httpx2-ratelimit` | 토큰 버킷, 호스트별 제한 |
| `httpx2-vcr` | 요청/응답 녹화·재생 |
| `httpx2-circuitbreaker` | 서킷 브레이커 |
| `httpx2-otel` | OpenTelemetry 자동 계측 |

유료 Pro: 중앙 대시보드, 분산 레이트리밋, SSO/감사 로그/SLA. 팀당 월 $49~299.
근거: `httpx-sse`, `httpx-ws` 같은 단일 기능 패키지도 큰 다운로드를 기록했고,
`httpx2` 생태계는 현재 비어 있어 선점 가능합니다.

### 4) SaaS: LLM API 비용·성능 모니터링 — 수익 상한 최대

`event_hooks` 기반으로 한 줄 계측부터 시작:

```python
client = instrument(httpx2.AsyncClient())
```

대시보드: 모델/엔드포인트별 토큰·비용 실시간, p50/p95/p99 레이턴시, 에러·재시도율,
예산 초과 알림, 프롬프트 로그(마스킹), 팀별 비용 분배.

| 티어 | 가격 |
| --- | --- |
| Free | 월 10만 요청 |
| Pro | 월 $29~99 |
| Team | 월 $199~499 |
| Enterprise | 연 $10k+ |

경쟁: Langfuse, Helicone, Pydantic Logfire.
차별화 방향: 한국어·국내 결제, 원화 환산 비용, 국내 LLM(하이퍼클로바, Solar) 지원.

### 5) 데이터 수집/API 통합 서비스

| 상품 | 가격 |
| --- | --- |
| 크롤링 대행 프로젝트 | 건당 200만~2,000만원 |
| 데이터 구독(B2B) | 월 50만~500만원 |
| API 통합 대행 | 건당 300만~1,500만원 |
| 스크래핑 API(SaaS) | 요청당 과금 |

준수사항: `robots.txt`, 저작권·개인정보보호법, 이용약관, 상대 서버 부하 최소화. 합법 범위 내에서만.

### 6) 보일러플레이트/템플릿 판매 — 가장 빠른 첫 수익

| 상품 | 가격 |
| --- | --- |
| FastAPI + httpx2 + React AI 챗봇 스타터킷 | $49~149 |
| 비동기 크롤러 템플릿 | $29~79 |
| AI 에이전트 스켈레톤 | $79~199 |
| GitHub 스폰서(프라이빗 저장소 접근) | 월 $5~50 |

### 7) MCP 서버 / 에이전트 툴 제품화

- OpenAPI 스펙 → MCP 툴 자동 변환 호스팅 (월 $19~199)
- 한국 서비스 전용 MCP 팩 (카카오, 네이버, 토스, 공공데이터포털)
- 사내 에이전트 구축 SI (2,000만~1억원)

### 8) 기업 LTS/보안 유지보수

구버전 보안 패치 백포트, SBOM + 취약점 리포트, 의존성 업그레이드 대행.
연 1,000만~5,000만원/고객사. 금융·공공·의료는 유지보수 중단 라이브러리가 감사 리스크입니다.

### 9) 오픈소스 평판 → 간접 수익

업스트림 기여로 "Pydantic 컨트리뷰터" 이력 확보 → 연봉·프리랜스 단가 상승,
컨퍼런스 발표(회당 50만~300만원), 해외 리모트 채용.

### 추천 로드맵

```
0~2개월    유튜브 10편 + httpx2-retry OSS 공개          (비용 0, 신뢰 확보)
2~4개월    전자책 + 보일러플레이트 판매                  (월 50만~300만원)
4~8개월    httpx2-migrate CLI 공개 → 컨설팅 전환         (건당 300만~800만원)
8~18개월   LLM 비용 모니터링 SaaS MVP                    (MRR 구축)
장기       LTS 계약 + 엔터프라이즈                       (연 단위 안정 매출)
```

### 리스크

- `httpx2`는 신생이라 전용 수요가 작을 수 있음 →
  "파이썬 비동기 / AI 통합"이라는 큰 주제 안에 `httpx2`를 배치하는 포지셔닝이 안전
- Pydantic 본사가 유사 영역(Logfire)을 이미 보유 → 정면 경쟁 대신 한국 시장·특화 영역
- OSS 확장 패키지는 업스트림 내장화로 가치가 사라질 수 있음(SSE/WebSocket이 그 예) →
  교육·컨설팅·모니터링처럼 운영·조직 문제를 푸는 제품이 더 방어적

---

## 13. 기타 발견 사항 및 권고

- `CLAUDE.md`가 일반 파일이 아니라 **내용 자체가 symlink 타겟인 형태**로 커밋되어 있습니다.
  동작은 하지만 일반 파일로 전환하는 것이 안전합니다.
- `AGENTS.md`의 규칙: 강제 푸시 금지, 인라인 코드는 단일 백틱 사용,
  코딩 에이전트는 자신을 공동 작성자로 넣지 않음(저작자는 엔지니어 본인만).
- CI는 Python 3.10~3.15 매트릭스에서 lint(ruff) → mypy strict → unasync 검증 →
  빌드 → 테스트 → 커버리지 100% 순으로 통과해야 합니다.
