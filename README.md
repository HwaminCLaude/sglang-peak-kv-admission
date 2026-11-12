한국어 · [English](README.en.md)

# 긴 LLM 답변이 스트리밍 도중 최대 1분 남짓 멈추는 이유

**SGLang 0.5.18 · KV 부족으로 생기는 요청 선점(retraction)의 원인 규명과 <!--n:patch.lines_added-->102<!--/n-->줄 패치, 그리고 그 대가**

[![cpu-tests](https://github.com/HwaminJung00/sglang-peak-kv-admission/actions/workflows/cpu-tests.yml/badge.svg)](https://github.com/HwaminJung00/sglang-peak-kv-admission/actions/workflows/cpu-tests.yml)

Qwen3-4B · RTX 4090 1장 · 답변 길이를 미리 정해 둔 합성 요청 <!--n:trace.n_requests-->120<!--/n-->개

SGLang 0.5.18은 KV 캐시 메모리가 바닥나면 생성 중이던 요청의 메모리를 회수하고 그 요청을 대기열 맨 뒤로 보낸다. 이것이 선점(retraction)이고, 선점된 답변은 다시 차례가 올 때까지 멈춘다. 원인은 기본 설정의 스케줄러에 있다. 스케줄러는 실행 중인 요청이 앞으로 쓸 KV 메모리를 실제보다 훨씬 적게 예약한다. 1만 토큰을 더 생성할 요청에도 약 400토큰만 예약하는 경우가 있어서, 새 요청을 너무 많이 받아들이고 곧 메모리가 바닥난다. 감당 한계를 크게 넘는 초당 2요청 부하에서는 세 번의 실행 모두 답변 <!--n:trace.n_requests-->120<!--/n-->개 중 <!--n:q2.default.stall5_range-->19–20<!--/n-->개가 생성 도중 <!--n:q2.default.stall_range_s-->30.8–64.8<!--/n-->초씩 멈췄다.

멈춤이 곧 선점이라는 것은 패치하지 않은 SGLang을 초당 0.5~8요청의 7단계 부하로 돌려 확인했다. 서버 로그의 선점 <!--n:diag.stall_match-->89/89<!--/n-->건이 클라이언트에서 잰 5초 넘는 멈춤과 빠짐없이 1:1로 맞았다.

SGLang에는 길이가 정해진 요청에 대해 남은 출력 길이를 전부 따져 보는 정확한 계산이 이미 있지만, 프롬프트 공유 캐시(radix cache)를 끈 경우에만 쓰인다. 같은 계산을 캐시를 켠 상태에서도 하는 검사를 덧붙이는 <!--n:patch.lines_added-->102<!--/n-->줄 패치를 만들었다. 패치의 옵션(`--enable-peak-kv-reservation`, 기본은 꺼짐)을 켜면 초당 2요청에서 멈춘 요청이 <!--n:q2.mine.stall5-->0<!--/n-->개가 되고(3회 측정), 캐시를 끄는 기존 옵션 `--disable-radix-cache`만으로도 <!--n:q2.noradix.stall5-->0<!--/n-->개가 된다(1회 측정). 대신 패치를 켜면 첫 토큰까지 걸리는 시간(TTFT)의 p99가 <!--n:q2.default.ttft_p99_s-->64.2<!--/n-->초에서 <!--n:q2.mine.ttft_p99_s-->92.3<!--/n-->초로 늘어난다. 모든 요청이 정해 둔 길이를 다 채울 때까지 생성하는 조건, 곧 서버가 답변 길이를 미리 아는 조건에서 잰 결과다.

5주 스터디의 개인 과제이고, 패치 방향은 스터디에서 받은 힌트를 따랐다. 제출 뒤 검증과 문서화에는 AI 도구를 썼다([누가 무엇을 했나](#누가-무엇을-했나)).

**생성 중 멈춤(빨강)이 시작 전 대기(회색)로 옮겨 간다**

![초당 2요청에서 요청별 타임라인: 기본값과 패치](docs/figures/hero_timeline.png)

*한 줄이 요청 하나이고 파랑은 생성 중인 구간이다. 그림 속 q2는 초당 2요청, default (flag off)는 패치 옵션을 끈 기본값, α=1.0은 패치가 남은 길이를 전부 예약한다는 뜻이다. 세 번 중 한 번의 실행(r2)을 그린 것이라 멈춘 시간의 범위가 본문과 조금 다르다. 그림 맨 아래 줄의 09-07 sweep은 위의 7단계 부하 실험이다.*

## 결과 한눈에

| 초당 2요청, 요청 <!--n:trace.n_requests-->120<!--/n-->개 | 생성 중 멈춘 요청 | 요청별 최대 토큰 간격의 p99 | TTFT p99 | goodput |
|---|---|---|---|---|
| 기본값: 패치를 넣고 옵션을 끔 (3회 중앙값) | <!--n:q2.default.stall5-->19<!--/n-->개 | <!--n:q2.default.maxitl_p99_s-->63.1<!--/n-->초 | <!--n:q2.default.ttft_p99_s-->64.2<!--/n-->초 | <!--n:q2.default.goodput-->0.403<!--/n--> |
| **패치**: 옵션을 켬 (3회 중앙값) | <!--n:q2.mine.stall5-->0<!--/n-->개 | <!--n:q2.mine.maxitl_p99_s-->0.11<!--/n-->초 | <!--n:q2.mine.ttft_p99_s-->92.3<!--/n-->초 | <!--n:q2.mine.goodput-->0.440<!--/n--> |
| `--disable-radix-cache` (기존 옵션, 1회) | <!--n:q2.noradix.stall5-->0<!--/n-->개 | <!--n:q2.noradix.maxitl_p99_s-->0.12<!--/n-->초 | <!--n:q2.noradix.ttft_p99_s-->89.9<!--/n-->초 | <!--n:q2.noradix.goodput-->0.435<!--/n--> |
| `--max-running-requests 40` (기존 옵션, 1회) | <!--n:q2.tuned_n40.stall5-->0<!--/n-->개 | <!--n:q2.tuned_n40.maxitl_p99_s-->0.09<!--/n-->초 | <!--n:q2.tuned_n40.ttft_p99_s-->93.5<!--/n-->초 | <!--n:q2.tuned_n40.goodput-->0.432<!--/n--> |

최대 토큰 간격은 한 요청의 스트림이 가장 오래 끊긴 시간이다. goodput은 출력 토큰당 평균 시간이 <!--n:trace.tpot_slo_ms-->60<!--/n--> ms 이하인 요청 수를, 첫 요청을 보낸 때부터 마지막 답변이 끝날 때까지의 시간으로 나눈 값(req/s)이고 TTFT는 따지지 않는다. goodput 차이의 절반은 가장 긴 답변 하나가 일찍 끝난 데서 나온다([계산은 기술 요약](docs/TECHNICAL_SUMMARY.ko.md#숫자를-읽는-법)). 패치하지 않은 SGLang에서도 기본값처럼 <!--n:q2.upstream.stall5-->19<!--/n-->개가 멈췄다(1회 측정).

## 원인과 근거

기본 설정에서는 실행 중인 요청마다 `min(남은 토큰, 4096) × new_token_ratio`만큼만 예약하고([코드](third_party/sglang_v0_5_18/schedule_policy.py#L654)), 이 비율은 0.7에서 0.098까지 내려간다. 정확한 계산은 [여기](third_party/sglang_v0_5_18/schedule_policy.py#L1065)에 있고 radix cache를 끈 경우에만 실행된다([코드](third_party/sglang_v0_5_18/schedule_policy.py#L1213)).

1. **선점 하나에 멈춤 하나.** 위의 <!--n:diag.stall_match-->89/89<!--/n--> 대응에서 짝 없는 멈춤도 없었다(부하마다 1회 실행).
2. **계산한 한계 부근에서 선점이 시작된다.** KV 풀이 선점 없이 감당할 수 있는 도착률을 풀 크기와 답변 길이로 계산하면 <!--n:diag.little_lambda_max-->0.56–0.67<!--/n--> req/s이고, 실제로 초당 0.5요청에서는 선점이 없다가 <!--n:diag.first_retraction_qps-->0.6<!--/n-->요청에서 처음 나왔다.
3. **멈춘 시간은 재계산이 아니라 대기열에서 기다린 시간이다.** 선점된 요청이 다시 실행된 시점을 로그에서 찾을 수 있었던 <!--n:diag.reprefill_confirmed-->38<!--/n-->건에서, 그때까지 걸린 시간과 멈춘 시간의 차이가 <!--n:diag.reprefill_delay_vs_stall_max_s-->0.90<!--/n-->초 이내였다.
4. **초당 2요청에서 더 보수적으로 받아들이면 선점이 사라진다.** 표의 세 방법은 모두 선점이 없었고, 원인과 무관해 보이는 chunked prefill을 끈 비교 실험에서는 선점이 그대로였다([수치는 기술 요약](docs/TECHNICAL_SUMMARY.ko.md#진단-긴-답변이-멈추는-이유)).

## 이 결과가 말해 주지 않는 것

- **멈춤은 옮겨 갔을 뿐이다.** 첫 토큰을 10초 넘게 기다린 요청이 <!--n:q2.default.wait10-->43<!--/n-->개에서 <!--n:q2.mine.wait10-->60<!--/n-->개로 늘었다(3회 중앙값). 대화형 서비스에는 그대로 못 쓴다.
- **패치가 기존 옵션보다 낫다는 근거는 없다.** 표의 goodput 차이는 작고 옵션 쪽은 1회 측정이다. 패치는 radix cache를 켜 둔다는 점이 다르지만, 요청들이 공유하는 프롬프트 접두부(prefix)가 <!--n:trace.shared_prefix_tokens-->36<!--/n-->토큰뿐이라 그 가치는 재지 못했다.
- **답변 길이를 미리 아는 조건이다.** 길이를 모르는 요청에서는 최대 길이(`max_tokens`)가 상한일 뿐이라, 전부 예약하면 지나치게 보수적이 된다. 그런 트래픽에서는 재지 않았다.
- **환경이 하나다.** GPU, 모델, 버전, 요청 세트가 하나씩이고 `--schedule-conservativeness`는 재지 않았다.

## 누가 무엇을 했나

- **스터디가 준 것.** 요청 세트, 측정 도구, 실험 절차, "요청을 받아들일 때 peak KV를 보수적으로 예약해 보라"는 힌트([스터디 저장소](https://github.com/mlleo/inference-engine-study)).
- **내가 한 것.** 부하별 측정, 선점과 멈춤의 1:1 대응 확인, 정확한 계산이 기본 설정에서 꺼져 있다는 발견, 패치, 실험 자동화와 <!--n:budget.runs-->53<!--/n-->회 실행, 본 실험 전에 정해 둔 선택 규칙, 분석. GPU 측정은 모두 제출 전에 했다.
- **제출 뒤 AI 도구([Claude Code](https://claude.com/claude-code))와 한 것.** 같은 데이터의 재계산과 검증, 동치 테스트, 지표 계산 모듈, 그림, 이 문서를 포함한 정정 문서 초안(공개본은 직접 검토).

## 직접 확인하고 더 읽기

GPU 없이 1분 안에 패치의 판정 논리와 문서의 수치를 확인할 수 있다(GPU 측정을 다시 하지는 않는다).

```bash
python3 tests/run_cpu_tests.py               # 패치 판정 논리의 단위 테스트
python3 tests/test_upstream_equivalence.py   # 손대지 않은 SGLang 함수와 판정 비교
python3 tools/check_numbers.py               # 문서에 마커로 표시한 수치를 데이터에서 계산한 값과 대조
```

- [패치](patch/README.md): +<!--n:patch.lines_added-->102<!--/n-->/−<!--n:patch.lines_removed-->0<!--/n-->줄. 기존 검사 위에 조건을 하나 더할 뿐이라 요청을 보류할 수는 있어도, 원래 보류되던 요청을 받아들이지는 않는다([fork 커밋](https://github.com/HwaminJung00/sglang/commit/5157a47e8649c6b3cc43705114fa8ec41d8d733d))
- [기술 요약](docs/TECHNICAL_SUMMARY.ko.md), 10분: 부하별 결과, 비교 실험, 평가 방법, 예상 질문, 재현 절차
- [보고서](docs/REPORT.ko.md), 40분: 전체 분석과 부록
- [정오표](docs/ERRATA.md): 제출한 보고서의 오류와 누락 34건. 초당 2요청 측정값은 같고, 영향이 큰 7건은 그 값을 읽는 법을 바꿨다
- [CPU 테스트](tests/README.md) · [데이터](data/README.md) · [실험 기록](docs/EXPERIMENTS.md) · [본 실험 전에 써 둔 계획](docs/preregistration/plan_2026-09-13.md)

코드는 Apache-2.0, 문서와 데이터는 CC BY 4.0이다([LICENSE](LICENSE), [LICENSE-docs](LICENSE-docs), [NOTICE](NOTICE)). SGLang 프로젝트와 관계없는 개인 저장소다.
