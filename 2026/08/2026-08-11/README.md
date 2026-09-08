# 2026-08-11 개발 기록

모노레포(Vue 3 + TypeScript + Nx)의 **Git pre-commit 훅을 husky + lint-staged에서 lefthook으로 교체**하며 배운 점을 정리한다. 단순 도구 교체가 아니라, "커밋할 때마다 왜 이렇게 느린가"를 파고들면서 **type-checked ESLint의 비용 구조**와 **병렬화의 함정(OOM)**을 실측으로 확인한 하루였다.

---

## 1. 느림의 원인은 "파일 수"가 아니라 "TS 프로그램 재빌드"였다

lint-staged는 staged 파일을 **파일별로** linter에 넘긴다. 파일이 적으면 빠를 것 같지만, type-checked ESLint에서는 정반대였다.

### 배운 점
- 타입 정보를 쓰는 ESLint(`@typescript-eslint`의 type-aware 규칙)는 실행할 때마다 **TS 프로그램(~2.5GB)을 통째로 빌드**한다. 이 비용은 **대상 파일 수와 거의 무관**하다.
- 그래서 파일별로 돌리면 **파일 수만큼 TS 프로그램을 재빌드**한다. 7개 파일 = 재빌드 7번 = 약 47초.
- 반대로 staged 파일을 **한 프로세스에 묶어(batch)** 한 번에 lint하면 TS 프로그램을 **딱 한 번만** 빌드 → 약 20초.

```yaml
# lefthook.yml — staged 파일을 한 프로세스에서 일괄 lint
eslint:
  glob: '{apps,libs,packages}/**/*.{js,ts,vue}'
  run: pnpm exec eslint --fix --cache {staged_files}
  stage_fixed: true
```

### 교훈
> "파일 하나씩 처리 = 빠름"은 **고정비용이 없을 때만** 성립한다. type-checked lint처럼 실행마다 큰 고정비용(TS 프로그램)이 있으면, **묶어서 한 번**이 압도적으로 유리하다.

---

## 2. `stage_fixed`로 자동 수정분을 다시 스테이징

`--fix`가 파일을 고쳐도, 그 변경은 **working tree에만** 반영된다. 커밋에 포함되려면 다시 `git add` 해야 한다.

### 배운 점
- lefthook은 `stage_fixed: true` 한 줄로 **훅이 고친 파일을 자동으로 재스테이징**한다. lint-staged가 내부적으로 해주던 걸 명시적 옵션으로 제공.
- `--cache`를 붙이면 **변경 없는 파일은 건너뛰어** 재커밋 속도가 개선된다. (`.eslintcache`는 `.gitignore`에 추가)

---

## 3. 병렬화는 공짜가 아니다 — 동시 프로세스 × TS 메모리 = OOM

"묶어서 20초"도 느리니 병렬로 더 줄여볼까 했는데, 여기서 함정에 빠졌다.

### 배운 점
- ESLint를 병렬로 여러 프로세스 띄우면 **각 프로세스가 자기 TS 프로그램(~2.5GB)을 따로** 올린다. 동시 실행 N개 = 메모리 N배.
- 커밋 배치의 피크 메모리를 실측하니 ~3.4GB. 힙을 안 올리면 `JavaScript heap out of memory`로 죽는다.
- 그래서 운영 훅은 **직렬(`parallel: false`) + `--cache` + 힙 상향(`--max-old-space-size=6144`)** 으로 안전하게 두고, 병렬화는 **실험 도구로 분리**했다.

```yaml
env:
  # TS 프로그램 빌드에 메모리가 필요해 힙 상향 (batch 피크 실측 ~3.4GB)
  NODE_OPTIONS: '--max-old-space-size=6144'
```

### 교훈
> 병렬화의 이득(시간)과 비용(메모리)은 트레이드오프다. **"빨라지는가"가 아니라 "죽지 않는가"를 먼저** 봐야 한다. 최적화는 추측이 아니라 **실측**으로.

---

## 4. 운영 훅과 실험 훅을 분리 — 안전을 기본값으로

병렬/힙 조합을 검증하려면 일부러 OOM을 재현해봐야 하는데, 그걸 **운영 커밋 흐름에 섞으면 안 된다.**

### 배운 점
- lefthook의 **명명된 그룹**을 활용해, 운영 `pre-commit`과 별개로 `bench-parallel` 그룹을 두었다. 이 그룹은 커밋에 자동 연결되지 않고 `pnpm exec lefthook run bench-parallel`로만 실행된다.
- 벤치 스크립트(`tools/lefthook-bench/bench.sh [파일수] [동시실행수] [힙MB]`)로 **직렬 vs 병렬의 시간·피크 메모리·OOM 여부**를 실측한다. `동시실행수↑ + 힙MB↓`로 밀면 OOM이 재현된다.

### 운영 반영 판단 기준
1. 벤치로 **OOM 안 나는 (동시실행수, 힙) 조합**을 찾는다.
2. 그 조합의 **속도 이득**이 직렬 대비 유의미한지 본다.
3. 이득이 충분하면 운영 `pre-commit`을 병렬로 전환, 아니면 **직렬 + `--cache` 유지**(안전 우선).

### 교훈
> 실험은 **격리**하고, 운영의 기본값은 **가장 안전한 쪽**으로. 성능 변경은 근거(실측 로그)를 남긴 뒤에만 반영한다.

---

## 5. husky → lefthook 마이그레이션 체크리스트

실제로 손댄 지점들. 도구를 바꾸면 **연결된 설정을 빠짐없이** 옮겨야 한다.

| 항목 | before (husky) | after (lefthook) |
|------|----------------|------------------|
| 설치 스크립트 | `"prepare": "husky"` | `"prepare": "lefthook install"` |
| 훅 정의 | `.husky/pre-commit` (셸) | `lefthook.yml` (선언형) |
| staged 대상 처리 | `lint-staged.config.js` | `glob` + `{staged_files}` |
| 자동수정 재스테이징 | lint-staged 내장 | `stage_fixed: true` |
| 의존성 | `husky`, `lint-staged` | `lefthook` |
| pnpm 빌드 허용 | — | `onlyBuiltDependencies`에 `lefthook` 추가 |

### 배운 점
- `.husky/pre-commit`, `lint-staged.config.js`를 **삭제**하고, `package.json`의 `prepare` 스크립트와 devDependencies를 함께 정리해야 잔재가 안 남는다.
- lefthook은 네이티브 바이너리라 pnpm의 `onlyBuiltDependencies` 허용목록에 넣어야 postinstall이 정상 동작한다.

---

## 오늘의 한 줄
> "느리다고 무작정 병렬로 돌리면 메모리로 갚는다. type-checked lint의 진짜 비용은 파일 수가 아니라 **TS 프로그램 재빌드 횟수** — 묶어서 한 번 빌드하고, 병렬화는 실측으로 안전을 확인한 뒤에만."
