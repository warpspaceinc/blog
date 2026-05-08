---
title: "LLM이 100개 entity짜리 JSON을 못 고치기 시작하는 지점에서"
date: 2026-05-08
draft: false
tags: ["llm", "json", "agent", "rfc6902", "오픈소스"]
categories: ["기술"]
summary: "LLM에게 큰 JSON을 critic 피드백으로 고치라고 시키면, 흔히 쓰는 'full-regen' 패턴은 우리가 생각한 것보다 훨씬 일찍 깨집니다. 100개 entity / 183개 edge KG에서 gpt-4o-mini가 어떻게, 왜 무너지는지 측정한 이야기 — 그리고 sub-agent 분해가 왜 'cost optimization'이 아니라 'correctness 컴포넌트'인지."
author: "Rick Choi"
---

LLM이 큰 JSON 객체 — 다막 소설 플랜, 에이전트 메모리, 생성된 지식그래프, 설정 트리 — 를 만들고, critic이 결함을 짚었을 때 이를 고치는 가장 흔한 패턴은 **"전체를 다시 출력하라"** 입니다. 우리는 이 패턴이 *언제* 깨지는지를 측정해봤습니다. 결과는 놀라웠습니다 — 깨지는 지점이 우리가 생각한 것보다 훨씬 일찍, 그리고 훨씬 dramatic하게 옵니다.

라이브러리 + 측정 결과 + 실험 코드 모두 [GitHub에 공개](https://github.com/warpspaceinc/json-correction)했습니다.

## 무엇을 측정했나

타깃은 *지식그래프(KG) 교정* 태스크. 정답 KG에 의도적으로 결함을 주입하고 (정답을 알기에 fix rate / drift를 객관 측정 가능), critic이 검출, LLM이 고치게 합니다. 결함 종류는 6개:

- `relation_swap`: edge의 predicate를 잘못된 것으로 교체
- `entity_drop`: entity를 삭제 (참조 무결성 깨뜨림)
- `entity_paraphrase`: label을 다른 표기로 변경
- `type_violation`: entity의 type을 뒤집음
- `dangling_ref`: 존재하지 않는 entity ID를 참조하는 edge 추가
- `duplicate`: 같은 entity를 다른 ID로 중복

비교 컨디션은 6개:

| 컨디션 | 메커니즘 |
|---|---|
| `oracle` | 정답 결함 로그를 받아 deterministic하게 inverse 적용. 상한선. LLM 안 씀. |
| `B0` | LLM에게 "전체 KG를 다시 출력해" — 흔한 production 패턴 |
| `B1` | LLM에게 "RFC 6902 patch ops 한 번 emit해" — JSON Whisperer 스타일 단발 |
| `O1` | B1을 critic loop에 넣음 (sub-agent 없음) |
| `O2` | O1 + path_finder sub-agent (전체 context) |
| `O2N` | O2 + context narrowing |

전부 `gpt-4o-mini` (OpenRouter), `temperature=0`. 측정 메트릭: structural fix rate, excess drift (critic이 고치라 안 한 필드의 변경 수), 토큰 수, wall clock.

## 실험 진행 — Phase A부터 ablation까지

체계적으로 답을 찾기 위해 phase를 나눠서 측정했습니다:

| Phase | 무엇을 검증 |
|---|---|
| **A** — oracle | 정답 결함 로그를 알 때 deterministic patcher가 100% 고치는지. 라이브러리 파이프라인 자체가 동작하는지. |
| **B-0** — B0 측정 | full-regen 표준 패턴의 drift 정량화 |
| **B-1** — B1 측정 | JSON Whisperer 스타일 단발 patch의 한계 |
| **B-2** — O1 / O2 | critic loop, path_finder sub-agent 효과 분리 |
| **B-3** — O2N | context narrowing 추가 후 closure |
| **B-4** — size sweep | KG가 커지면 어디서 깨지는가 |
| **B-5** — ablation | 각 component의 contribution 분리 |

작은 fixture (16 entity / 14 edge, 6 결함, 6 seeds 평균) 풀 비교:

![per-condition comparison](/blog/images/posts/llm-json-correction-loop/fig1_condition_compare.png)

| Cond | Fix% | Drift | Tokens | Wall |
|---|---|---|---|---|
| oracle (UB) | 100 | 0.2 | 0 | 0s |
| B0 full-regen | **100** | **4.0** | 2,506 | 14.5s |
| B1 single-shot patch | 55 | 5.5 | **1,774** | 3.1s |
| O1 loop+patch | 64 | 4.8 | 6,218 | 9.6s |
| O2 +path_finder | 84 | 5.5 | 13,244 | 19.7s |
| **O2N +narrowing** | **97** | **4.5** | 8,482 | 17.2s |

### 첫 번째 발견: full-regen은 "drift"한다

B0 (full-regen)는 fix rate 100%로 결함을 다 고치지만, **평균 4개 필드를 critic이 고치라 안 한 곳에서 변경**합니다. label paraphrase, edge 순서 변경, ID 미세 변경 등. 이게 "drift" — 100% fix rate 뒤에 숨은 작은 부패.

작은 KG에서는 드러나지 않을 수 있지만, **하류 시스템이 ID에 의존하는 경우** (예: chapter 4 시퀀스가 character ID를 참조하는 소설 시스템) drift가 silently 무결성을 깨뜨립니다. 그리고 LLM은 항상 "약간씩" drift합니다 — temperature=0에서도.

### per-seed 분산도 정직하게

평균만 보면 안전해 보이지만, seed별 분포는 다음과 같습니다:

![per-seed scatter](/blog/images/posts/llm-json-correction-loop/fig3_per_seed_scatter.png)

oracle은 항상 (0 drift, 100% fix). B0는 fix는 안정적이지만 drift가 0~7로 흔들립니다. B1/O1은 fix rate 자체가 0~100% 사이를 배회합니다 (단발의 단점). **O2N만 안정적으로 right corner에 모입니다** — 이게 sub-agents가 추가하는 "신뢰성".

### 두 번째 발견: naive surgical patcher는 더 못 고친다

직관적으로는 "통째로 다시 쓰는 대신 변경분만 patch로 주면 cheaper + drift 없음" 이어야 합니다. JSON Whisperer (arXiv 2510.04717)가 정확히 그 주장이고, 실제로 그들 벤치마크에선 31% 토큰 절감 + 5% 이내 품질 유지를 보였습니다.

우리가 측정한 결과는 다릅니다:

- **B1 (단발 patch)**: fix rate **55%**. critic이 잡은 결함의 절반 만 고침.
- **O1 (loop + patch, sub-agent 없음)**: **64%**. loop가 도움이 되지만 한참 모자람.

왜? 한 케이스를 직접 디버깅해봤습니다 (seed=1):

```
Critic이 critic이 잡은 8개 issue:
  /edges/2:  dangling_ref (Q2 not in entities)
  /edges/3:  dangling_ref (Q2 not in entities)
  /edges/4:  type_violation (predicate 'spouse_of' wants Person→Person, got Person→Film)
  /edges/5:  dangling_ref (Q15 not in entities)
  /edges/8:  type_violation (predicate 'starred_in' wants Person→Film, got Film→Film)
  ...

LLM이 emit한 patch ops 8개:
  {'op': 'remove', 'path': '/edges/2'}    ← Q2를 *복원*해야 하는데 edge를 *삭제*함
  {'op': 'remove', 'path': '/edges/3'}
  {'op': 'remove', 'path': '/edges/4'}
  ...
  {'op': 'remove', 'path': '/edges/9'}    ← /edges/9는 invalid: 위에서 5개 remove로 인덱스 어긋남
  {'op': 'remove', 'path': '/edges/10'}   ← 같은 이유로 invalid
```

**두 가지 failure mode**가 동시에 발생:

1. **"Lazy remove"**: LLM은 결함을 *고치는* 대신 결함이 있는 항목을 *삭제*합니다. dangling_ref(Q2 missing)의 올바른 fix는 Q2를 복원하는 거지만, LLM은 Q2를 참조하는 edge들을 통째로 지웁니다. 데이터 손실.

2. **"Index drift"**: array에서 `/edges/2`를 remove하면 `/edges/3`은 이제 원래 `/edges/4`를 가리킵니다. ascending 순서로 remove 8개 emit하면 5개째부터 invalid. 이건 JSON Whisperer 논문이 EASE 인코딩으로 풀려고 한 바로 그 문제.

프롬프트 강화로 두 failure는 어느 정도 완화 가능합니다:
- "복원 > 삭제, 데이터 잃지 마라"
- "remove ops은 descending index 순서로"

하지만 더 근본적인 세 번째 failure는 프롬프트로 안 풀립니다.

### 세 번째 발견: symptom과 root cause는 다르다

가장 어려운 결함은 `type_violation` 입니다. Q12라는 entity의 type이 Film에서 Person으로 뒤집히면, **Q12의 type field 자체는 그냥 "Person"** 이라 critic이 *Q12에서 직접 잡지 못합니다*. 대신 Q12가 등장하는 모든 edge에서 type_violation이 발견됩니다 — 예를 들어 "Song Kang-ho starred_in Q12" edge는 starred_in이 (Person, Film)을 요구하는데 (Person, Person)으로 보임.

critic은 *증상* (`/edges/N`)을 가리킵니다. *근본 원인* (`/entities/Q12/type`)이 아니라.

LLM이 `/edges/N`을 받으면, *edge의 predicate를 바꿔* type_violation을 "고칩니다". critic은 만족하지만 — Q12의 type은 여전히 틀렸고, Q12를 참조하는 다른 edge들이 다음 iteration에서 또 flag됨, 그리고 LLM은 그것들의 predicate도 바꿉니다. 결국 모든 관련 edge가 silently 부패한 채로 critic은 happily 통과시킵니다.

이게 **path_finder sub-agent의 motivation**입니다.

### 네 번째: path_finder + narrowing이 gap을 닫는다

path_finder는 critic이 가리킨 symptom pointer를 분석해서 root cause pointer로 redirect하는 LLM call입니다. "이 edge에 type_violation이 보이는데, predicate-type 표를 보니 Q12의 type이 잘못됐다, redirect: `/edges/N` → `/entities/Q12/type`."

context narrowing은 sub-agent와 patcher 둘 다에게 *전체 KG*가 아니라 *영향받는 슬라이스*만 보여줍니다 — flagged pointer가 가리키는 entity/edge + 1-hop 이웃.

결과:
- O1 (loop만): 64%
- O2 (+ path_finder, full context): 84%
- **O2N (+ narrowing): 97%**

작은 fixture에선 O2N이 oracle 100%까지 3pp 이내로 따라잡습니다.

## 100 entity로 키우면? — 깨지는 지점

이게 진짜 흥미로운 부분입니다.

작은 fixture만 보면 O2N의 토큰 비용은 B0보다 3.4배 *높습니다*. Sub-agent 호출이 추가되니까. "Surgical patcher가 비싼 거 아닌가?" 하는 의문이 들 수 있습니다.

KG를 100 entity / 183 edge로 키워서 똑같이 8 결함 주입, B0 vs O2N 비교:

![size sweep](/blog/images/posts/llm-json-correction-loop/fig2_size_sweep.png)

**size=100에서 B0가 *catastrophic하게 실패*합니다:**

```
size=100 (100 ent, 183 edges, 8 defects):
  B0:   fix=  0%  tokens=73,740   wall=435s
  O2N:  fix=100%  tokens=17,117   wall=26s
```

8.5분 동안 73K 토큰을 태우고 한 결함도 못 고침. 5번 hardcap.

원인: `gpt-4o-mini`의 `max_tokens=8192` ceiling. 100 entity + 183 edge JSON을 다시 출력하기엔 모자라서 array 중간에 잘립니다 (truncated). critic은 잘린 JSON을 보고 *더 많은* dangling_ref를 발견합니다. 다음 iteration에서 다시 regen → 또 잘림 → 반복. Loop의 hardcap 정책이 5번에 멈춰주지 않았으면 무한 토큰 소모.

O2N은? **17K 토큰, 26초, 100% fix.** 토큰이 거의 안 늘었습니다 (작은 fixture 18K → 큰 KG 17K).

이유: surgical patcher의 토큰 비용은 *KG 크기*가 아니라 *수정 면적*에 비례합니다. 8개 결함 → 영향받는 슬라이스 ~10-15 entity → 일정한 비용. KG가 100배 커져도 17K 토큰.

이게 우리 라이브러리의 **central empirical claim** 입니다:

> **Surgical patching의 비용은 edit footprint에 비례. Full-regen의 비용은 document size에 비례. 비자명한 크기의 JSON state에서, surgical은 strictly dominate한다 — 토큰 면에서, 그리고 *작동 여부* 면에서.**

깨지는 지점은 모델별로 다릅니다 (Claude Opus 200K context는 더 큰 KG까지 버팁니다). 하지만 *어딘가에는 깨지는 지점이 있다는 것 자체*가 구조적입니다 — auto-regressive decoding은 출력이 길수록 wall time과 per-token error rate가 모두 증가합니다.

## Ablation — narrowing은 cost가 아니라 correctness다

처음 개발할 때 우리는 narrowing을 "토큰 절감 trick" 정도로 생각했습니다. 그런데 100-entity 사이즈에서 ablation을 돌려보니:

![ablation](/blog/images/posts/llm-json-correction-loop/fig4_ablation_size100.png)

| 컨디션 | path_finder | narrowing | Fix% | Drift | Tokens |
|---|:---:|:---:|---|---|---|
| B0 | n/a | n/a | 0% | 8 | 73,740 |
| O1 | ✗ | ✗ | 35% | 23 | 42,621 |
| O1N | ✗ | ✓ | 57% | 13 | 14,392 |
| **O2** | ✓ | ✗ | **FAILED** | - | - |
| O2N | ✓ | ✓ | 100% | 6 | 17,117 |

**O2 (path_finder + full context) 자체가 동작 안 함.** path_finder가 emit하는 JSON output이 *parse 불가능*했습니다. full-context prompt 길이가 ~15K input tokens에 도달하면 — 모델이 표면적으로는 훨씬 긴 context를 지원함에도 불구하고 — structured output 신뢰도가 떨어집니다.

즉:

> **Narrowing은 "premature optimization"이 아니다. Sub-agent들이 동작하기 위한 prerequisite다.**

Production에서 "narrowing은 나중에 최적화로 추가하자"고 미루면, 어느 순간 sub-agent가 silently JSON을 깨기 시작합니다. 우리는 측정했습니다.

## 정리

큰 JSON state를 LLM-driven critic loop으로 교정할 때:

1. **Full-regen은 우리가 생각하는 것보다 일찍 깨진다.** 100 entity는 production-realistic 사이즈고, gpt-4o-mini로는 동작하지 않습니다. 더 큰 모델은 깨지는 지점이 더 뒤지만, *어딘가에 깨지는 지점이 있다는 사실 자체*는 auto-regressive decoding의 구조적 특성입니다.

2. **단순 surgical patcher (JSON Whisperer + critic loop) 만으로는 부족하다.** Symptom-vs-root-cause 함정 때문에 fix rate가 64%에 정체.

3. **Sub-agents (path_finder)가 fix rate를 닫는다.** 33pp 향상.

4. **Context narrowing은 cost optimization이 아니라 correctness 컴포넌트다.** 빼면 sub-agent 자체가 동작 안 함.

각 컴포넌트가 load-bearing — 하나만 빼도 무너집니다.

### 한계 (정직하게)

이 측정의 scope:
- **단일 모델** (`gpt-4o-mini`). Claude 3.5 / GPT-4o 큰 모델은 깨지는 지점이 더 뒤일 것. 하지만 surgical의 *상대적* 우위는 KG가 충분히 크면 모든 모델에서 발생.
- **단일 도메인** (KG). Generality는 architectural한 argument. 다른 도메인 (config refactor, 소설 narrative state)에서 검증 필요.
- **합성 결함**. 실제 LLM이 만드는 결함 분포와는 다를 수 있음.
- **Ablation은 size=100에서 seed=42 1회**. 더 robust하려면 5+ seeds.

### 더 알아보기

- 라이브러리: [github.com/warpspaceinc/json-correction](https://github.com/warpspaceinc/json-correction). Apache-2.0, `pip install` 가능 (PyPI 곧 등록), quickstart 예제부터 KG 벤치마크까지 포함.
- 본인 도메인 적용 시: critic / path_finder의 도메인 지식 (예: schema, constraint) 만 plug-in 하면 됩니다. Loop driver, planner, convergence policy는 그대로.

큰 JSON 다루는 LLM 시스템 운영하시는 분들, 한 번 측정해보세요 — 본인 시스템이 *언제* 깨지기 시작하는지 모르고 있을 가능성이 높습니다. 측정해서 알게 된 게 우리에게 가장 큰 수확이었습니다.
