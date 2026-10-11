# Phase 0 — DevSpace RPi3 Ubuntu Server 테스트 보고서

> 실행 날짜: **2026-10-12 KST** (작업은 10월 11일 밤 시작)  
> 실행 공간: DevSpace `ws_bddfce5d90` / `/home/elsaphaba/HongDev/chat-on-steroids-headless`  
> 실행 서버: Raspberry Pi 3, Ubuntu **24.04.5 LTS**, **aarch64**, kernel `6.8.0-1065-raspi`  
> Python venv: `/home/elsaphaba/HongDev/HongDevVenv` / Python **3.12.3** / pip 26.1  
> Node: **v24.14.0**, npm **11.17.0** (별도 설치된 Node v24.21.0 존재)  
> 작업 트리: `main`, HEAD **`1d23ecdb8ab4dd5864f7f835778f539c51128526`**, tracked worktree **clean**  
> 이 결과는 **RPi3에서 실행한 정적 검사**로서 독립 Headless 서비스의 성공적인 실행, 타입 검사, Vitest 통과를 의미하지 않음

## 1. 수행 결과 요약

| 항목 | 결과 | 근거 및 의미 |
| --- | --- | --- |
| HongDevVenv 활성화 | **PASS** | `$VIRTUAL_ENV` 및 Python `sys.prefix` 확인 |
| 소스 계약·구조 정적 검사 | **18/18 PASS** | Venv Python 3.12를 통한 read-only 검사 |
| TypeScript 소스 전수 경로 스캔 | **PASS (lexical)** | `src` TypeScript 파일 299개, 모듈 의존성 경로 1,175개 |
| 상대 경로 import 해석 | **PASS** | TS/TSX 및 JSON 리소스 포함, 미해결 경로 **0개** |
| MCP 브라우저 의존성 추적 | **확인됨(포팅 블로커)** | `mcp/server.ts → mcp/tools.ts → mcp/tools-core.ts → bridge.ts` |
| Plugin Electron 의존성 추적 | **확인됨(포팅 블로커)** | `mcp/tools-plugins.ts → plugins/manager.ts → secrets.ts → electron` |
| `npm ci` 의존성 설치 | **미완료** | 첫 실행 호출 제한에 도달; 두 번째 오프라인 실행 `ENOTCACHED` |
| `npm run typecheck` | **미실시 — 차단** | npm 설치 불완전 |
| `npm run verify:ci` / Vitest | **미실시 — 차단** | npm 설치 불완전 / 저메모리 ARM 장치 |
| 서버용 Headless runtime smoke test | **미실시 — 구현 전** | `src/headless/main.ts` 아직 없음 |
| Git tracked 파일 변경 | **없음** | `git status --porcelain` 빈 출력 |

**중요:** 완료 항목은 `18개 정적 검사`에 한정한다. 런타임 및 CI 테스트를 `PASS`로 집계하지 않는다.

## 2. 실행 환경 점검

```text
VIRTUAL_ENV=/home/elsaphaba/HongDev/HongDevVenv
Python=3.12.3
Host=Linux aarch64 / Ubuntu 24.04.5 LTS
Node.js=24.14.0
npm=11.17.0
Alternate installed Node.js=24.21.0
Memory=899 MiB RAM / 1.4 GiB Swap
Disk=/dev/mmcblk0p2, 115 GB total, ~44 GB free
Repository branch=main (matches upstream/fork baseline source commit)
Tracked git status=clean
```

**환경 제약:** Python Venv는 Python 명령과 패키지를 격리하지만 npm/Node.js 의존성을 설치하거나 TypeScript 테스트 런타임을 제공하지 않는다. 현재 `HongDevVenv` 안의 Python으로 소스 구조를 확인한 것과, npm/Vitest 통합 테스트 성공은 구분해야 한다. 원본 CI는 Ubuntu 24.04 **x86_64**와 **Node 22**를 사용하므로 RPi3 ARM64 + Node 24 테스트는 확장 환경 증빙이지 기존 CI와 동일한 재현 환경이 아니다.

## 3. Venv 정적 검사 18건 결과

테스트는 `source /home/elsaphaba/HongDev/HongDevVenv/bin/activate` 이후 프로젝트 최상위에서 `python -B -c` 로 실행한 **읽기 전용 Python 소스·계약 검사**이다. 프로젝트 파일을 생성·수정하지 않았다.

| No. | 검사 | 결과 |
| ---: | --- | --- |
| 1 | Git HEAD가 고정된 원본 SHA인지 | PASS |
| 2 | `HongDevVenv` 활성화 여부 | PASS |
| 3 | `src` 파일 인벤토리: 총 299, `main` 177, `shared` 55 | PASS |
| 4 | 기존 Vitest `test/*.test.ts` 파일 325개 확인 | PASS |
| 5 | 원본 플러그인 카탈로그 7개 확인 | PASS |
| 6 | 지원 대상 5개에서 Blender/Unity 제외되는 계약 확인 | PASS |
| 7 | `@playwright/mcp` 및 대표 browser tool 선언 유지 | PASS |
| 8 | 원본 MCP `core`/`desktop`/`plugins` surface 선언 확인 | PASS |
| 9 | Headless 독립 entrypoint 미구현 상태 확인 | PASS |
| 10 | `secrets.ts` Electron 직접 의존 확인 | PASS |
| 11 | `plugins/manager.ts` Electron OAuth opener 확인 | PASS |
| 12 | `mcp/kernel.ts` + `tools-core.ts` 브라우저 브리지 직접 참조 확인 | PASS |
| 13 | 원본 CI의 Node 22 / Ubuntu 24.04 매트릭스 확인 | PASS |
| 14 | `typecheck`/`test`/`verify:ci` npm scripts 선언 확인 | PASS |
| 15 | 전체 `src`의 상대경로 TypeScript/TSX/JSON import 해석 | PASS |
| 16 | MCP 서버 → 브라우저 브리지 import chain 확인 | PASS |
| 17 | Plugins surface → Electron secrets import chain 확인 | PASS |
| 18 | Git tracked worktree 변경 없음 | PASS |

합계 **PASS 18 / FAIL 0**.

**검사기 수정 기록:** 첫 실행은 17/18로, `src/renderer/pet-assets/animations.json` 에 대한 상대경로 import 3건을 검사기가 TypeScript 파일로만 해석해 `FAIL`을 반환했다. 실제 JSON 파일이 존재함을 확인한 후 `.json` 리소스 import 해석을 추가하여 동일한 18개 검사를 재실행했고 **18/18 통과**했다. 이 실패는 소스 결함이 아닌 **검사 스크립트의 false positive**였다.

## 4. 전체 상대경로 Import 조사

- `src/main` TypeScript 177개, `src/shared` 55개, `src/renderer` 62개, `src/preload` 5개 = **총 299개**.
- Python 정적 스캐너는 `import`, `export ... from`, `require`의 **literal string** 경로를 추출하고 `.js` → `.ts`/`.tsx`, `index.ts`/`index.tsx` 및 리소스 `.json`을 해석한다.
- 분석한 TypeScript 소스 간 **unique directed relative import edges = 1,175**. TypeScript 파일이 아닌 JSON import는 상대경로가 존재함을 확인하지만 TS 의존성 그래프의 edge 수에는 포함하지 않는다.
- **경로 해석 실패 = 0개**.
- 현재 기준선에는 **Headless 전용 entrypoint가 존재하지 않아** Headless 번들의 금지 모듈 도달 수를 직접 검증할 수 없다. 대신 기존 MCP/Plugins 진입점의 대표 forbidden 경로를 확인하였다.

### 대표 의존성 경로

```text
src/main/mcp/server.ts
  → src/main/mcp/tools.ts
  → src/main/mcp/tools-core.ts
  → src/main/bridge.ts                      [Headless 제외 필요]

src/main/mcp/tools-plugins.ts
  → src/main/plugins/manager.ts
  → src/main/secrets.ts                     [Electron safeStorage 교체 필요]
```

**한계:** 이는 TypeScript Compiler API의 AST 분석이나 실제 Node 런타임 import graph 검증이 아니다. 주석·문자열에 등장하는 유사 구문이 포함될 수 있고, 조건부 import, type-only import, 재수출 및 번들러 tree-shaking semantics를 완전히 구분하지 않는다. `src` 299개 **정적 경로 전수 스캔**은 수행했지만 **CI용 AST/import graph 게이트의 완성/통과**로 표시하지 않는다.

## 5. npm 의존성 설치 시도와 차단 사항

### 5.1 초기 시도

```bash
source /home/elsaphaba/HongDev/HongDevVenv/bin/activate
ELECTRON_SKIP_BINARY_DOWNLOAD=1 PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
npm_config_maxsockets=2 npm ci --prefer-offline --no-audit --no-fund
```

- npm 명령은 장치에 상당한 다운로드/설치 부하를 발생시켰고 DevSpace 호출 시간 제한에 도달하였다.
- 호출 후 npm 프로세스는 실행 중이지 않았으며, `node_modules` 디렉터리는 **불완전한 상태**였다.
- 이를 성공적인 `npm ci` 로 집계하지 않음.

### 5.2 캐시 기반 재시도

```bash
ELECTRON_SKIP_BINARY_DOWNLOAD=1 PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
NODE_OPTIONS=--max-old-space-size=320 npm_config_maxsockets=1 \
npm ci --offline --ignore-scripts --no-audit --no-fund
```

- **실패/중단:** `ENOTCACHED` — `yargs@17.7.3` tarball이 npm 로컬 캐시에 없으며 오프라인 설치를 완료할 수 없었다.
- 버전 경고: `@asamuzakjp/css-color@7.1.2`, `@asamuzakjp/dom-selector@9.2.2`, `jsdom@30.1.1`, `w3c-xmlserializer@6.0.0` 는 현재 `Node 24.14.0`에서 지원 범위 미달(`^22.22.2 || ^24.15.0 || >=26.0.0` 요구)을 경고했다.
- `HongDev`에는 **Node 24.21.0** 이 설치되어 있어 버전 불일치 해소 후보지만, **이번 실행에서 Node 24.21로 타입 검사/Vitest를 성공적으로 재실행한 사실은 없다.**
- 오프라인 npm 설치 실패 이후 `node_modules/.bin` 이 존재하지 않음을 확인; TypeScript 및 Vitest 실행 파일을 정상 설치된 것으로 취급하지 않는다.

### 5.3 권장 후속 조치

1. **먼저 Node 24.21.0 또는 공식 CI 호환 Node 22의 요구 버전을 선정**하고 일관된 PATH에서 실행한다. 앱 기본 지원 런타임은 실제 테스트가 성공한 뒤 확정한다.
2. 가능하면 RAM이 충분한 **Ubuntu 24.04 x86_64** 환경에서 `npm ci` + `npm run typecheck` + `npm run verify:ci`를 완주한다.
3. ARM64 RPi3 테스트는 메모리·스왑·빌드 시간과 native prebuilt availability를 고려하여 **작은 단위의 테스트만 선택적으로** 실행한다. 설치가 온전히 성공하기 전엔 Vitest나 타입 검사 성공을 추정하지 않는다.
4. 실패한 설치가 남긴 **ignored `node_modules`** 디렉터리는 자동 삭제하지 않았다. 기존 tracked 파일은 clean이며, 디렉터리 정리는 명시적 범위가 확정된 후 수행한다.
5. 추가할 Headless CI는 Core/Plugins만 빌드하는 Node-only package를 목표로 하여 Electron/GUI 패키지를 강제로 설치하지 않아야 한다.

## 6. Phase 0 상태 갱신

| 게이트 | 현재 상태 |
| --- | --- |
| P0-G1: 전수 TypeScript import 경로 스캔 | **부분 완료**: `src` 299개 lexical 스캔, 1,175 edges, 상대 import 0 unresolved. 실제 AST/CI gate 아직 없음 |
| P0-G2: fork 자체 `npm ci`/typecheck/verify:ci 재현 | **차단**: RPi3 설치 미완료(호출 시간 제한/`ENOTCACHED`/엔진 경고) |
| P0-G3: Headless 전용 entrypoint의 금지 import 검증 | **미착수**: Phase 1 구현 필요 |
| P0-G4: Linux SecretStore 계약 | **설계 미확정**, 현재 Electron 의존성을 테스트로 확인 |
| P0-G5: permission별 live `tools/list` fixture | **미실시**: npm 의존성 미설치 및 MCP headless 미구현 |

**요약:** Phase 0의 *정적 코드 조사 및 계약 검사*는 RPi3 환경에서 실행했고 **18/18 통과**했다. **Node/npm 런타임 단위·통합 테스트는 통과하지 않았다(미실시)**. 이 상태로 Phase 0 전체를 완료 처리하지 않는다.
