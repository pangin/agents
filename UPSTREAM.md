# UPSTREAM — agents

이 저장소는 `event-catalog/agents`의 **GitHub fork**(fork network 내부, public)입니다. 원본 기준선과 내부 수정선을 분리해 추적하며, 기계가 읽는 값은 [upstream.json](./upstream.json)에 있습니다.

| 항목                          | 값                                                                                                                                                                                              |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| upstream                      | https://github.com/event-catalog/agents (default branch `main`)                                                                                                                                 |
| fork                          | https://github.com/pangin/agents                                                                                                                                                                |
| 관계                          | GitHub fork (public). 원본 fork network에 연결되어 있어 GitHub compare/PR UI로 upstream과 비교 가능                                                                                             |
| 기준(pinned) SHA              | `ed544eb00da486c8f7ec227f92b2c947186f8012` (2026-07-07T12:15:42+01:00)                                                                                                                          |
| 현재 fetched / integrated SHA | `upstream.json` → `.sha.fetched`, `.sha.integrated` (sync workflow가 갱신)                                                                                                                      |
| owner                         | 성욱 (pangin)                                                                                                                                                                                   |
| 역할 분류                     | **excluded** — Flue 기반 LLM 에이전트를 PR에서 실행하는 composite GitHub Action(BUSL-1.1, production 사용 제한). core clean run의 직접 의존이 아니며 스택 구성요소로서 사용자 지시에 따라 fork. |
| 라이선스(root)                | BUSL-1.1 (Change Date 2030-06-25 → Apache-2.0)                                                                                                                                                  |
| 동기화 주기                   | 매주 월요일 00:00 UTC (09:00 KST) 예약 실행 + workflow_dispatch 수동 실행                                                                                                                       |
| 동기화 identity               | deploy key `upstream-sync`(secret `UPSTREAM_SYNC_SSH_KEY`). 미등록 시 GITHUB_TOKEN으로 push하며 경고 출력                                                                                       |
| Linear                        | GONG-871, GONG-872, GONG-876, GONG-875                                                                                                                                                          |

## 브랜치 모델

| 브랜치                             | 용도                                                  | 규칙                                                                                                         |
| ---------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `vendor/upstream-main`             | upstream `main`의 **exact commit**만 담는 원본 기준선 | 내부 commit·metadata 금지. 사람의 direct push 금지, sync identity만 fast-forward 갱신. force-push·삭제 금지  |
| `main`                             | 내부 수정·배포 기준선(default branch)                 | PR로만 변경, merge commit만 허용(squash/rebase 비활성), force-push·삭제 금지, required check = `Fork Verify` |
| `sync/upstream-<YYYYMMDD>-<sha12>` | upstream 변경을 `main`에 반영하는 PR 브랜치           | bot이 생성. `main`에는 bot이 직접 push하지 않음                                                              |

Git remote는 GitHub 저장소의 속성이 아니라 **checkout/CI 설정**입니다. 모든 checkout과 CI에서 다음 규칙을 적용합니다.

```bash
git clone https://github.com/pangin/agents.git && cd agents
git remote add upstream https://github.com/event-catalog/agents.git
git remote set-url --push upstream DISABLED   # upstream은 fetch-only
git fetch upstream --tags
```

## 동기화 절차 (`.github/workflows/upstream-sync.yml`)

1. `upstream/main`와 tag를 fetch (fetch-only remote).
2. `vendor/upstream-main`이 fetched commit의 조상인지 확인 → 아니면(history rewrite / non-fast-forward) **중단**, owner 검토.
3. `LICENSE*/NOTICE*/COPYING*` 변경이 있으면 **중단**. owner가 검토 후 `acknowledge_license_change=true`로 수동 재실행해야 진행.
4. `vendor/upstream-main`을 fetched commit으로 fast-forward push(sync identity). 새 tag도 push(기존 tag는 덮어쓰지 않음).
5. `main`에서 `sync/upstream-*` 브랜치를 만들고 `git merge --no-ff`(ancestry 보존). 충돌 시 **중단**, owner가 수동 통합.
6. `upstream.json`의 `sha.fetched/integrated`를 갱신해 커밋하고 `main`으로 PR 생성. PR 본문에 fetched SHA, 이전 integrated SHA, 포함된 upstream commit, 내부 patch diff stat을 기록.
7. PR은 required check 통과 후 **Create a merge commit**으로만 merge (squash/rebase 금지).

수동 실행: Actions → `Upstream Sync` → Run workflow. 실행 요약(Job Summary)에 fetched/vendor/integrated SHA, ahead/behind, 내부 patch diff가 표시됩니다.

## 상태 확인 명령

```bash
git fetch upstream --tags && git fetch origin
jq '.sha' upstream.json                                              # pinned / fetched / integrated
git rev-list --left-right --count origin/vendor/upstream-main...origin/main   # upstream-only  내부-only
git diff --stat origin/vendor/upstream-main origin/main               # 내부 patch diff
git merge-base --is-ancestor origin/vendor/upstream-main upstream/main && echo "vendor ⊂ upstream: OK"
```

## Clean build / run (runbook)

| 항목            | 값                                                                                                                                                                 |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Node            | 24.x                                                                                                                                                               |
| package manager | pnpm (lockfile: pnpm-lock.yaml)                                                                                                                                    |
| install         | `pnpm install --frozen-lockfile`                                                                                                                                   |
| build           | `pnpm run build`                                                                                                                                                   |
| test            | `n/a — upstream `pnpm test`(typecheck:evals + evals)는 pinned SHA의 clean install에서 실패함(TS2307: Cannot find module 'vite'); `pnpm run build`(tsc)까지만 검증` |
| run             | `uses: pangin/agents@main (소비 저장소 workflow) 또는 pnpm exec flue run pr-review --root . --target node --payload '{"workspace":"<checkout>"}'`                  |
| smoke           | install/build/typecheck:evals 성공, evals 실행(라이브 suite skip)                                                                                                  |
| CI              | `.github/workflows/fork-verify.yml` — PR/push(main, vendor/upstream-main)/수동 실행. jobs: metadata, vendor-integrity, build                                       |

license key와 `.env` 없이(OSS-only) 실행합니다. 빌드·타입체크·오프라인 evals만.

## 라이선스 경계

- root: BUSL-1.1 (Change Date 2030-06-25 → Apache-2.0)
- 상용/제한 경로: `저장소 전체`
- production 제한: Production Purpose 사용 금지(평가·개발·테스트만), production은 commercial license 필요
- license key 환경변수: 없음
- 라이선스 파일: `LICENSE`
- upstream에서 위 파일이 바뀌면 sync workflow가 자동 반영을 중단하고 owner 검토를 요구합니다.

## upstream workflow가 fork에서 갖는 위험

- .github/ 디렉터리가 없어 upstream workflow 없음
- action.yml 자체가 소비자 workflow에서 실행될 때 PR 코멘트·PR 생성을 수행
- pinned SHA에서 `pnpm run typecheck:evals`가 clean install 시 실패(evals/support/skill-vite-plugin.ts: Cannot find module 'vite') — upstream 문제, fork 변경과 무관

## 보호 설정 (GitHub rulesets)

- `main`: PR 필수(승인 1, repository admin은 PR merge 시 bypass 가능), required check `Fork Verify`, merge commit만 허용, force-push·삭제 금지.
- `vendor/upstream-main`: 갱신은 deploy key(sync identity)만 허용, force-push·삭제 금지(bypass 없음).
- 적용 상태는 `upstream.json` → `.protection.appliedAt` 및 GitHub Settings → Rules에서 확인합니다.
