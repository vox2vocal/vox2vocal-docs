# Vox2Vocal Git Policy

이 문서는 모든 Vox2Vocal 독립 repository에 동일하게 적용하는 Git 커밋 정책을 정의한다.

## Required Identity

모든 커밋의 author와 committer는 아래 값과 정확히 일치해야 한다.

```txt
gitbyul <gitbyul@gmail.com>
```

다른 개인 계정, bot 계정, 로컬 머신 이메일, GitHub noreply 이메일은 허용하지 않는다.

각 repository에서 다음 명령으로 local Git 설정을 고정한다.

```bash
scripts/install-git-policy-hooks.sh
```

이 명령은 다음 설정을 적용한다.

```bash
git config user.name "gitbyul"
git config user.email "gitbyul@gmail.com"
git config core.hooksPath .githooks
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
- 본문 bullet 사이에 빈 줄을 넣지 않는다.

예시:

```txt
chore(infra): 데이터 저장소 이미지 버전 고정

- PostgreSQL 이미지를 postgres:17.10-alpine으로 고정
- Redis 이미지를 redis:7.2.14-alpine3.21로 고정
- infra README에 데이터 저장소 이미지 기준 추가
```

## Local Hook

각 repository에는 다음 hook을 둔다.

- `.githooks/commit-msg`: 커밋 메시지와 local Git identity 검사
- `.githooks/pre-push`: push 대상 커밋의 메시지와 author/committer 검사

hook은 개발자 실수를 빠르게 막기 위한 장치다. `--no-verify`로 우회될 수 있으므로 최종 강제는 GitHub ruleset과 required CI check에서 수행한다.

## GitHub Actions

각 repository에는 `.github/workflows/git-policy.yml`을 둔다.

Workflow는 PR에 새로 포함되는 커밋만 검사한다. 기존 히스토리에는 과거 정책 위반 커밋이 있으므로 전체 히스토리를 검사하지 않는다.

검사 항목:

- author name
- author email
- committer name
- committer email
- commit message header
- commit message body

## Required GitHub Ruleset

각 repository의 `main` branch에 동일한 ruleset을 적용한다.

필수 설정:

- Require a pull request before merging
- Require status checks to pass before merging
- Required check: `git-policy`
- Block force pushes
- Restrict deletions
- Restrict updates to `gitbyul` only
- Do not allow bypassing rules
- Require signed commits

권장 설정:

- Require linear history
- Require conversation resolution before merging
- Require branches to be up to date before merging

## Merge Policy

GitHub UI가 생성하는 squash commit 또는 merge commit은 committer가 GitHub 계정으로 기록될 수 있다. 최종 main 히스토리에서도 committer를 `gitbyul <gitbyul@gmail.com>`로만 유지하려면 GitHub UI에서 생성하는 merge commit을 사용하지 않는다.

가장 엄격한 방식은 다음 흐름이다.

1. feature branch에서 정책을 통과한 커밋을 만든다.
2. PR에서 `git-policy` check를 required로 통과시킨다.
3. `main`에는 ruleset을 적용해서 무단 push와 force push를 막는다.
4. release 또는 merge 권한은 `gitbyul` 계정으로만 운영한다.

GitHub UI merge를 허용하는 경우, PR 커밋의 author/committer는 강제할 수 있지만 GitHub가 최종 merge commit committer로 남을 수 있다. 이 정책에서는 이를 허용하지 않는다.
