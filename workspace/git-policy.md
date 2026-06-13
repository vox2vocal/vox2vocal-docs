# Vox2Vocal Git Policy

이 문서는 모든 Vox2Vocal 독립 repository에 동일하게 적용하는 Git 커밋 정책, local hook, GitHub Actions CI, 운영 절차를 정의한다.

Vox2Vocal workspace는 monorepo가 아니라 여러 독립 Git repository를 한 workspace에 둔 구조다. 따라서 이 정책 파일들은 각 repository에 동일하게 존재해야 한다.

## Applied Repositories

정책 적용 대상:

```txt
vox2vocal-api-gateway
vox2vocal-docs
engine-voice-analysis
vox2vocal-worker
vox2vocal-app
engine-audio-ingest
vox2vocal-user-service
engine-voice-pitch
vox2vocal-infra
vox2vocal-bff-server
```

각 repository의 `main` history는 2026-06-13에 한 번 rewrite되어 author, committer, commit message 정책을 모두 만족하도록 정리되었다. 이후에는 hook, CI, GitHub ruleset으로 신규 위반 커밋을 차단한다.

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

설정 확인:

```bash
git config --get user.name
git config --get user.email
git config --get core.hooksPath
```

## Required Commit Message

커밋 메시지는 항상 아래 형식을 사용한다.

```txt
type(scope): 한글 제목

- 한글 bullet body
```

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

예시:

```txt
chore(infra): 데이터 저장소 이미지 버전 고정

- PostgreSQL 이미지를 postgres:17.10-alpine으로 고정
- Redis 이미지를 redis:7.2.14-alpine3.21로 고정
- infra README에 데이터 저장소 이미지 기준 추가
```

## Policy Files

각 repository에는 다음 파일을 둔다.

```txt
scripts/validate-git-policy.sh
scripts/install-git-policy-hooks.sh
.githooks/commit-msg
.githooks/pre-push
.github/workflows/git-policy.yml
```

`scripts/validate-git-policy.sh`는 공통 validator다.

사용법:

```bash
scripts/validate-git-policy.sh --check-config
scripts/validate-git-policy.sh --message-file <path>
scripts/validate-git-policy.sh --commit <sha>
scripts/validate-git-policy.sh --range <base-sha> <head-sha>
```

전체 현재 history 검증:

```bash
scripts/validate-git-policy.sh --range 0000000000000000000000000000000000000000 HEAD
```

## Local Hook

각 repository에는 다음 hook을 둔다.

- `.githooks/commit-msg`: local Git identity와 커밋 메시지 검사
- `.githooks/pre-push`: push 대상 커밋의 author, committer, 커밋 메시지 검사

hook은 개발자 실수를 빠르게 막기 위한 장치다. `--no-verify`로 우회될 수 있으므로 최종 강제는 GitHub Actions required check와 GitHub ruleset에서 수행한다.

hook 설치 또는 재설치:

```bash
scripts/install-git-policy-hooks.sh
```

hook 동작 테스트:

```bash
printf 'chore(test): 한글 훅 검증\n\n- commit-msg hook 정상 동작을 검증합니다.\n' > /tmp/valid-commit-msg
.githooks/commit-msg /tmp/valid-commit-msg

printf 'chore: invalid\n' > /tmp/invalid-commit-msg
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

- author name
- author email
- committer name
- committer email
- commit message header
- commit message body

PR 검사 방식:

- PR base commit에 `scripts/validate-git-policy.sh`가 있으면 base의 validator를 사용한다.
- PR base commit에 validator가 없으면 PR head의 validator를 사용한다.
- 검사는 PR에 새로 포함되는 commit range만 대상으로 한다.
- validator는 `bash`로 실행한다. `/tmp/validate-git-policy.sh`를 직접 실행하지 않는다.

직접 실행 예:

```bash
base_sha=<pull-request-base-sha>
head_sha=<pull-request-head-sha>
bash scripts/validate-git-policy.sh --range "$base_sha" "$head_sha"
```

## GitHub Ruleset

각 repository의 `main` branch에 동일한 ruleset을 적용한다.

필수 설정:

- Require a pull request before merging
- Require status checks to pass before merging
- Required check: `git-policy`
- Block force pushes
- Restrict deletions
- Restrict updates to `gitbyul` only
- Do not allow bypassing rules

권장 설정:

- Require linear history
- Require conversation resolution before merging
- Require branches to be up to date before merging

서명 커밋 정책:

- 현재 rewritten history는 signing key 없이 생성되어 unsigned 상태다.
- `Require signed commits`는 GPG 또는 SSH signing key를 먼저 구성한 뒤 새 커밋부터 적용한다.
- 기존 history까지 signed 상태로 만들려면 signing key 구성 후 별도 history rewrite가 필요하므로 기본 운영 절차로 삼지 않는다.

## Merge Policy

GitHub UI가 생성하는 squash commit 또는 merge commit은 committer가 GitHub 계정으로 기록될 수 있다. 최종 `main` history에서도 committer를 `gitbyul <gitbyul@gmail.com>`로만 유지하려면 GitHub UI에서 생성하는 merge commit을 사용하지 않는다.

가장 엄격한 방식은 다음 흐름이다.

1. feature branch에서 정책을 통과한 커밋을 만든다.
2. PR에서 `git-policy` check를 required로 통과시킨다.
3. `main`에는 ruleset을 적용해서 무단 push와 force push를 막는다.
4. release 또는 merge 권한은 `gitbyul` 계정으로만 운영한다.
5. 최종 반영은 정책을 만족하는 commit을 fast-forward 또는 trusted automation으로 반영한다.

GitHub UI merge를 허용하는 경우, PR 커밋의 author/committer는 강제할 수 있지만 GitHub가 최종 merge commit committer로 남을 수 있다. 이 정책에서는 이를 허용하지 않는다.

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
