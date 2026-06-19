# Vox2Vocal Git Policy

문서 버전: `v2026.06.19`
적용일: `2026-06-19`
상태: `Active`

이 문서는 Vox2Vocal 독립 repository의 Git identity, commit convention, branch/PR workflow, local hook, GitHub Actions CI, main merge 운영 절차를 정의한다.

Vox2Vocal workspace는 monorepo가 아니라 여러 독립 Git repository를 한 workspace에 둔 구조다. 따라서 정책 파일은 각 repository에 동일하게 두되, 일반 코드 repo와 문서 repo는 commit message 세부 규칙을 분리한다.

## Applied Repositories

일반 코드 repo:

```txt
engine-audio-ingest
engine-voice-analysis
engine-voice-pitch
vox2vocal-agent-skills
vox2vocal-api-gateway
vox2vocal-app
vox2vocal-bff-server
vox2vocal-design-kit
vox2vocal-infra
vox2vocal-user-service
vox2vocal-worker
```

문서 repo:

```txt
vox2vocal-docs
```

기존 repository의 `main` history는 2026-06-13에 한 번 rewrite되어 author, committer, commit message 정책을 모두 만족하도록 정리되었다. 이후 신규 위반은 rewrite로 사후 보정하지 않고 hook, CI, branch/PR workflow에서 차단한다.

## Required Identity

모든 커밋의 author와 committer는 아래 값과 정확히 일치해야 한다.

```txt
gitbyul <gitbyul@gmail.com>
```

다른 개인 계정, bot 계정, 로컬 머신 이메일, GitHub noreply 이메일은 허용하지 않는다.

각 repository에서 다음 명령으로 local Git 설정과 hook 경로를 고정한다.

```bash
scripts/install-git-policy-hooks.sh
```

이 명령은 다음 설정을 적용한다.

```bash
git config user.name "gitbyul"
git config user.email "gitbyul@gmail.com"
git config core.hooksPath .githooks
```

## Local Agent Workflow

다른 에이전트가 로컬에서 작업하더라도 Git commit identity는 항상 `gitbyul <gitbyul@gmail.com>`로 고정한다. 에이전트별 이름, 로컬 머신 사용자, GitHub noreply, bot identity는 commit author 또는 committer로 남기지 않는다.

에이전트는 작업을 시작하기 전에 대상 repository 안에서 아래 명령을 실행한다.

```bash
scripts/install-git-policy-hooks.sh
scripts/validate-git-policy.sh --check-config
```

설정이 수동으로 필요한 경우 아래 값을 repository-local config로 지정한다.

```bash
git config user.name "gitbyul"
git config user.email "gitbyul@gmail.com"
git config core.hooksPath .githooks
```

에이전트 작업 규칙:

- `git config --global`에 의존하지 않고 각 repository의 local config를 확인한다.
- `--author`, `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`, `GIT_COMMITTER_EMAIL`로 다른 identity를 주입하지 않는다.
- commit 전 `scripts/validate-git-policy.sh --check-config`를 실행한다.
- commit 후 `scripts/validate-git-policy.sh --commit HEAD` 또는 `scripts/validate-git-policy.sh --range origin/main HEAD`를 실행한다.
- PR 생성 전 push 대상 branch가 정책 형식을 만족하는지 확인한다.
- GitHub UI에서 자기 PR에 `Approve`를 누르지 않는다. GitHub는 PR 작성자가 자기 PR을 approve하는 것을 허용하지 않으며, `Can not approve your own pull request`는 정책 위반이 아니라 GitHub 기본 동작이다.
- Required approvals는 `0`으로 운영한다. 이 정책에서 강제하는 것은 PR 존재, Git policy check, 고정된 author/committer이지 다른 사람의 승인 필수가 아니다.

## Commit Message

### 일반 코드 Repo

일반 코드 repo의 커밋 메시지는 항상 ticket을 포함한다.

```txt
type(scope): [TICKET] 한글 제목

- 한글 bullet body
```

예시:

```txt
feat(auth): [V2V-123] 로그인 토큰 갱신 추가

- refresh token 회전 흐름을 gateway와 user-service에 연결합니다.
- 만료 token 재사용 감지 테스트를 추가합니다.
```

ticket 형식은 대문자 project key와 숫자를 사용한다.

```txt
[V2V-123]
```

### 문서 Repo

문서 repo는 ticket을 선택으로 두고, 문서 버전 또는 문서 변경 단위를 bracket에 표시한다.

```txt
type(scope): [VERSION] 한글 제목

- 한글 bullet body
```

예시:

```txt
docs(policy): [v2026.06.16] Git 정책 운영 방식 갱신

- 코드 repo와 문서 repo의 커밋 메시지 규칙을 분리합니다.
- main 병합 push 예외 조건을 명시합니다.
```

문서 버전업을 진행하기 전에는 현재까지의 문서 수정 내용을 반드시 별도 커밋으로 먼저 정리한다. 버전업 커밋에는 버전 번호, 날짜, changelog, 해당 버전의 최소 필요 수정만 포함하고 이전 작업 수정과 섞지 않는다.

### 공통 규칙

허용 type:

```txt
feat
fix
docs
chore
refactor
test
ci
```

필수 규칙:

- `type`은 허용 type 중 하나여야 한다.
- `scope`는 필수이며 영어 소문자 kebab-case로 작성한다.
- 제목에는 한글이 포함되어야 한다.
- 제목과 본문 사이에는 빈 줄을 정확히 한 줄 둔다.
- 본문은 필수다.
- 본문의 모든 줄은 `- `로 시작한다.
- 본문에는 한글이 포함되어야 한다.
- 본문 bullet 사이에 빈 줄을 넣지 않는다.

## Branch And PR Workflow

일반 코드 repo는 항상 별도 작업 branch에서 커밋하고 PR을 생성한다.

branch 형식:

```txt
type/TICKET-short-summary
```

예시:

```txt
feat/V2V-123-auth-refresh
fix/V2V-204-token-case-mapping
chore/V2V-000-git-policy-pr-flow
```

일반 코드 repo에서 `main` 직접 작업은 금지한다.

- `commit-msg` hook은 `main`에서 새 커밋을 만들지 못하게 차단한다.
- `pre-push` hook은 `main` push를 기본 차단한다.
- PR은 GitHub에서 생성하고 `git-policy` check를 통과해야 한다.
- GitHub UI merge 버튼은 사용하지 않는다.

문서 repo는 기존 운영처럼 `main`에서 직접 문서 커밋을 만들 수 있다. 단, commit message와 body 규칙, 문서 버전업 전 선행 커밋 규칙은 반드시 지킨다.

## Main Merge Policy

GitHub UI가 생성하는 squash commit 또는 merge commit은 committer가 GitHub 계정으로 기록될 수 있다. 최종 `main` history에서도 committer를 `gitbyul <gitbyul@gmail.com>`로만 유지하려면 GitHub UI에서 생성하는 merge commit을 사용하지 않는다.

일반 코드 repo의 main 반영은 다음 방식으로 운영한다.

1. ticket branch에서 정책을 통과한 커밋을 만든다.
2. ticket branch를 push하고 PR을 생성한다.
3. PR에서 `git-policy` check를 통과시킨다.
4. 필요한 코드 리뷰나 agent review가 있으면 완료한다.
5. GitHub UI merge 버튼을 누르지 않는다.
6. `gitbyul` 로컬 환경에서 PR branch를 `main`에 fast-forward로 반영한다.
7. 승인된 PR 번호를 명시한 뒤 `main`을 push한다.

권장 명령:

```bash
git checkout main
git pull --ff-only origin main
git fetch origin <branch-name>
git merge --ff-only origin/<branch-name>
GIT_POLICY_ALLOW_MAIN_MERGE_PUSH=1 GIT_POLICY_PR_NUMBER=<number> git push origin main
```

이 예외는 `main`에서 바로 개발/커밋/push하는 것을 허용한다는 뜻이 아니다. 이미 PR과 CI를 통과한 branch를 `gitbyul` 계정으로 최종 반영하기 위한 병합 절차다.

GitHub UI merge, `gh pr merge --merge`, `gh pr merge --squash`, `gh pr merge --rebase`는 사용하지 않는다. 이러한 방식은 GitHub가 merge/squash/rebase commit을 만들거나 committer를 GitHub 계정으로 기록할 수 있으므로 최종 `main` history의 committer 고정 정책과 충돌할 수 있다.

PR 작성자와 merge 수행자가 모두 `gitbyul`이어도 허용한다. 같은 작성자가 PR을 만들고 merge하는 경우에도 GitHub에서 자기 PR에 approve review를 남길 필요는 없다.

## Policy Files

각 repository에는 다음 파일을 둔다.

```txt
scripts/validate-git-policy.sh
scripts/install-git-policy-hooks.sh
.githooks/commit-msg
.githooks/pre-push
.github/workflows/git-policy.yml
```

`scripts/validate-git-policy.sh` 사용법:

```bash
scripts/validate-git-policy.sh --check-config
scripts/validate-git-policy.sh --message-file <path>
scripts/validate-git-policy.sh --branch-name <branch-name>
scripts/validate-git-policy.sh --push-ref <local-ref> <remote-ref>
scripts/validate-git-policy.sh --commit <sha>
scripts/validate-git-policy.sh --range <base-sha> <head-sha>
```

전체 현재 history 검증:

```bash
scripts/validate-git-policy.sh --range 0000000000000000000000000000000000000000 HEAD
```

## Local Hook

각 repository에는 다음 hook을 둔다.

- `.githooks/commit-msg`: local Git identity, 현재 branch, commit message 검사
- `.githooks/pre-push`: push branch, main push 예외 조건, push 대상 커밋의 author, committer, commit message 검사

hook은 개발자 실수를 빠르게 막기 위한 장치다. `--no-verify`로 우회될 수 있으므로 최종 강제는 GitHub Actions required check와 GitHub ruleset에서 함께 수행한다.

hook 설치 또는 재설치:

```bash
scripts/install-git-policy-hooks.sh
```

hook 동작 테스트:

```bash
printf 'feat(auth): [V2V-123] 로그인 훅 검증\n\n- commit-msg hook 정상 동작을 검증합니다.\n' > /tmp/valid-commit-msg
.githooks/commit-msg /tmp/valid-commit-msg

printf 'feat(auth): 로그인 훅 검증\n\n- ticket 누락 메시지는 실패해야 합니다.\n' > /tmp/invalid-commit-msg
.githooks/commit-msg /tmp/invalid-commit-msg
```

두 번째 명령은 실패해야 정상이다.

## GitHub Actions

각 repository에는 `.github/workflows/git-policy.yml`을 둔다.

Workflow 이름:

```txt
git-policy
```

Trigger:

- `pull_request`
- `workflow_dispatch`

검사 항목:

- PR branch name
- author name
- author email
- committer name
- committer email
- commit message header
- commit message body

PR 검사 방식:

- branch 이름은 PR head branch를 검사한다.
- commit range 검사는 PR에 새로 포함되는 commit만 대상으로 한다.
- PR base commit에 `scripts/validate-git-policy.sh`가 있으면 base의 validator를 사용한다.
- PR base commit에 validator가 없으면 PR head의 validator를 사용한다.
- validator는 `bash`로 실행한다.

직접 실행 예:

```bash
base_sha=<pull-request-base-sha>
head_sha=<pull-request-head-sha>
bash scripts/validate-git-policy.sh --branch-name "$branch_name"
bash scripts/validate-git-policy.sh --range "$base_sha" "$head_sha"
```

## GitHub Ruleset

일반 코드 repo의 `main` branch에는 다음 보호를 적용한다.

필수 설정:

- Require status checks to pass before merging
- Required check: `git-policy`
- Block force pushes
- Restrict deletions
- Restrict updates to `gitbyul` only
- Required approvals: `0`

PR 필수 정책은 운영상 필수다. 단, GitHub UI merge를 사용하지 않고 로컬 fast-forward merge를 사용하려면 `Require a pull request before merging`을 무조건 non-bypass로 켜면 의도한 `gitbyul` 로컬 병합 push도 차단될 수 있다.

따라서 GitHub ruleset은 아래 둘 중 하나로 운영한다.

- ruleset에서 `gitbyul`의 제한된 bypass를 허용하고 `Require a pull request before merging`을 켠다.
- bypass 구성이 불가능한 경우 `Require status checks`, `Restrict updates to gitbyul`, local `pre-push`의 main push 예외 변수, 운영 절차로 PR 필수를 강제한다.

문서 repo는 기존 운영처럼 직접 문서 커밋을 허용할 수 있다. 다만 force push 금지, deletion 금지, identity/message validator는 유지한다.

동일 작성자 PR을 허용하려면 아래 review 설정은 끈다.

- Required approvals `1` 이상
- Require review from specific teams
- Require review from Code Owners
- Require approval of the most recent reviewable push

위 설정을 켜면 `gitbyul` 혼자 PR을 만들고 병합하는 운영과 충돌한다.

권장 설정:

- Require linear history
- Require conversation resolution before merging
- Require branches to be up to date before merging

서명 커밋 정책:

- 현재 rewritten history는 signing key 없이 생성되어 unsigned 상태다.
- `Require signed commits`는 GPG 또는 SSH signing key를 먼저 구성한 뒤 새 커밋부터 적용한다.
- 기존 history까지 signed 상태로 만들려면 signing key 구성 후 별도 history rewrite가 필요하므로 기본 운영 절차로 삼지 않는다.

## History Rewrite

기존 history 전체 rewrite는 예외 상황에서만 수행한다. 2026-06-13에 한 차례 기존 위반 history를 정리했으므로, 이후 정책 위반은 rewrite로 사후 보정하지 않고 hook, CI, ruleset에서 차단한다.

부득이하게 rewrite가 필요한 경우 필수 절차:

1. `git fetch --all --prune`으로 원격 최신 상태를 확인한다.
2. repo 내부 branch가 아니라 외부 bundle로 백업한다.
3. rewrite 전후 `HEAD^{tree}`를 비교해 파일 내용이 바뀌지 않았는지 확인한다.
4. `refs/original/*`를 정리한다.
5. `scripts/validate-git-policy.sh --range 0000000000000000000000000000000000000000 HEAD`로 전체 history를 검증한다.
6. 관련 문서에 남은 old commit hash 참조를 새 hash로 갱신한다.
7. 원격 반영은 `git push --force-with-lease origin main`만 사용한다.

## Force Push

2026-06-13 history rewrite를 원격 `main`에 반영하기 위해 한 차례 `--force-with-lease` push를 수행했다. 이후 `main` force push는 금지한다.

force push 후 다른 clone은 일반 `git pull`로 자연스럽게 따라오지 못할 수 있다. 작업자는 다음 중 하나를 선택한다.

```bash
git fetch origin
git reset --hard origin/main
```

또는 repository를 새로 clone한다.
