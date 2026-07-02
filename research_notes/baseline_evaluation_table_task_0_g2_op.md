# Baseline Evaluation Report — `table_task_0_g2_op`

**작성일**: 2026-07-02
**목적**: GenieSim robustness/약점 연구(포트폴리오 프로젝트)의 baseline 재현 및 약점 분석 단계 기록

## 1. 실험 개요

| 항목 | 값 |
|---|---|
| 로봇 | `G2_omnipicker` |
| 정책(model) | **ACoT-VLA π0.5**, checkpoint `instruction_and_robust_pi05` (env `PI05_GENIE_SIM_INSTRUCTION_AND_ROBUST`, `scripts/serve_policy.py --port 8999`) |
| 평가 하네스 | `geniesim_benchmark` (`model_arc=corobot`, websocket client → 로컬 추론 서버) |
| 태스크 그룹 | `table_task_0_g2_op` 산하 5개 instruction-following 카테고리 |
| 총 에피소드 수 | 4,401 |
| 평가 단계(step) | `Follow`(접근/정렬) → `PickUpOnGripper`(파지, z_diff>0.02m & gripper-object 거리<0.2m 이면 성공, 이진 판정) |

> 참고: 이 5개 태스크는 AgiBot World Challenge의 "instruction" board 성격(자연어 지시 이해 + 파지)이며, 기존에 파악해둔 leaderboard 상의 "robust"(perturbation) 축 수치(Robot Pose 0.26, Camera Position 0.31)와는 다른 벤치마크 세트다. 직접 비교하지 말 것.

### 소스 데이터
- `output/benchmark/table_task_0_g2_op/pick_block_number/evaluate_ret_00.json`
- `output/benchmark/table_task_0_g2_op/pick_block_shape/evaluate_ret_00.json`
- `output/benchmark/table_task_0_g2_op/pick_common_sense/evaluate_ret_00.json`
- `output/benchmark/table_task_0_g2_op/pick_follow_logic_or/evaluate_ret_00.json`
- `output/benchmark/table_task_0_g2_op/pick_object_type/evaluate_ret_00.json`

## 2. 태스크별 요약

| 태스크 | n | Follow (가중평균) | PickUp 성공률(≈E2E) | 종합 average |
|---|---:|---:|---:|---:|
| pick_follow_logic_or | 1000 | 0.918 | **0.720** | 0.819 |
| pick_block_number | 1070 | 0.911 | **0.687** | 0.799 |
| pick_object_type | 331 | 0.961 | **0.499** | 0.730 |
| pick_block_shape | 1000 | 0.662 | **0.351** | 0.507 |
| pick_common_sense | 1000 | 0.428 | **0.276** | 0.352 |

`PickUp 성공률` 컬럼이 사실상 태스크 최종 성공률(E2E)을 결정한다. 5개 중 **pick_common_sense**와 **pick_block_shape**가 가장 취약.

## 3. 교차 패턴 1 — 왼팔(Left arm) 파지 성공률이 오른팔의 절반 수준

같은 오브젝트 세트를 좌/우팔로만 바꿔 지시했을 때의 가중평균 (Follow / PickUp):

| 태스크 | Left Follow | Left PickUp | Right Follow | Right PickUp | Left/Right PickUp 비율 |
|---|---:|---:|---:|---:|---:|
| pick_object_type | 0.938 | **0.373** | 0.993 | 0.674 | 0.55x |
| pick_block_shape | 0.618 | **0.224** | 0.706 | 0.478 | 0.47x |
| pick_block_number | 0.885 | 0.665 | 0.936 | 0.707 | 0.94x |
| pick_follow_logic_or | 0.860 | 0.698 | 0.961 | 0.737 | 0.95x |

**해석**: 왼팔 열세는 모든 태스크에 균일하게 나타나지 않는다. 숫자 블록(`pick_block_number`)이나 OR-논리(`pick_follow_logic_or`)처럼 비교적 표준적인 형태의 물체에서는 좌/우 격차가 5~6%p로 작다. 반면 **작거나 불규칙한 실물 오브젝트(펜, 장난감차, 루빅스 큐브)와 복잡한 기하 도형(삼각기둥, 육각기둥, 원기둥)**을 다루는 `pick_object_type`, `pick_block_shape`에서는 왼팔 파지 성공률이 오른팔의 절반 수준(0.47~0.55배)으로 떨어진다.

→ 단순 "왼팔 하드웨어/캘리브레이션 문제"라기보다 **"왼팔 + 작거나 비정형 형태 물체" 조합에서만 커지는 문제**로 보임. 훈련 데이터에서 이 조합의 시연이 부족했거나, 왼쪽 손목 카메라(`gripper_l_base_link/Left_Camera`) 기반 비주얼 서보잉 정밀도가 떨어질 가능성.

### 세부 사례 — `pick_block_shape`의 삼각기둥/육각기둥
- Left 삼각기둥: Follow=0.89 (접근은 잘 됨) → **PickUp=0.00 (100% 파지 실패)**
- Right 삼각기둥: Follow=1.00 → PickUp=0.47
- Left 육각기둥: Follow=0.12 (접근 단계부터 실패) → PickUp=0.10
- Right 육각기둥: Follow=0.50 → PickUp=0.28

삼각기둥은 "접근은 되는데 절대 못 집음"(정책의 파지 동작 문제), 육각기둥은 "접근 자체가 잘 안 됨"(인식/도달 문제)으로 실패 양상이 다르다 — 하나의 원인으로 묶이지 않을 수 있음.

### 프레임 단위 케이스 스터디 — `pick_object_type` "Left arm picks up toys" 실패 에피소드
`recording_0318_20260702_070319_867910_af37a65b` (task: 왼팔로 루빅스 큐브 집기, 실패) 의 손목 카메라(`hand_left.webm`) 프레임을 열어본 결과:
1. 영상 시작(0%): 그리퍼가 이미 큐브 바로 옆까지 접근해 있음
2. ~10% 지점: 그리퍼가 큐브에서 위로/뒤로 물러남
3. 30%~98%(끝까지): 그리퍼가 완전히 정지, 계속 열린 채로 아무 동작도 하지 않음

→ "잡으려다 실패"가 아니라 **물체 직전까지 왔다가 후퇴한 뒤 정책이 멈춰버리는(freeze) 패턴**. 파지 모션 자체를 커밋하지 못하고 있음을 시사 — IK/reachability 한계라기보다 정책의 행동 결정(action decoding) 문제일 가능성이 높음. (표본 1개, 일반화 검증 필요)

## 4. 교차 패턴 2 — Common-sense(간접 지시) 이해 실패는 "파지"가 아니라 "인식/접근" 단계에서부터 발생

`pick_common_sense`는 Follow 단계 평균이 0.428로, 다른 태스크(0.66~0.96) 대비 훨씬 낮다. 즉 로봇이 **올바른 물체 쪽으로 다가가지도 못하는** 경우가 많다 — 파지 동작 능력이 아니라 자연어→타겟 오브젝트 매핑(언어적 그라운딩) 실패로 보임.

가장 취약한 지시문 (score=Follow, e2e=PickUp 성공률):

| 지시문 | Follow | PickUp | n |
|---|---:|---:|---:|
| I need to remember something, give me a marking tool | 0.00 | 0.00 | 20 |
| I need something to relieve boredom | 0.00 | 0.00 | 20 |
| Choose a gift for someone who likes puzzles | 0.00 | 0.00 | 30 |
| I'm going to take notes at a meeting, prepare tools for me | 0.00 | 0.00 | 50 |
| I need something that can be held and played with | 0.00 | 0.00 | 20 |
| I want to make fruit salad, give me a fruit | 0.05 | 0.05 | 20 |
| I need something to exercise brain power | 0.20 | 0.00 | 50 |

반대로 직접적인 지시는 잘 처리:

| 지시문 | Follow | PickUp | n |
|---|---:|---:|---:|
| Get me an apple-like fruit | 1.00 | 1.00 | 20 |
| Give me a bottle of juice | 0.65 | 0.575 | 40 |

→ 은유적/간접적 표현("brain power", "relieve boredom", "puzzles")일수록 완전히 실패하고, 준-직접적 표현("apple-like fruit")은 잘 처리됨. **언어 그라운딩이 얕은 어휘 매칭에 의존**하고 있을 가능성.

## 5. 교차 패턴 3 — OR-logic 중 "paper cup or stationery / stationery or beverage" 조합만 국소적으로 완전 실패

`pick_follow_logic_or`는 전반적으로 강함(PickUp 0.72)이지만 예외적으로 0.00으로 무너지는 조합이 있음:

| 지시문 | Follow | PickUp | n |
|---|---:|---:|---:|
| Left arm picks up the paper cup or stationery on the table | 0.00 | 0.00 | 20 |
| Right arm picks up the paper cup or stationery on the table | 1.00 | 0.00 | 30 |
| Right arm picks up the stationery or beverage on the table | 1.00 | 0.00 | 30 |

다른 OR 조합(예: "fruit or beverage", "red block or stationery")은 대부분 0.9~1.0인데, 유독 **"paper cup"이 포함된 조합**에서만 붕괴 — paper cup 오브젝트 자체의 인식/파지 난이도이거나 해당 asset의 물성(가볍고 얇은 재질) 문제일 가능성. 소표본(n=20~30)이라 재현성 확인 필요.

## 6. 종합 및 우선순위

| 원인 축 | 근거 | 확신도 |
|---|---|---|
| 왼팔 + 작은/비정형 오브젝트 파지 실패 | pick_object_type, pick_block_shape에서 일관된 2배 격차, n=500~1000 | 중~높음 |
| 파지 직전 "freeze" 행동 패턴 | 프레임 케이스 스터디 1건 | 낮음 (검증 필요) |
| 간접/은유적 언어 지시 그라운딩 실패 | pick_common_sense Follow 0.428, 다수 지시문 0.00 | 높음 |
| paper cup 관련 오브젝트 파지/인식 취약 | pick_follow_logic_or 국소 0.00 사례 3건 | 낮음 (소표본) |

**포트폴리오 스토리로 가장 강한 축은 "왼팔 + 작은/비정형 오브젝트" 패턴** — 두 개의 독립된 태스크(1500 에피소드)에서 재현되고, 수치가 명확하며, 좌/우 대칭 비교라는 깔끔한 ablation 구조를 가진다.

## 7. 다음 단계 (개선 작업 계획, 초안)

1. **가설 재검증**: `pick_object_type`/`pick_block_shape`의 왼팔 실패 에피소드 영상을 5~10개 더 샘플링해 "freeze" 패턴이 얼마나 일반적인지 정량화 (case study를 n=1 → 통계로 격상)
2. **원인 좁히기**: 왼팔 실패가 (a) 정책의 행동 생성 문제인지 (b) 왼쪽 손목 카메라 캘리브레이션/화질 문제인지 구분 — 실패 에피소드에서 `Left_Camera` 프레임 품질(블러, FOV, occlusion) 직접 비교
3. **데이터 수집**: 연구 계획에 있던 cuRobo 자동 데이터 수집 파이프라인으로 "왼팔 + 작은/비정형 오브젝트" 조합 데모를 추가 수집
4. **Fine-tuning**: 수집한 데이터로 `instruction_and_robust_pi05` 체크포인트를 추가 학습
5. **재평가**: 동일 5개 태스크로 재평가해 이 리포트의 수치(특히 좌/우 PickUp 비율)와 직접 비교 → before/after 스토리 완성

## 8. 재현 방법 (기록용)
- 원본 로그: `benchmark_results/if_all/batch_console.log` (config: `source/geniesim_benchmark/src/geniesim_benchmark/config/g2op_if_*.yaml`)
- 추론 서버: `cd ACoT-VLA && bash scripts/server.sh 0 8999 PI05_GENIE_SIM_INSTRUCTION_AND_ROBUST`
