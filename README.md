# review-harness

이미 구축된 AI 작업 환경(harness)을 진단·최적화하는 에이전트 스킬.
**Claude Code**와 **Codex CLI** 두 런타임을 각각 전용 변형판으로 지원한다.

[design-harness](https://github.com/jsj9346/design-harness)가 **Gap을 채운다면**,
이 스킬은 **Gap이 사라졌거나 애초에 없었던 항목을 걷어낸다.** 같은 차집합을
반대 방향으로 돈다:

```
기존 harness − 모델이 이미 함 − 런타임이 이미 함 − 다른 층이 이미 맡음
             − 실측 근거 있는 항목  =  제거 대상
```

하네스는 만들 때가 아니라 **모델이 좋아질 때** 낡는다. 작년에 정당했던 규칙이
올해는 모델 능력과 중복이 된다 — 리뷰는 1회성 청소가 아니라 모델 교체 주기에
맞춘 정기 작업이다.

## 핵심 구분 — 능력 중복 ≠ 보장 중복

"모델이 이미 잘하는 영역을 하네싱하는 것도 중복"이 이 스킬의 출발점이지만,
예외가 하나 있다. 모델이 95% 해내는 일과 100% 보장되어야 하는 비가역·치명
Invariant는 다르다. 판정 질문은 **"모델이 이걸 하는가"가 아니라 "모델이 이걸
안 한 경우가 한 번 있을 때 복구 가능한가"**다.

- 복구 가능 → **능력 중복, 제거**
- 복구 불가 → **강제층 유지.** 단 같은 규칙이 지침층에도 있으면 지침층을 제거

이 구분이 없으면 review-harness는 안전장치를 "중복"이라며 삭제하는 도구가 된다.

## 판정 근거 — 하이브리드 2단

| 단계 | 방법 | 대상 |
|---|---|---|
| 1차 | [references/model-baseline.md](references/model-baseline.md) 대조 | 전 항목 |
| 2차 | 실측 ablation (규칙을 뺀 채 대표 태스크 K회 실행) | Tier B·고유 맥락·강제층 항목 |

모든 능력 중복 판정에는 `[verified: <모델>-<버전> / <YYYY-MM>]`을 각인한다.
버전 없는 "모델이 잘함" 판정은 무효이며, **더 작은 모델로의 다운그레이드는
표식 항목 전체를 재검토 대상으로 만든다.**

baseline은 SKILL.md와 분리된 갱신 대상 파일이다 — 모델이 바뀌어도 절차는
안 낡는다.

## 진단 렌즈

**D — 중복**: D1 모델 · D2 런타임 · D3 층간 · D4 자산 · D5 문서
**S — 잔재**: S1 Dead · S2 Drift · S3 Unused · S4 Speculative · S5 Over-specified

조치: `REMOVE` · `MERGE` · `MOVE` · `PROMOTE` · `KEEP` · `WATCH`

`PROMOTE`가 있는 이유 — 리뷰는 줄이기만 하는 작업이 아니다. 비가역 Invariant가
지침층에만 있으면 그건 **과소 하네스**이며, 제거 목록과 같은 비중으로 보고한다.

## 구조

```
review-harness/
├── SKILL.md                      # Claude Code 전용
├── references/
│   └── model-baseline.md         # 모델 능력 기준표 (갱신 대상, 버전 스탬프)
└── codex/
    └── SKILL.md                  # Codex CLI 전용
```

두 변형판은 원칙·절차가 같고 **강제층의 형태만 다르다**: Claude Code는
hooks·permissions, Codex는 `approval_policy`·`sandbox_mode`·`rules`. 이 차이가
D2(런타임 중복) 판정 기준을 바꾸므로 런타임에 맞는 파일을 쓸 것.

## 절차 요약

1. **하네스 인벤토리** — 파일이 아니라 **규칙 한 줄 = 항목 하나**로 분해
   (파일 단위로 보면 "이 파일은 유용함"에서 멈추고 그 안의 죽은 규칙을 못 본다)
2. **상시 비용 측정** — 상시 로드(전역 지침 + 스킬 description)와 조건부
   로드(스킬 본문)를 **섞어 세지 않는다**. "스킬이 15개나 된다"는 그 자체로
   비용이 아니다
3. **판정** — baseline 1차 → ablation 2차 → 항목 간 중복(D3·D4)·잔재 교차 검사
4. **Invariant 보호 게이트** — 이 게이트를 통과하지 않은 REMOVE는 적용 금지
5. **최적화안 제시** — 원장 표 + before/after 수치 + WATCH·미판정 목록, 승인 대기
6. **적용 → 회귀 검증** — 제거는 가설이다. 대표 태스크로 확인하고 실패하면
   롤백 후 그 관측을 KEEP 근거로 기록

**0개 제거로 끝나는 리뷰는 하네스가 건강하다는 뜻이지 리뷰 실패가 아니다.**

## 설치 — Claude Code

```bash
git clone https://github.com/jsj9346/review-harness ~/.claude/skills/review-harness
```

사용: `/review-harness` 또는 "하네스 검토해줘", "규칙이 너무 많은 것 같아" 등
자연어 호출.

## 설치 — Codex CLI

```bash
git clone https://github.com/jsj9346/review-harness /tmp/review-harness
mkdir -p ~/.codex/skills/review-harness
cp -r /tmp/review-harness/codex/* ~/.codex/skills/review-harness/
cp -r /tmp/review-harness/references ~/.codex/skills/review-harness/
```

사용: `/skills`에서 선택, `$review-harness`로 명시 호출, 또는 작업 설명이
description과 매칭되면 자동 호출.

## 관련 스킬

- **[design-harness](https://github.com/jsj9346/design-harness)** — 하네스를 **설계·생성**한다. 리뷰 중 빈 Gap이 발견되면
  여기로 넘긴다 (리뷰가 설계로 번지면 리뷰 결과를 검증할 수 없다).
- **[design-graph](https://github.com/jsj9346/design-graph)** — 여러 Node를 **연결**하는 그래프를 설계한다.

자세한 내용은 [SKILL.md](SKILL.md) / [codex/SKILL.md](codex/SKILL.md).

## License

[MIT](LICENSE)
