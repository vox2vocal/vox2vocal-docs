# Vox2Vocal MVP PRD Draft

문서 버전: v0.11
작성일: 2026-06-13
상태: 초안
작성 기준: `pm-context` + `prd-writer` skill 기준, `prd-reviewer` readiness pass 반영

## Context Brief

### Brief

- Product or feature: Vox2Vocal MVP
- Target users: J-POP, 우타이테, 보컬 곡을 배우는 음악 학습자와 이를 지도하는 보컬/음악 교육자
- Problem: 학습자는 한 곡을 연습하면서 자신의 음정, 박자, 발음, 표현이 원곡 또는 목표 보컬과 얼마나 다른지 객관적으로 파악하기 어렵다. 교육자는 학생의 녹음물을 반복해서 듣고 피드백해야 하며, 구간별 문제를 빠르게 시각화하고 설명할 도구가 부족하다.
- Goal: P0 내부 운영에서 사용자가 관리자 등록 J-POP 곡을 선택하고, target section을 선택한 뒤, 앱에서 본인 목소리를 녹음해 보정/생성된 self-voice section preview로 들어보는 경험을 1차로 검증한다. 2차로 현재 음정과 목표 음정의 차이를 보여준다. 진성/비성/두성 등 발성/공명 유형 후보는 P0 필수 구현이 아니라 P1 이후 교사/전문가 검토 실험으로 둔다.
- Constraints: 현재 앱은 로그인/회원가입 UI 중심이며 실제 인증 API 연결 전 단계다. 백엔드는 BFF, API Gateway, User Service의 인증 계약이 존재한다. 엔진은 전체 아키텍처와 일부 audio-ingest 구현이 존재하며, 나머지 엔진은 문서 기준 MVP 범위가 정의되어 있다. 한 곡 단위 입력은 처리 시간, 비용, chunking, 품질 일관성에 대한 기술 검토가 필요하다.
- Success criteria: baseline과 target은 아직 실제 사용자/처리 데이터가 없어 가정이 필요하다. P0 내부 운영에서는 "이 preview가 내 목소리처럼 들린다" 4점 이상을 self-voice preview의 1차 성공 기준으로 두고, section-level job completion, pitch matching usefulness, failure reason tagging, safety block rate를 우선 측정한다.

### Known Facts

- Expo React Native App/Web 앱이 있고 로그인, 회원가입 화면이 구현되어 있다. Source: `vox2vocal-app/README.md`, `vox2vocal-app/src/features/auth/`
- BFF는 앱용 GraphQL endpoint를 제공하고 API Gateway를 gRPC로 호출하는 구조다. Source: `vox2vocal-bff-server/README.md`
- API Gateway는 `SignUp`, `Login`, `GetCurrentUser` gRPC 계약을 가진다. Source: `vox2vocal-api-gateway/proto/gateway.proto`
- User Service는 사용자 도메인, PostgreSQL, Prisma, 비밀번호 인증 정책을 소유한다. Source: `vox2vocal-user-service/README.md`, `src/users/policies/`
- 엔진 아키텍처는 Audio Ingest에서 시작해 Voice Analysis, Voice Pitch, Phoneme Alignment, Rhythm Timing, Melody Mapping, Singing Synthesis, Vocoder Render, Mix Master, Evaluation, Safety Rights로 이어지는 파이프라인을 지향한다. Source: `vox2vocal-docs/engine/README.md`
- Safety Rights는 초기 MVP부터 포함되어야 하며 사용자 본인 voice input만 허용하는 방향이 문서화되어 있다. Source: `vox2vocal-docs/engine/safety-rights/README.md`

### Confirmed Decisions

- Self-voice preview의 P0 primary success 기준은 "내 목소리처럼 들림"을 primary rating으로 둔다.
- P0 내부 운영 사용자 규모는 학습자 10명, 교육자 2명으로 둔다.
- P0 내부 운영은 Ken Kamikita의 `Mist` 전체 section map을 song package에 등록하고, 1차 target section은 `intro`로 둔다.
- P0 내부 운영에서는 한 곡 전체가 아니어도 target section 단위 preview/분석이 성공하면 P0 success 후보로 인정할 수 있다.
- 학습자 core flow는 `곡 선택 -> 파트/section 선택 -> 본인 목소리 녹음 -> take review -> 처리 시작`으로 둔다. 파일 업로드는 recorder를 쓸 수 없는 경우의 fallback 또는 내부 운영 보조 경로다.
- 목표 음정 기준은 원곡 음원 분석과 엔진이 추정한 note sequence를 함께 사용한다.
- P0 내부 운영에서 노래/reference audio는 관리자만 등록한다. 학습자는 관리자 등록 곡과 section을 선택하고 본인 목소리 take만 녹음 또는 fallback upload로 제출한다.
- 관리자 곡 등록은 곡명/아티스트, 원곡/reference audio, 언어, 가사, BPM, key, 구간 정보를 포함하는 song package를 만드는 흐름으로 둔다.
- 곡명/아티스트, 언어, 가사, 음원 metadata는 YouTube, Spotify, lyrics provider 등 외부 도메인/API에서 가져올 수 있게 설계하되, 실제 provider는 라이선스와 API 이용 조건을 확인한 뒤 확정한다.
- P0는 pitch-first로 진행한다. 가사 sync는 P1 실험 범위로 미룬다.
- BPM과 key는 원곡 기본값에서 가져오거나 엔진이 추정한 값을 기본값으로 사용하고, 사용자가 수정할 수 있게 한다.
- 일본어 가사 정렬과 lyric sync는 P0에서 제외하고 P1 실험으로 미룬다.
- 원곡 분석과 엔진 추정 note sequence가 충돌하면 confidence 기반으로 처리하고, 신뢰도가 낮거나 충돌이 큰 구간은 강제 판정하지 않는다.
- 진성/비성/두성 등 발성/공명 유형 분석은 P0 필수 구현에서 제외하고, P1 이후 교사/전문가 검토용 실험으로 둔다.
- 교사/전문가가 검토한 발성/공명 라벨은 P0 필수 데이터가 아니며, P1 이후 후보 데이터로 저장할 경우 consent, provenance, label confidence, 교사 간 일치도를 함께 저장한다.
- Self-voice preview가 완전히 실패하면 최종 job은 failed로 본다. 단, pitch extraction, alignment, evaluation 등 stage별 성공/실패 artifact는 별도로 기록한다.
- P0 내부 운영 결과물은 다운로드하지 않고 앱 안에서만 재생한다.
- 보관 기간은 최소 1개월을 기준으로 둔다. raw audio는 장기 보관하지 않으며, raw audio를 제외한 분석/audit 데이터는 최대 1년 보관을 기준으로 검토한다.
- P0 song package 필수 입력값은 title, artist, language, BPM, key, reference audio, source/provenance, rights clearance status, usage status, section map, target section start/end, reference pre-listen flag/scope, lyrics display flag/scope, lyrics sync flag, license expiry/re-review date로 확정한다.
- 필수 동의는 회원가입 또는 최초 로그인 시 1회 받을 수 있다. 단, 각 job은 현재 정책 version/scope와 일치하는 consent snapshot을 저장해야 하며, 정책 문서/동의 범위/version이 바뀌면 녹음 또는 job 제출 전에 재동의를 요구한다.
- 녹음 화면의 원곡 듣기와 가사 표시는 기본적으로 차단한다. `reference_prelisten_allowed`, `lyrics_display_allowed`, `lyrics_sync_allowed` 같은 권리 플래그와 display scope가 명시적으로 허용될 때만 section-limited, app-only, signed, audited 방식으로 노출한다.
- P0 기본값은 녹음 중 원곡 동시 재생을 허용하지 않는 것이다. 권리 플래그가 허용되더라도 원곡/reference audio는 녹음 전 section pre-listen으로 제공하고, 활성 녹음이 시작되면 재생을 멈춘다.

### Song Section Definition

P0에서 기준이 되는 단위는 "후렴"만이 아니라 song package에 등록된 `section`이다. 제품/엔진 관점에서는 음악 이론적 명칭보다 관리자 song package에 등록된 `section_id`, `section_label`, `start_timestamp`, `end_timestamp`, `source_reference_asset_id`가 기준이다. 모든 timestamp는 하나의 고정된 reference audio asset의 0:00을 기준으로 한다.

- P0 target song: Ken Kamikita - `Mist`
- P0 default target section: `intro`
- P0 default target timestamp: `0:00-0:28`
- P0 default target duration: 28초
- 권장 target section 길이: 20-40초. `intro`는 28초로 이 범위 안에 있으므로 P0 기본값으로 적합하다.
- 필수 metadata: section id, section label, start timestamp, end timestamp, representative lyric cue, BPM, key, reference audio source/provenance, rights clearance status, reference pre-listen flag, lyrics display flag, lyrics sync flag, display scope, license expiry/re-review date
- P0 success 판단은 곡 전체가 아니라 선택된 target section의 self-voice preview와 pitch feedback을 기준으로 한다.
- P0 target tradeoff: `intro`는 짧고 감정 표현이 명확해 self-voice preview 검증에는 적합하지만, 나레이션/독백 중심이라 singing pitch matching과 고음부 synthesis 대표성은 약하다. `chorus_1`은 P1 또는 두 번째 validation target 후보로 유지한다.

#### Mist Section Timeline

| Section ID | Time Range | Label | Notes |
| --- | --- | --- | --- |
| `intro` | `0:00-0:28` | Intro / 나레이션 도입부 | 조용한 피아노 반주 위에 "날지 못하는 날개라면~" 하고 독백을 읊조리는 파트. P0 default target section. |
| `verse_1_a_humming` | `0:28-0:42` | 1절 A멜로 1 / 허밍 파트 | 나레이션 이후 어쿠스틱 기타 반주가 들어오며 가사 없이 "아아아아~", "Uh~ Huh~" 계열의 허밍을 넣는 감성적 구간. |
| `verse_1_a_vocal` | `0:42-1:17` | 1절 A멜로 2 / 본격 보컬 파트 | 허밍 이후 담담하고 웅얼거리듯 첫 가사를 시작하는 구간. 시작 가사: "濡れた土の匂い..." |
| `verse_1_b` | `1:17-1:44` | 1절 B멜로 / 빌드업 | 드럼 사운드가 추가되면서 후렴구를 향해 감정을 서서히 고조시키는 구간. 시작 가사: "いつか辿った道が..." |
| `chorus_1` | `1:44-2:16` | 1절 Chorus / 첫 번째 후렴 하이라이트 | 악기 연주가 가장 풍성해지며 감정이 폭발하는 첫 후렴구. 시작 가사: "誰かが、未来の僕らを探してる..." |
| `interlude` | `2:16-2:42` | Interlude / 간주 | 1절이 끝나고 다음 악장으로 넘어가기 전의 연주 구간. |
| `bridge` | `2:42-3:09` | Bridge / C멜로 | 독백하듯 다시 조용해졌다가 마지막 후렴을 위해 에너지를 응축하는 구간. 시작 가사: "手を伸ばす、淡い、淡い、あの光..." |
| `chorus_2` | `3:09-3:41` | 2절 Chorus / 두 번째 후렴 | 다시 한번 감정이 터져 나오는 익숙한 후렴구 파트. |
| `last_chorus` | `3:41-4:18` | Last Chorus / 최종 후렴 | 보컬의 에너지가 최고조에 달하며 감정을 거침없이 쏟아내는 마지막 하이라이트. 시작 가사: "君が歩み、耐え抜いたその最期に..." |
| `outro` | `4:18-4:53` | Outro / 후주 | 모든 격정적인 세션이 멈추고 피아노 선율만 흐르며 여운을 남기고 마무리되는 구간. |

P0에서는 Section Selection 화면을 별도 단계로 둔다. 실제 선택지는 `intro` 하나로 시작할 수 있지만, 제품 모델은 곡 선택과 파트/section 선택을 분리해 향후 section 비교와 full-song merge로 확장할 수 있게 한다.

### Rights And Reference Audio Policy Draft

P0는 실제 노래를 reference로 사용하는 제품이므로 저작권, 음원 권리, 가사 권리, provider 약관, voice/right-of-publicity 리스크를 최고 보수 기준으로 다룬다. 원칙은 "권리와 출처가 확인된 관리자 등록 reference만 내부 분석에 사용하고, 외부 provider는 P0에서 audio source가 아니라 metadata 후보 source로만 다룬다"이다.

Sources: [U.S. Copyright Office AI Report](https://www.copyright.gov/ai/), [U.S. Copyright Office Digital Replicas Report](https://www.copyright.gov/ai/Copyright-and-Artificial-Intelligence-Part-1-Digital-Replicas-Report.pdf), [U.S. Copyright Office Generative AI Training Report](https://www.copyright.gov/ai/Copyright-and-Artificial-Intelligence-Part-3-Generative-AI-Training-Report-Pre-Publication-Version.pdf), [YouTube API Services Terms](https://developers.google.com/youtube/terms/api-services-terms-of-service), [Spotify Developer Policy](https://developer.spotify.com/policy), [ElevenLabs Terms](https://elevenlabs.io/terms-of-use), [ElevenLabs Safety](https://elevenlabs.io/safety), [ElevenLabs Prohibited Use Policy](https://elevenlabs.io/use-policy).

- P0 reference audio must be admin-provided and must have explicit source/provenance, allowed use, retention period, and deletion owner recorded before publication to learners.
- YouTube, Spotify, lyrics providers, and similar external services must not be used to download, rip, cache, analyze, train on, remix, or derive audio unless a written license or provider-approved use path explicitly permits it.
- Metadata provider integration may collect candidate title, artist, language, album, artwork, BPM/key hints, provider ids, or lyrics metadata only when each provider's terms permit the use. Metadata ingestion must remain optional and non-blocking for P0.
- Lyrics and lyric timestamps are optional in P0. If lyrics are stored or shown, lyrics source, license, retention, display scope, line/section scope, and sync confidence must be tracked separately from reference audio. Full lyrics must not be written to logs.
- Reference audio may be used by default only for app-internal target pitch extraction, section validation, quality comparison, and internal review. Learner-facing reference pre-listen is hidden unless explicit rights flags permit section-limited app-only playback. Reference audio must not be exposed for download, sharing, model training, third-party voice cloning, public streaming, marketing content, or user export.
- If learner-facing reference pre-listen is allowed, it must be short-lived signed playback, section-limited, no-download/share, auditable, and stopped before active microphone recording starts. The recording screen must warn users to use headphones so original audio does not leak into their vocal take.
- Generated self-voice preview must be app-only playback during P0 internal operation. Download, share links, public posting, and commercial use are out of scope.
- The product must not clone or imitate Ken Kamikita's voice, a featured artist's voice, a celebrity voice, a character voice, or any third-party voice. The only allowed generated vocal identity is the submitting user's own voice.
- If reference rights are uncertain, provider terms are unclear, or provenance is incomplete, the song package status must be `rights_blocked` and cannot be published to learners.
- If a copyright or rights complaint is received, related reference audio, generated previews, and derivative artifacts must be blocked from playback while the policy owner reviews the case.

### Retention Draft

| 데이터 | 보관 기간 | 정책 |
| --- | --- | --- |
| 사용자 원본 보컬 raw audio | 최소 1개월, 기본 30일 | P0 내부 운영 검증 후 삭제 대상이다. 1년 보관 데이터셋에는 포함하지 않는다. |
| 관리자 등록 reference audio | 최소 1개월, P0 곡 카탈로그 운영 기간 동안 보관 | pitch target 추출과 앱 내 비교 분석에만 사용한다. rights clearance status와 provider/license 조건에 따라 더 짧게 삭제할 수 있다. |
| section lyric/display data | license 조건 또는 P0 곡 카탈로그 운영 기간 중 더 짧은 기간 | source, license, display scope, sync confidence를 reference audio와 분리해 보관한다. full lyrics는 로그에 남기지 않는다. |
| canonical/processed audio | 최소 1개월, 기본 30일 | 재처리와 디버깅을 위해 보관하되 raw audio 장기 보관 대체물로 쓰지 않는다. |
| generated self-voice preview | 최소 1개월, 기본 30일 | 앱 내 재생만 허용하고 다운로드/export는 차단한다. |
| pitch/note/alignment JSON | 최대 1년 | raw audio 없는 분석 데이터로 보관 가능하다. source provenance와 job id를 유지한다. |
| preview 품질 리포트와 confidence score | 최대 1년 | 품질 개선, 회귀 분석, P0 평가에 사용한다. |
| 교사/전문가 발성 라벨 | P1 이후 최대 1년 후보 | P0 필수 데이터가 아니다. P1 이후 저장 시 consent, source provenance, label confidence, 교사 간 일치도를 함께 저장할 때만 candidate data로 사용한다. |
| audit log | 최대 1년 | 권한, 동의, 접근, preview 요청 기록을 보관한다. raw audio를 포함하지 않는다. |
| 운영 로그 | 30-90일 | 장애 분석과 처리시간 측정에 사용한다. 개인정보/원본 음성 포함을 금지한다. |

### Data Governance And Security Draft

Vox2Vocal P0 data governance는 NIST CSF의 risk management 관점, NIST SP 800-88의 media sanitization 개념, OWASP Logging Cheat Sheet의 민감정보 비로그 원칙을 따른다. Source: [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework), [NIST SP 800-88 Rev. 2](https://csrc.nist.gov/pubs/sp/800/88/r2/final), [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html).

| Data Class | Examples | Security Level | Storage And Access | Retention And Deletion |
| --- | --- | --- | --- | --- |
| User raw voice audio | 사용자 원본 보컬 upload | Critical personal voice data | Separate private bucket, dedicated KMS key, no public URL, least-privilege service access, break-glass human access only | 기본 30일. deletion job 이후 recovery infeasible 상태를 목표로 삭제 evidence만 보관 |
| Processed/canonical audio | normalized wav, mono PCM, trimmed section | High personal voice data | Same boundary as raw audio unless proven de-identified | 기본 30일. raw audio 장기 보관 대체물로 사용 금지 |
| Generated self-voice preview | 보정/생성된 사용자 목소리 preview | High personal/generated voice data | App-only signed playback, no download/export/share, watermark/provenance metadata where feasible | 기본 30일. 삭제 요청 또는 권리 이슈 시 playback block 후 삭제 |
| Admin reference audio | `Mist` reference asset | High copyright-restricted data | Admin/service-only access, separate rights metadata, no learner download, no model training | P0 catalog 운영 기간 또는 license 조건 중 더 짧은 기간 |
| Section map and target notes | section timestamps, note sequence JSON | Medium analysis data | Internal API access, source/provenance required | 최대 1년. source reference id와 rights status 유지 |
| Pitch and quality reports | F0, confidence, clipping, artifact flags | Medium analysis data | Product/QA/engine access with pseudonymous user id | 최대 1년. raw audio 없이 보관 |
| Ratings and failure tags | 1-5 score, failure tags, `other` text | Medium product data | Pseudonymized analytics access, free text review for personal data leakage | 최대 1년. candidate data opt-in과 분리 |
| Expert labels | vocal-mode candidate labels, confidence | High candidate training data | P1 이후 별도 consent, provenance, label confidence, inter-rater agreement required | P1 이후 최대 1년 후보. P0 필수 데이터 아님 |
| Audit logs | consent, access, allow/deny, deletion evidence | High governance data | Append-only store, restricted security/platform access, no raw audio, no secrets, no tokens | 최대 1년. 법/정책상 필요한 경우만 연장 |
| Operational logs | job id, stage, latency, error code | Low/Medium operational data | Structured logs, no raw audio, no generated preview, no credentials, no full lyrics | 30-90일 |

Security requirements:

- All object storage containing user voice, generated preview, or reference audio must be private by default and encrypted at rest.
- Audio artifacts and reference assets must use short-lived signed access for app playback or internal processing only.
- Human access to raw audio requires time-bound approval, purpose, ticket id, reviewer, and immutable audit record.
- Logs must not contain raw audio, generated preview URLs, access tokens, refresh tokens, passwords, provider secrets, full lyrics, or sensitive free-text content.
- Deletion evidence must include artifact id, data class, deletion request source, retention deadline, deletion job id, deletion status, timestamp, and actor/service id. Evidence must not include raw audio or playable preview.
- If deletion fails, the artifact must be marked `deletion_failed`, playback must be blocked for rights-sensitive assets, and the platform/storage owner must review it.
- Each data class must have an owner before build start: Product owner for purpose/scope, Platform/storage owner for retention/deletion, Security/policy owner for access and rights review, Engineering owner for implementation.

### P1 Vocal Label Data Draft

교사/전문가가 판단한 진성, 비성, 두성 등 발성/공명 라벨은 P0 필수 범위가 아니다. P1 이후 수집하더라도 바로 모델 학습에 사용하지 않는다. 먼저 검토용 candidate data로 수집하고, 다음 metadata를 함께 저장해야 한다.

- consent: 사용자가 향후 품질 개선 또는 학습 데이터 후보로 쓰는 데 동의했는지
- source provenance: 어떤 사용자 녹음, 어떤 곡, 어떤 구간, 어떤 엔진 output에서 나온 라벨인지
- label confidence: 교사/전문가가 해당 라벨을 얼마나 확신하는지
- inter-rater agreement: 두 명 이상의 교사/전문가 판단이 얼마나 일치하는지
- label version: 진성/비성/두성 정의와 라벨링 가이드 버전

이 데이터가 충분히 쌓이면 추후 AI 또는 엔진이 불특정 다수의 목소리에서도 발성/공명 후보를 더 안정적으로 판단할 수 있는지 검토한다. 단, P0에서는 자동 확정 판정이 아니라 교사/전문가 검토를 돕는 후보 정보로만 사용한다.

### Processing Time Definition

처리시간은 사용자가 관리자 등록 곡과 section을 선택하고 본인 목소리 take를 녹음 또는 fallback upload로 제출한 뒤, 앱에서 preview를 들을 수 있을 때까지 걸리는 시간이다.

- Time to first preview: 첫 구간 self-voice preview를 앱에서 재생할 수 있을 때까지의 시간
- Time to full result: 선택한 구간 또는 곡 전체의 preview, pitch feedback, 품질 리포트가 모두 준비될 때까지의 시간
- Queue time: 작업이 엔진 처리 전에 대기한 시간
- Engine time: audio ingest, pitch, alignment, synthesis, render, evaluation 등 엔진 처리 시간

P0 내부 운영에서는 처리시간 목표를 공격적으로 잡지 않는다. 우선 구간 단위라도 안정적으로 성공하는지 검증하고, 이후 beta/제품화 단계에서 안정성과 속도를 함께 최적화한다.

### Quantified Thresholds Draft

P0 threshold는 P0 운영 중 조정될 수 있지만, engineering handoff와 QA 재현성을 위해 다음 기본값으로 시작한다.

| Area | P0 Threshold | Behavior |
| --- | --- | --- |
| Target section duration | 20-40초 권장, `intro`는 28초 | 20초 미만은 학습 가치 부족 후보, 40초 초과는 처리 안정성 리스크로 review |
| Upload hard max | 60초 | 초과 시 full-song 처리하지 않고 trim/section 선택 요청 |
| Section length tolerance | target duration 대비 `±5초` 또는 `±20%` 중 더 큰 값 | tolerance 밖이면 section mismatch warning 또는 trim 요청 |
| Pitch frame confidence trusted | `>= 0.70` | scoring과 feedback에 사용 가능 |
| Pitch frame low confidence | `0.45-0.69` | feedback에는 표시하되 overconfident score에서 제외 |
| Pitch frame excluded | `< 0.45` | pitch scoring 제외, low-confidence artifact로 기록 |
| Target source conflict | reference vs engine note 차이 `> 50 cents`가 voiced frames의 20% 이상 | disputed section으로 표시하고 강제 판정 금지 |
| Severe target conflict | reference vs engine note 차이 `> 1 semitone`가 voiced frames의 10% 이상 | `needs_review` 후보로 표시 |
| Preview primary success | 학습자 10명 중 5명 이상이 primary question 4점 이상 | 4점 미만은 success로 계산하지 않음 |
| Preview quality guardrail | 2점 이하 응답이 2명 초과 | quality risk로 보고 P0 개선 필요 |
| Failure tag completion | 4점 미만 응답 100%가 failure tag 또는 `other` 포함 | 미포함 응답은 metric invalid |
| Job completion target | valid P0 section jobs 중 60% 이상이 `completed` and `preview_available=true` | status와 output flag를 함께 사용해 계산 |
| Time to preview observation | P50 10분 이하, P95 30분 이하를 관찰 기준으로 기록 | P0 hard success metric은 아님 |
| Timeout threshold | 60분 초과 | `failed` 또는 `failed_with_partial_artifacts` 후보 |
| Deletion execution | retention deadline 이후 24시간 이내 deletion job 실행 | 실패 시 `deletion_failed`와 owner review |

### Consent And Access Draft

ElevenLabs benchmark 기준으로, Vox2Vocal은 P0 내부 운영에서 더 보수적인 consent model을 적용한다. 참고한 방향은 "본인 또는 권한 있는 목소리만 사용", "필요 권리 보유", "동의 없는 타인 목소리 모방 금지", "고위험 voice 차단 및 기술적 검증", "삭제 및 학습 사용 opt-out"이다. Source: [ElevenLabs Terms of Service](https://elevenlabs.io/terms-of-use), [ElevenLabs Safety](https://elevenlabs.io/safety), [ElevenLabs Prohibited Use Policy](https://elevenlabs.io/use-policy)

P0 consent는 다음처럼 분리한다.

- Own voice consent: 사용자가 녹음 또는 fallback upload로 제출한 보컬이 본인 음성임을 확인한다. 분석/preview 생성의 필수 동의다.
- Generated preview consent: 본인 음성을 기반으로 generated self-voice preview가 생성되고 앱 안에서 재생될 수 있음을 확인한다. 필수 동의다.
- Expert review consent: 교사/전문가 또는 내부 reviewer가 preview, pitch report, failure reason tags를 검토할 수 있음을 확인한다. P0 job 제출 조건으로 별도 동의한다.
- Candidate data consent: failure reason tags와 non-audio analysis artifact를 향후 품질 개선 후보 데이터로 저장할 수 있음을 별도 opt-in으로 받는다. Vocal-mode label candidate data는 P1 이후 별도 동의로 다룬다.
- Retention/deletion notice: raw audio 30일 기본 보관, generated preview 30일 기본 보관, raw audio 없는 analysis/audit/label 데이터 최대 1년 후보 보관을 고지한다.

Required service consent may be collected once during signup or first login, but the app must still bind each job to the exact consent snapshot used at submission time. If consent type, policy document version, scope, required/optional status, or display/playback rights scope changes, recording submission must be blocked until the user re-consents. The "no recording/capture/redistribution/public posting" restriction applies to reference audio, lyrics, generated previews, and app output; it must not be phrased as a ban on recording the user's own voice.

P0 접근권한은 다음처럼 제한한다.

- 사용자 본인: 본인 recorded/fallback input, generated preview, pitch report, job status, 삭제 요청에 접근한다.
- 엔진 개발자: debugging 목적의 job artifact와 필요한 최소 raw/canonical audio에 제한적으로 접근한다. 접근은 time-bound, audited, least-privilege로 둔다.
- 교사/전문가: 사용자가 동의한 job의 preview, pitch report, failure reason tags에 접근한다. raw audio 직접 접근은 기본 차단하고 필요 시 별도 권한으로만 허용한다.
- Product/QA: user-identifiable raw audio 없이 quality report, metrics, failure reason, anonymized artifact에 접근한다.
- 운영/보안 관리자: retention, deletion, audit, incident response에 필요한 metadata에 접근한다. raw audio 접근은 break-glass로 제한한다.
- 마케팅/일반 운영 인력: raw audio, generated preview, expert label에 접근하지 않는다.

### Job State Ownership Draft

모범사례 기준으로 conversion job의 source of truth는 앱 UI나 개별 engine worker가 임의로 결정하지 않아야 한다. 다만 P0에서 전용 Conversion Job Orchestrator 신규 서비스를 필수화하지 않는다. P0는 기존 backend/worker 안의 bounded module로 canonical job state를 시작하고, P0 검증 이후 필요하면 전용 orchestrator로 분리한다. Temporal은 workflow의 event history를 source of truth로 두고 failure 이후에도 상태를 복원하는 durable execution을 제공한다. AWS Step Functions도 state machine/execution history와 long-running, auditable workflow를 지원한다. Source: [Temporal durable execution](https://docs.temporal.io/temporal), [Temporal workflow event history](https://docs.temporal.io/workflows), [AWS Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html)

Vox2Vocal P0 권장 구조는 다음과 같다.

- Canonical job state module: job id, canonical state, stage transition, retry, partial/final decision, retention deadline을 소유한다.
- BFF: 앱용 GraphQL facade로 create job, upload initiation, job status query/subscription, result retrieval만 중계한다. BFF는 job state의 source of truth가 아니다.
- API Gateway: 내부 service 호출과 auth/user context 전달을 담당한다.
- Engine workers: ingest, pitch, alignment, synthesis, render, evaluation stage를 수행하고 stage result event와 artifact pointer를 emit한다.
- Event stream or stage result table: stage events/results를 전달하거나 저장한다. canonical job state module은 idempotent하게 반영한다.
- Read model: 앱 조회용 job projection은 canonical job state에서 생성한다.

P0 상태 모델은 `created`, `queued`, `processing`, `preview_ready`, `completed`, `failed`, `failed_with_partial_artifacts`, `blocked`, `needs_review`, `deleted`를 기준으로 한다. Self-voice preview가 완전히 실패하면 최종 상태는 `failed` 또는 `failed_with_partial_artifacts`이며, pitch feedback만 성공해도 P0 success로 보지 않는다.

Output availability is tracked separately from job state:

- `preview_available`: self-voice preview can be played in-app.
- `pitch_report_available`: pitch feedback exists.
- `section_limited`: output is limited to a target section rather than full song.
- `rating_required`: primary preview rating should be collected.
- `failure_tags_required`: rating is below 4 and failure tags are required.

### Assumptions

- 1차 MVP의 핵심 검증은 "상업 출시 품질의 완성 보컬"이 아니라 "내 목소리가 보정/생성된 노래를 들어보며 연습 동기를 얻는 self-voice preview 경험"이다.
- 장기 제품 목표 입력 길이는 J-POP 또는 우타이테 노래 한 곡 기준이다. 단, P0 내부 운영에서는 `Mist` target section 처리로 제한한다.
- 보이스 사용 정책은 사용자 본인 음성만 허용한다.
- 외부 배포, 상업 이용, 제3자 음성 복제는 MVP 범위에서 제외하고 Safety Rights에서 차단한다.
- 1차 산출물은 "내 목소리가 보정/생성된 노래를 들어보는 self-voice song preview"다.
- 2차 산출물은 현재 내 목소리의 음정과 노래에 필요한 목표 음정이 얼마나 맞는지 보여주는 pitch matching feedback이다.
- 3차 산출물인 진성, 비성, 두성 등 발성/공명 유형 후보와 confidence는 P0에서 제외하고 P1 이후 실험으로 둔다.
- P0 내부 운영에서 section-limited preview는 허용 가능한 성공 상태다. P0 default target은 `intro`이지만, 제품 모델은 전체 song section map을 기준으로 한다.
- 학습자는 원곡/reference audio를 직접 업로드하지 않고 관리자 등록 곡과 section을 선택한다.
- P0 내부 운영은 속도보다 안정적 성공 여부를 우선 검증한다. 추후에는 안정성과 속도를 모두 제품 품질 기준으로 둔다.
- 추후 다른 프로젝트에서는 같은 song package/metadata ingestion 엔진을 재사용해 외부 사용자가 노래를 업로드하는 확장 가능성을 열어둔다.
- 모바일 앱을 primary surface로 두되, Expo Web도 같은 핵심 흐름을 지원하는 방향으로 설계한다.
- 출시 형태는 P0 내부 운영이다.

### Risks

- self-voice preview 품질이 낮으면 1차 가치가 바로 흔들릴 수 있다. 학습 리포트는 이를 보조하지만 대체하지 않는다.
- 진성/비성/두성 분류는 음성교육 용어와 모델 output의 대응이 애매하므로 P0에서 제외한다. P1 이후 다룰 경우에도 확정 판정보다 후보/근거/신뢰도로 표현해야 한다.
- 관리자 곡 등록에서 외부 음원/가사 metadata를 사용하면 저작권, API 이용 조건, 저장 기간, 접근 권한, 삭제 정책을 명확히 해야 한다.
- 권리/동의 정책이 약하면 제품 리스크가 기술 리스크보다 커질 수 있다.
- 한 곡 단위 입력은 처리 시간, 비용, 메모리, 엔진 간 artifact 크기, 오류 복구 난이도를 크게 키울 수 있으므로 P0에서는 target section으로 제한한다.
- 현재 인증 UI와 백엔드 인증 계약 사이의 실제 연동이 완료되지 않으면 upload/conversion 흐름의 사용자 식별과 audit이 막힌다.

### Recommended Next Skill

- Skill: `prd-writer`
- Reason: 프로젝트 문서와 코드에서 제품 방향, 현재 구현 상태, 엔진 MVP 범위를 확인했으므로 engineering handoff 가능한 PRD 초안 작성으로 진행한다.

## Executive Summary

Vox2Vocal P0 MVP는 음악 학습자가 Ken Kamikita의 `Mist`를 선택하고, `intro` target section을 선택한 뒤, 앱에서 본인 목소리 take를 녹음하면 시스템이 이를 보정/생성된 self-voice section preview로 들려주는 내부 운영 제품이다. 같은 결과 화면에서 현재 음정과 목표 음정의 차이를 선택된 section 기준으로 보여준다.

이 PRD의 목적은 "내 목소리로 보정/생성된 target section preview를 들어보는 경험"이 실제 음악 학습 동기를 만들 수 있는지 확인하는 것이다. 따라서 P0는 상업 배포, 다중 보이스 모델, 정교한 DAW 편집, 전체 곡 생성, 자동 발성 유형 분석보다 가입, 관리자 곡 등록, 곡 선택, section 선택, 앱 내 본인 목소리 녹음, take review, 처리 상태 확인, section-limited self-voice preview 앱 내 재생, 음정 매칭 피드백, 실패 원인 태깅, 본인 음성 권한 차단을 우선한다.

## Problem Statement

음악을 배우는 사람은 J-POP, 우타이테, 보컬 곡을 연습할 때 "내 목소리로 잘 부르면 어떻게 들릴지"를 바로 확인하기 어렵다. 원곡을 듣고 따라 부를 수는 있지만, 자신의 목소리가 보정되거나 목표 음정에 맞춰졌을 때의 결과를 듣지 못하면 연습 방향을 체감하기 어렵다. 또한 실제 연습은 곡 전체보다 특정 파트를 반복하는 경우가 많기 때문에 곡 선택, 파트 선택, 녹음, take 확인이 자연스럽게 이어져야 한다.

그 다음 문제는 원인 파악이다. 학습자는 현재 자신의 음정이 무엇인지, 노래에 필요한 음정과 얼마나 다른지 알기 어렵다. 장기적으로는 진성/비성/두성 같은 발성 또는 공명 유형도 교육적 보조 정보가 될 수 있지만, P0는 이를 자동 판정하지 않는다. P0는 self-voice section preview를 1차 가치로 제공하고, 음정 매칭과 실패 원인 태깅을 통해 다음 개발 판단에 필요한 근거를 모으는 데 집중한다.

## Target Users

- Primary: J-POP, 우타이테, 애니송, 보컬 곡을 배우는 음악 학습자
- Secondary: 학생의 보컬 연습을 지도하는 음악 교사, 보컬 트레이너, 온라인 강사
- P0 internal participants: 제품팀, 엔진 개발자, QA, 제한된 음악 교육 협력자

## Goals

- 학습자가 Ken Kamikita - `Mist`를 선택하고 `intro` section을 선택한 뒤, 앱에서 본인 목소리 take를 녹음해 "내 목소리처럼 들리는" 보정/생성 preview를 들어볼 수 있다.
- P0 내부 운영에서 target section self-voice preview, pitch 비교, 품질 리포트가 안정적으로 성공하는지 검증한다.
- 학습자가 현재 음정과 목표 음정의 차이를 선택된 section 기준으로 이해할 수 있다.
- 학습자가 preview를 들은 직후 1-5점 평가를 남기고, 4점 미만이면 실패 원인을 태깅할 수 있다.
- 제품/엔진 팀이 section preview 실패 원인, pitch mismatch, artifact, 처리 stage 실패를 구분해서 볼 수 있다.
- 모든 분석/preview 요청은 본인 음성 정책, 앱 내 재생 정책, 보관 정책, audit 기록을 통과해야 한다.
- preview가 완전히 실패한 작업은 P0 success로 보지 않고, pitch-only 결과는 partial artifact로만 남긴다.

## Non-goals

- 제3자 유명인, 아티스트, 캐릭터 보이스 복제 지원
- 사용자가 직접 업로드한 본인 음성이 아닌 보이스 모델 사용
- 상업 배포 라이선스, 정산, 마켓플레이스 기능
- DAW 수준의 waveform 편집, MIDI editor, 멀티트랙 믹싱
- 실시간 변환, 라이브 레슨 스트리밍, 반주 포함 멀티트랙 믹싱
- 모든 장르와 모든 언어의 완전 지원. MVP는 J-POP/우타이테 학습 맥락을 우선한다.
- 고급 expression control, voice conversion, custom voice training
- 불특정 다수의 목소리에 대한 완전 자동 발성/공명 판정 모델
- P0 진행을 막을 수준의 완전 자동 YouTube/Spotify/lyrics provider ingestion
- YouTube, Spotify, lyrics provider 등 외부 provider에서 audio를 download/rip/cache/analyze/train/remix하는 기능
- P0에서 모든 곡 전체 구간의 완전한 preview 품질 보장
- P0에서 자동 진성/비성/두성 판정 또는 learner-facing vocal-mode feedback
- P0에서 한국어/일본어 phoneme alignment, lyric sync, syllable-level feedback
- P0에서 Conversion Job Orchestrator 신규 서비스 구현을 필수화하는 것

## Scope

### P0 Build Slice

- 관리자 수동 song package 등록: Ken Kamikita - `Mist`를 기준으로 title, artist, language, BPM, key, reference audio, source/provenance, rights clearance status, usage status, section map, target section start/end를 필수로 등록한다.
- 학습자 곡/파트 선택 및 본인 목소리 녹음: 학습자는 관리자 등록 곡을 선택하고, 별도 Section Selection 단계에서 `intro`를 선택한 뒤 앱 recorder로 본인 목소리 take를 녹음한다. 파일 업로드는 recorder를 쓸 수 없는 경우의 fallback 또는 내부 운영 보조 경로로 둔다.
- Section-first processing: P0 내부 운영은 `intro` target section에서 self-voice preview, pitch comparison, quality report가 안정적으로 생성되면 성공 후보로 본다.
- Pitch-first processing: 가사 sync는 P1 실험 범위로 두고, P0는 pitch target extraction과 pitch comparison을 우선한다.
- App-only preview and rights-gated guide: generated preview는 앱 내 재생만 허용하고 다운로드/export/share를 차단한다. Reference audio pre-listen과 lyrics display는 별도 권리 플래그가 허용하는 경우에만 section-limited로 노출한다.
- Safety and audit: 본인 음성 확인, rights decision, operation audit log, raw audio retention decision을 job 단위로 기록한다.
- Failure reason tagging: 4점 미만 preview 평가는 사용자가 실패 원인을 태그하고, 내부 reviewer가 기술 원인을 보강할 수 있어야 한다.

### P1 Experimental Scope

- 곡 전체 one-pass 처리 또는 chunked full-song merge
- YouTube, Spotify, lyrics provider 등 외부 provider 기반 metadata/lyrics 자동 수집
- 라이선스 또는 provider-approved use path가 없는 원곡 audio 자동 수집/분석
- 활성 microphone recording 중 원곡/reference audio 동시 재생
- AI 또는 alignment engine 기반 가사 timestamp sync 자동화
- 일본어 가사/음절 alignment
- 자동 vocal-mode candidate 생성과 teacher/expert-facing 검토 UI
- 교사/전문가 라벨 기반 future model-improvement dataset 설계
- Conversion Job Orchestrator 신규 서비스 분리

### In Scope

- 이메일/비밀번호 기반 회원가입, 로그인, 현재 사용자 조회
- 모바일 앱 및 웹에서 인증 후 MVP 작업 화면 접근
- 앱 내 본인 목소리 녹음: mic permission, count-in, timer, input meter, stop, replay own take, retake, submit
- 오디오 fallback upload: `wav`, `mp3`, 사용자 본인 보컬 연습 파일. in-app recorder가 platform-native format을 만들 경우 `audio-ingest`가 authoritative normalization을 수행한다.
- P0 object storage: MinIO를 사용하되 S3-compatible contract를 유지한다.
- 관리자 song package 등록: pitch target 추출과 비교 분석 목적의 원곡/reference audio, metadata, BPM, key, section map 정보를 등록. P0에서는 Ken Kamikita - `Mist` 수동 등록과 `intro` target section을 기준으로 하며 provider 자동화와 lyrics sync는 P1 실험 범위로 둔다.
- 학습자 곡 선택: 학습자는 관리자 등록 곡만 선택할 수 있다.
- 학습자 파트/section 선택: P0에서는 `intro`만 selectable이어도 곡 선택 이후 별도 section selection step을 둔다.
- 녹음 가이드: 권리 플래그가 허용되면 section-limited reference pre-listen과 line/section-scoped lyric cue를 제공한다. 권리 플래그가 없거나 만료되면 section label, timestamp, count-in, 녹음 길이만 제공한다.
- 입력 길이 목표: `Mist` target section 기준 20-40초. P0 default `intro`는 28초다. P0에서는 전체 곡 입력을 요구하지 않는다.
- 입력 metadata: 곡/작업명, 언어, BPM, key, 원곡/목표곡 정보, section id, section label, target section start/end
- Audio Ingest: 표준 PCM/WAV 변환, mono 변환, 무음 탐지, 발화 구간 timestamp 생성
- Voice Pitch: target section F0 추출, confidence filtering, MIDI note 변환, target pitch 비교 JSON
- Target Pitch Mapping: 관리자 reference audio 또는 엔진 추정 note sequence를 사용한 target section pitch/note 생성
- Self-voice Preview: 사용자 본인 음성 기반 target section self-voice preview 생성
- Preview Evaluation: preview playable 여부, section coverage, clipping/artifact 후보, pitch deviation, 실패 원인 태그 수집
- Safety Rights: 사용자 본인 voice input만 허용, conversion audit log
- 작업 상태 화면: queued, processing, preview_ready, completed, failed, failed_with_partial_artifacts, blocked
- 결과 화면: 앱 내 self-voice preview 재생, 현재/목표 음정 비교, 1-5점 평가, 4점 미만 실패 원인 태그, 실패 사유
- Contact follow-up: P0에서는 gate와 데이터 모델만 정의하고 기본 disabled로 둔다. 오류 안내, 후속 인터뷰, 내부 운영 follow-up 목적의 이메일/SNS 연락처 수집은 별도 opt-in, 암호화 저장, audit, contact collection gate를 통과한 뒤 Later decision으로 활성화한다.

### Out Of Scope

- 소셜 로그인 실제 연동
- 비밀번호 재설정 flow
- 결제/구독/크레딧
- 팀/협업 프로젝트
- 공개 공유 링크
- P0 내부 운영 결과물 다운로드 또는 외부 export
- 학습자/일반 사용자의 원곡/reference audio 직접 업로드
- 고급 vocal editing UI
- model training pipeline
- 운영자 관리 콘솔
- 제3자 보이스 모델 선택, 타인 음성 변환, 캐릭터/아티스트 보이스 복제
- P0 내부 운영 단계의 공개 출시 또는 외부 배포 기능

## User Stories

- As a music learner, I want to sign up and log in so that my practice recordings and feedback reports are tied to my account.
- As a music learner, I want to select the admin-registered `Mist` song and then select a song part/section so that I practice the intended target.
- As a music learner, I want to record my own `Mist` target section take in the app so that I can hear a corrected/generated section preview in my own voice.
- As a music learner, I want to replay or retake my own recording before submission so that preventable silence, clipping, or wrong-length takes do not waste processing time.
- As a music learner, I want to see section guidance, and when rights allow, hear the section or see line-level lyrics before recording so that I know what to sing without exposing copyrighted content incorrectly.
- As a music learner, I want to compare my current pitch with the target pitch for the target section so that I know whether I am singing the right notes.
- As a music learner, I want to rate whether the preview sounds like my voice so that the product team can judge whether the core experience works.
- As a music learner, I want to tag why a low-rated preview failed so that the team can distinguish voice similarity, pitch, timing, artifact, and playback issues.
- As a user, I want unauthorized voice usage to be blocked clearly so that I understand that only my own voice is allowed.
- As a product/engineering team member, I want evaluation metrics, failure reason tags, and engine artifacts so that I can debug quality regressions in the P0 section preview loop.

## Functional Requirements

### FR-001 Account Creation

- The app must allow users to create an account with email, password, display name, and terms consent.
- Signup or first login may collect the required P0 service consent set if every consent item is unbundled, versioned, and recorded with scope.
- Email must be normalized and validated.
- Password must be at least 8 characters.
- Display name must be required and limited to the backend policy maximum.
- Successful signup must return access token, refresh token, expiry, and user profile.

### FR-002 Login And Current User

- The app must allow users to log in with email and password.
- Failed login attempts must follow backend lockout policy.
- The app must store session state securely enough for MVP and attach access tokens to authenticated requests.
- The app must retrieve the current user through the BFF/Gateway flow.

### FR-003 Song And Section Selection

- The learner must select an admin-registered song package before selecting a section.
- The learner must select a target section before recording or submitting a take.
- The selected P0 song package must be Ken Kamikita - `Mist` and provide default language, BPM, key, reference audio, rights clearance status, section map, default target section metadata, and recording-guide rights flags. Lyrics are optional in P0.
- The default P0 target section must be `intro` with timestamp `0:00-0:28` against the registered reference audio asset.
- P0 may expose only one selectable section, but the UI and API must still model song selection and section selection separately.
- The learner may edit BPM and key defaults when correction controls are enabled.
- The system must snapshot selected song id, section id, BPM/key, reference rights state, lyrics/reference display flags, and consent version before engine processing starts.

### FR-004 Recording, Take Review, And Audio Ingest

- The app must provide an in-app recorder as the primary input path for the user's own voice.
- The recorder must support mic permission request, count-in, section timer, input level meter, stop, replay own take, retake, and submit.
- The system may support `wav` and `mp3` fallback upload when recorder use is unavailable or internally enabled.
- If the app records in a platform-native format, `audio-ingest` is the authoritative normalization boundary. The app may pre-normalize when reliable, but the job must not reach `audio_ingest=succeeded` until the server/engine validates and normalizes the input.
- The system must reject unsupported formats with a clear reason.
- The system must convert accepted input to a standard mono WAV/PCM asset.
- The system must produce metadata including sample rate, channels, duration, loudness estimate, silence segments, speech/voice segments, and `audio_asset_id`.
- The system must record original duration and whether the submitted take matches the configured target section length tolerance.
- If the recording or fallback upload is longer than the P0 section input limit, the system must ask the user to trim, retake, or select the expected section instead of silently processing the full song.
- Silence, clipping, too-low volume, no-voice, too-short, and too-long checks should be surfaced during Take Review when feasible before processing.

### FR-005 Admin Song Package Management

- The system must allow admins to register reference/target songs for P0 internal operation.
- P0 admin song package registration must require title, artist, reference audio, language, BPM, key, full section map, target section start/end, source/provenance, rights clearance status, reference-audio usage status, reference pre-listen flag/scope, lyrics display flag/scope, lyrics sync flag, license expiry/re-review date, and deletion owner.
- Lyrics are optional and not required for P0.
- The system may support metadata ingestion from external music or lyrics providers, subject to licensing and API terms, but this must not block P0 manual registration.
- External providers must not be used to download, rip, cache, analyze, train on, remix, or derive audio unless a written license or provider-approved use path explicitly permits it.
- If source/provenance, rights clearance status, usage status, retention period, or deletion owner is missing, the song package must not be publishable to learners.
- If rights are uncertain or a complaint is received, the song package must enter `rights_blocked` and learner playback/processing must be disabled until reviewed.
- AI-assisted lyric/audio synchronization is P1 and must not block P0 pitch-first processing.
- Reference audio must be used by default only for analysis and comparison inside the app.
- Learner-facing reference pre-listen must be hidden unless `reference_prelisten_allowed=true`, a valid scope such as `section_prelisten` exists, and audit logging succeeds.
- Reference pre-listen must be section-limited, signed, app-only, no-download/share, and stopped before active microphone recording starts.
- Lyrics must be hidden unless `lyrics_display_allowed=true` and the display scope allows the selected section or line. `lyrics_sync_allowed=false` means the UI may not imply time-synced lyrics.
- Reference audio must not be exposed for download, sharing, model training, or third-party voice cloning.
- The system must track reference audio storage, retention, and deletion policy separately from the learner's own vocal recording.
- The song package pipeline should be reusable in future projects where non-admin users may be allowed to upload songs.

### FR-006 Safety Rights Check

- The system must verify that the user owns or is allowed to use the source audio asset.
- MVP must allow only the learner's own recorded or fallback-uploaded voice as the source vocal.
- The system must deny any request that attempts to use another person's voice, a celebrity/artist voice, a character voice, or an unverified target voice model.
- Each analysis or preview request must create an audit log with user id, source asset id, operation, decision, and policy reason.
- The system must collect separate consent records for own-voice upload, generated preview, expert review, candidate data use, and retention/deletion notice.
- Candidate data use must be opt-in and must not be bundled with required own-voice analysis consent.
- First-login required consent can satisfy job prerequisites only while consent version, policy document version, scope, and required/optional status remain current.
- Each submitted job must store an immutable consent snapshot id/hash. If any required consent is missing, withdrawn, expired, or out of date, recording submission and engine processing must be blocked until re-consent.
- Restrictions against recording/capture/redistribution/public posting must clearly apply to reference audio, lyrics, generated previews, and app output, not to the user's own voice recording step.

### FR-007 Target Section Pitch Extraction

- The system must extract F0 frames for the submitted target section practice recording.
- The system must mark voiced/unvoiced frames and confidence score.
- The system must convert confident F0 frames into MIDI note numbers.
- The system must compare detected pitch against target notes for the configured `Mist` target section.
- The system must use the Quantified Thresholds Draft for trusted, low-confidence, excluded, disputed, and review-needed pitch ranges.
- If the two target sources conflict, the system must use confidence-based handling, surface source attribution internally, and avoid overconfident scoring for disputed sections.
- Low-confidence pitch segments must be surfaced in the result screen and evaluation report.

### FR-008 Target Pitch Mapping

- The system must produce note sequence JSON with note start/end, target pitch, duration, and confidence.
- The system must preserve target source attribution: reference audio analysis, engine-derived note sequence, or manual/admin override.
- The system must support admin-provided BPM/key defaults and user correction only if correction controls are enabled.

### FR-009 Self-voice Section Preview

- The system must generate a corrected/generated self-voice section preview using the user's own voice only.
- The system must render the result for app-only playback.
- The render output must include clipping and loudness metadata.
- The preview must be clearly labeled as section-limited output and show the selected section label.
- Full-song preview generation is not required for P0.

### FR-010 Preview Evaluation And Failure Tagging

- The system must ask the primary question immediately after playback: "이 preview가 내 목소리처럼 들린다" on a 1-5 scale.
- The system must only show the rating prompt after `preview_played=true`.
- `preview_played=true` requires a real preview artifact, `pipeline_mode` in `partial_real` or `real_synthesis`, `mock_fixture_used=false`, `section_limited=true`, `playback_blocked=false`, a playback session id, successful playback URL audit, foreground unmuted playback, no severe playback error, and unique timeline coverage of at least 80% of preview duration.
- For the P0 `Mist intro` section, `preview_played=true` requires at least 22.4 seconds of distinct timeline coverage because the section is 28 seconds long.
- The app should send playback progress every 1 second during active foreground, unmuted playback, and must flush progress on pause, seek, end, background, mute, or error.
- Playback events must include event id, schema version, occurred-at timestamp, and session-scoped client sequence so the server can merge duplicate or out-of-order events.
- The server must compute `preview_played=true` from merged distinct played ranges. Client-reported total playback time alone must not count as metric eligibility.
- If the rating is below 4, the user must be asked to select one or more failure reason tags.
- P0 failure reason tags must include: `not_my_voice`, `not_song_like`, `pitch_wrong`, `timing_wrong`, `robotic_or_artifact`, `noise_or_clipping`, `too_short_or_incomplete`, `other`.
- Playback failure is not a preview quality rating tag. It must be captured as a separate `playback_problem_reported` event.
- Internal reviewers may add technical tags such as `pitch_extraction_low_confidence`, `target_note_conflict`, `synthesis_failure`, `render_failure`, or `rights_blocked`.
- The output must not be downloadable or externally shareable in P0 internal operation.

### FR-011 Job Status And Result UI

- The app must show job states: `created`, `queued`, `processing`, `preview_ready`, `completed`, `failed`, `failed_with_partial_artifacts`, `blocked`, `needs_review`.
- Completed jobs must put self-voice preview playback first.
- Completed jobs must show that the result is a section-limited preview and display the selected section label.
- Completed jobs must show current pitch vs target pitch, pitch match score or deviation, low-confidence pitch sections, rating prompt, failure reason tags if applicable, and basic quality report.
- Failed jobs must show stage, reason, and whether retry is allowed.
- Partial jobs must show which output exists, which output failed, and whether the user can still play a preview.
- Blocked jobs must show policy reason without exposing sensitive policy internals.

### FR-012 Preview Quality Decision

- The system must evaluate whether the generated self-voice preview is playable, complete or section-limited, clipped, or artifact-heavy.
- The system must expose preview quality status in user-facing language and detailed artifact metadata internally.
- The system must not mark a job as fully successful if the learning report succeeds but no self-voice preview or section-limited preview is available.
- If self-voice preview generation completely fails, the final user-facing job state must be `failed` or `failed_with_partial_artifacts`, not `completed`.

### FR-013 Pitch Feedback

- The system must show the user's detected pitch by time range or section.
- The system must show whether the detected pitch matches the target note, is sharp, is flat, or is low confidence.

### FR-014 Evaluation And Observability

- The system must store pitch deviation, clipping detection, preview rating, failure reason tags, and engine artifact comparison results.
- The system must store preview completion status, section coverage, and low-confidence reasons.
- Each engine stage must emit structured logs tied to job id.
- Quality reports must be available for internal review even if the user-facing version is simplified.
- P0 jobs must expose duration, per-stage processing time, failed stage, selected section id, section timestamp, and section coverage details for technical review.
- Operational logs must not contain raw audio, playable preview URLs, access tokens, refresh tokens, passwords, provider secrets, full lyrics, or sensitive free-text content.

### FR-015 Retention And Data Handling

- User vocal recordings, admin reference audio, generated previews, and analysis artifacts must be retained for at least 1 month during P0 internal operation unless a user/admin deletion request requires earlier deletion.
- Raw audio must not be used as the 1-year retained dataset.
- Non-audio analysis data, audit records, operational metadata, confidence scores, preview ratings, and failure reason tags may be retained for up to 1 year for debugging and evaluation.
- Retained non-audio data must preserve source provenance and consent status.
- Deletion and retention jobs must be observable enough for internal reviewers to confirm that raw audio is excluded from 1-year datasets.
- Deletion evidence must include artifact id, data class, deletion request source, retention deadline, deletion job id, deletion status, timestamp, and actor/service id without including raw audio or playable preview.
- If deletion fails, the artifact must be marked `deletion_failed`, rights-sensitive playback must be blocked, and platform/storage owner review must be required.
- Access to raw audio, generated preview, reference audio, failure reason tags, and analysis artifacts must follow the role-based access model in `Consent And Access Draft`.
- If consent is withdrawn, rights are blocked/under review/expired, or deletion is requested while a job is queued or processing, the system must block new engine requests and new playback URLs, then move the job or artifact to the appropriate `blocked`, `deletion_pending`, `deleted`, or `deletion_failed` state.

### FR-016 Job State Ownership Capability

- The system must have one canonical job state owner for P0 jobs.
- P0 may implement this as a bounded module inside an existing backend/worker service; a standalone Conversion Job Orchestrator service is not required for P0.
- BFF may expose app-facing GraphQL APIs for create job, upload initiation, job status, and result retrieval, but must not independently invent final job status.
- Engine workers must emit idempotent stage result events or equivalent stage result records with job id, stage, status, artifact pointer, error reason, confidence summary, and timing metadata.
- The canonical job state owner must derive final job status from stage records. If preview generation fails completely, pitch-only success must remain a partial artifact and must not become P0 success.

### Technical Design Notes

- A standalone Conversion Job Orchestrator remains the recommended long-term architecture once P0 validates the preview loop.
- Temporal, Step Functions, or an equivalent workflow engine can be evaluated after P0 if retry/replay, durable execution, and audit requirements grow.
- For P0, prefer the smallest maintainable implementation: a single backend-owned job state table/module plus idempotent stage result records.

## Acceptance Criteria

- Given a new user enters valid signup details and accepts terms, when they submit, then an account is created and the app receives auth tokens and user profile.
- Given required P0 consent was accepted at signup or first login, when the user starts a new job and the consent version/scope is still current, then the job stores a consent snapshot without forcing the full checklist again.
- Given a required consent version, policy document, or scope changed after the user's first-login consent, when the user attempts to record or submit a take, then the app blocks submission and requests re-consent.
- Given a user enters invalid email or password, when they submit signup/login, then validation errors are shown and no conversion job is created.
- Given an admin creates the Ken Kamikita - `Mist` P0 song package, when title, artist, language, BPM, key, reference audio, source/provenance, rights clearance status, usage status, section map, or target section start/end is missing, then the package cannot be published to learners.
- Given an admin creates the `Mist` P0 song package with required metadata, reference audio, source/provenance, rights clearance status, usage status, recording-guide rights flags, section map, and default target section `intro` at `0:00-0:28`, when validation passes, then learners can select the song and then select that section in the P0 app.
- Given reference audio rights are uncertain, blocked, expired, or missing review, when an admin tries to publish the song package, then the package enters `rights_blocked` and cannot be selected by learners.
- Given a rights complaint is received for a reference asset, when the policy owner marks the asset under review, then learner playback/processing for that asset and its generated previews is blocked until resolved.
- Given a learner selects a song but has not selected a section, when they try to record, then the app routes them to Section Selection first.
- Given a learner selects the `intro` section, when they enter the recorder, then the screen shows section label, timestamp, expected duration, and recording controls.
- Given `reference_prelisten_allowed=false` or the rights flag is expired, when the learner opens the recorder, then reference audio playback is hidden or disabled with a safe reason.
- Given `reference_prelisten_allowed=true`, when the learner plays the reference section, then playback is section-limited, signed, audited, app-only, no-download/share, and stops before active microphone recording starts.
- Given `lyrics_display_allowed=false`, when the learner opens the recorder, then full lyrics and line lyrics are hidden while section label/timestamp remain available.
- Given `lyrics_display_allowed=true` and scope includes the selected section, when the learner opens the recorder, then only the permitted section/line lyric cue is shown and no full lyrics are logged.
- Given microphone permission is denied or no microphone is available, when the learner enters recording, then the app explains the issue and may offer a fallback upload path if enabled.
- Given an authenticated user records a target section take or submits a supported fallback upload, when submission completes, then the system creates a P0 section preview job and an `audio_asset_id`.
- Given an admin registers reference audio, when target extraction runs, then the system uses the reference audio only for allowed app-internal analysis and rights-gated pre-listen, and does not expose it for download or sharing.
- Given a recorded take or fallback upload is longer than the configured P0 section tolerance, when validation runs, then the user is asked to trim, retake, or select the expected section instead of silently processing the full song.
- Given an unsupported fallback upload format is submitted, when validation runs, then the user sees a clear unsupported format message.
- Given the user does not confirm the source audio is their own voice, when they attempt to start analysis or preview generation, then the job is blocked before engine processing.
- Given the user consents to own-voice analysis but does not opt into candidate data use, when expert labels are created, then those labels are not included in future model-improvement candidate datasets.
- Given the user attempts to use another person's voice or an unauthorized target voice model, when Safety Rights runs, then the job is denied and an audit record is created.
- Given a valid target section practice recording with required metadata, when preview generation completes, then a corrected/generated self-voice section preview is available for app-only playback.
- Given preview generation fails completely but pitch feedback succeeds, when the job reaches final decision, then the user-facing job is `failed` or `failed_with_partial_artifacts`, and pitch artifacts remain available internally.
- Given a self-voice preview is generated, when the user opens the result screen, then preview playback is the primary action before detailed analytics.
- Given target notes are available from reference audio analysis or engine-derived note sequence, when pitch analysis completes, then the user can see current pitch, target pitch, and sharp/flat/on-target status for the selected target section.
- Given reference audio analysis and engine-derived note sequence conflict, when confidence or pitch disagreement crosses the Quantified Thresholds Draft, then the system labels the affected target range as low confidence, disputed, or review needed instead of forcing a target note.
- Given a user rates the preview below 4 on "이 preview가 내 목소리처럼 들린다", when the rating is submitted, then the user must select at least one failure reason tag or enter `other`.
- Given a user rates the preview 4 or higher, when the rating is submitted, then the response counts as success for the primary self-voice metric.
- Given a user opens a playable preview, when they only tap play briefly or replay the same short range, then `preview_played=true` is not set until unique timeline coverage reaches at least 80% of the preview duration.
- Given the `Mist intro` preview duration is 28 seconds, when the user hears at least 22.4 seconds of distinct timeline without severe playback error, foreground/mute/audit conditions are satisfied, and the preview is from `partial_real` or `real_synthesis`, then `preview_played=true` may be set.
- Given `preview_played=false`, when the user tries to submit the primary self-voice rating, then rating submission is blocked and the user may report a playback problem instead.
- Given playback progress events are duplicated, delayed, submitted out of order, or replayed with the same event id, when the server merges the events, then the server uses event id/schema validation and distinct played ranges, and does not inflate coverage from duplicate total time.
- Given a pipeline stage fails, when the user views the job, then the failed stage, plain-language reason, and retry guidance are shown.
- Given a completed P0 job, when internal reviewers inspect the artifacts, then preview completion status, pitch deviation, clipping status, rating, failure reason tags, and stage outputs are available.
- Given a P0 section job completes or fails, when internal reviewers inspect the artifacts, then duration, per-stage timing, failed stage, section id, section timestamp, and section coverage details are available.
- Given a raw audio artifact reaches its retention deadline, when the retention process runs, then raw audio is deleted or flagged for deletion and excluded from 1-year retained datasets.
- Given deletion succeeds, when internal reviewers inspect deletion evidence, then the evidence includes artifact id, data class, deletion request source, retention deadline, deletion job id, deletion status, timestamp, and actor/service id without raw audio or playable preview.
- Given deletion fails for a rights-sensitive artifact, when the deletion job reports failure, then playback is blocked and the artifact is marked `deletion_failed`.
- Given an engine worker emits a duplicate stage result, when the canonical job state owner receives it, then the job state remains idempotent and no duplicate user-facing completion is created.
- Given the app requests job status, when BFF serves the request, then BFF reads the canonical job projection rather than independently deciding final job status.
- Given mobile viewport `360 x 640`, when the user signs up, logs in, selects song/section, records a take, submits, and views a result, then primary actions remain reachable without horizontal scroll.

## P0 Readiness Criteria

- Build can start when the team registers Ken Kamikita - `Mist`, confirms the full section map, sets `intro` as `0:00-0:28`, and confirms required song package metadata.
- Build can start when the canonical P0 job state owner and the app-facing job status contract are agreed.
- Build can start when Safety Rights has a P0 policy for separated consent, first-login required consent reuse, job-level consent snapshots, blocked jobs, audit logging, and app-only playback.
- Build can start when reference audio source/provenance, rights clearance status, usage status, retention period, and deletion owner are recorded and reviewable.
- Build can start when reference pre-listen, lyrics display, and lyrics sync flags default to blocked and have explicit owner-controlled enablement rules.
- Build can start when the in-app recorder, take review, fallback upload decision, and mic permission handling are agreed.
- Build can start when self-voice preview evaluation uses a fixed primary question, for example: "이 preview가 내 목소리처럼 들린다" on a 1-5 scale, where 4 or higher counts as a successful response.
- Build can start when raw audio retention, generated preview retention, audit retention, and deletion ownership are documented.
- Build should not include provider automation as a P0 dependency unless licensing/API terms are already approved.

## Success Metrics

- Activation
  - Baseline: Unknown. No production usage data available.
  - Target: At least 5 of 10 learner participants create one practice analysis job within their first session, and both educator participants can open at least one reviewable result.
  - Guardrail: Signup/login failure due to client/backend integration errors below 2% of attempts.
- Job completion
  - Baseline: Unknown. Engine pipeline not yet fully implemented.
  - Target: At least 60% of valid P0 section jobs reach `completed` with `preview_available=true` and `section_limited=true`.
  - Guardrail: No unauthorized or non-self voice jobs reach analysis, preview, or synthesis.
- Self-voice preview usefulness
  - Baseline: Unknown.
  - Target: At least 5 of 10 learner participants rate the self-voice preview 4 or higher on the primary question, "이 preview가 내 목소리처럼 들린다". Any response below 4 does not count as success for this metric. No more than 2 learners should rate it 2 or lower.
  - Eligibility: Only ratings tied to a metric-eligible `preview_played=true` playback session count.
  - Guardrail: The result screen must not present a job as successful when preview generation failed completely.
- Time to preview
  - Baseline: Unknown.
  - Target: P0 internal operation prioritizes stable section-level success over speed. P50 10 minutes and P95 30 minutes are observation targets, not hard success gates.
  - Guardrail: Jobs over 60 minutes become timeout/failure candidates, and P95 processing time and failure reasons are visible internally for debugging.
- Pitch matching usefulness
  - Baseline: Unknown.
  - Target: At least 5 of 10 learner participants say the current-vs-target pitch feedback helps identify what to practice next.
  - Guardrail: Low-confidence pitch sections and disagreement between reference-audio target extraction and engine-derived note sequence must be clearly labeled and excluded from overconfident scores. Because `intro` is narration-heavy, speech-like pitch ranges must not be over-scored as singing pitch accuracy.
- Failure reason tagging quality
  - Baseline: Unknown.
  - Target: 100% of ratings below 4 include at least one failure reason tag or `other`.
  - Guardrail: Failure tags must separate user perception from technical diagnosis; user tags alone must not be treated as root cause.
- Quality diagnostics
  - Baseline: Unknown.
  - Target: 100% of completed jobs include preview status, pitch/timing/confidence reports, and clipping reports where preview output exists.
  - Guardrail: Evaluation artifact generation failure does not hide preview/render failure states.
- Safety and rights
  - Baseline: Unknown.
  - Target: 100% of analysis/preview requests produce an allow/deny decision and audit record.
  - Guardrail: If audit logging fails for rights-sensitive operations, the system fails closed.
- Section technical feasibility
  - Baseline: Unknown.
  - Target: P0 internal operation produces enough processing data to decide whether the `Mist` target section preview loop should continue, be shortened, or be redesigned.
  - Guardrail: The product must not imply full-song support during P0.
- Data retention compliance
  - Baseline: Unknown.
  - Target: 100% of raw audio artifacts follow the 30-day default retention policy, deletion jobs run within 24 hours after retention deadline, and raw audio is excluded from 1-year retained datasets.
  - Guardrail: No raw audio is retained for long-term model improvement without a separate approved consent and policy path.
- Recording funnel
  - Baseline: Unknown.
  - Target: song selected -> section selected -> recorder ready -> take submitted -> preview played -> rating submitted as separate funnel events.
  - Eligibility: `preview played` means the 80% unique timeline coverage threshold was reached for a non-mock `partial_real` or `real_synthesis` preview.
  - Guardrail: Reference playback unavailable, lyrics unavailable, or mic permission denied must not be silently counted as engine failure.

## Risks

- Preview quality risk: A technically complete pipeline may still produce self-voice previews that users find unnatural, robotic, off-pitch, or not recognizably their own voice.
- Pedagogy risk: Pitch/timing numbers may be technically correct but not actionable for music learners or teachers.
- Intro target risk: `intro` is narration-heavy, so it validates self-voice preview and app flow better than singing pitch matching or high-energy chorus synthesis.
- Failure-tag risk: User-selected failure tags may describe symptoms, not true technical root cause, so internal technical tagging is still needed.
- Reference audio risk: Using a real song as reference improves pitch target extraction but introduces copyright, provider-terms, storage, retention, and access-control risk.
- Rights clearance risk: If source/provenance, rights clearance status, usage status, or deletion owner is weak, even P0 internal operation can become unsafe to operate.
- Metadata provider risk: YouTube, Spotify, lyrics providers, and other external domains may have API, licensing, rate-limit, or lyrics availability constraints that affect song package automation.
- Training-data risk: Candidate labels or failure data can be misused as training data if consent boundaries and provenance are weak.
- Scope risk: Attempting full voice conversion, expression, custom voices, and full-track rendering in MVP will likely delay P0 validation.
- Safety risk: Voice rights and consent gaps can create product, legal, and trust risk even with internal participants.
- Delivery risk: Current app is auth UI only, while recorder/result screens and job orchestration are not yet visible in code.
- Integration risk: BFF currently exposes `me`; signup/login GraphQL mutations and upload/job APIs appear not yet implemented.
- Engine risk: Audio ingest is partially implemented, while many downstream engines are currently documented rather than implemented.
- Full-song risk: J-POP/utaite one-song inputs may exceed early engine assumptions around duration, chunking, memory, storage, and latency.
- Metric risk: There is no real baseline for section-level completion, self-voice preview usefulness, pitch matching usefulness, failure reason distribution, or activation.
- Recording UX risk: If mic permission, input meter, take review, or retake guidance is weak, users may fail before the engine receives a useful input.
- Reference leakage risk: If reference audio plays during active recording, the original song may leak into the user's take and weaken rights, quality, and voice-safety guarantees.
- UX risk: Users may expect polished full-song output, while P0 output may be section-limited or have quality caveats.

## Dependencies

- App: song selection, section selection, recorder/take review, fallback upload, job status screen, section result screen, app-only result playback UI, preview rating UI, failure reason tagging UI, contact follow-up gate UI when later enabled, token/session integration
- BFF: GraphQL mutations/queries for signup, login, admin song package registration, create project/job, upload initiation, job status, result retrieval, contact follow-up gate/preference endpoints disabled by default
- API Gateway: orchestration APIs for auth, song package, project/job, asset, conversion, and user context
- Job State Owner: canonical P0 job state, stage transition, retry, final decision, retention deadline, app-facing read model
- User Service: account, auth, user identity, role/status
- Storage: MinIO-backed S3-compatible object storage for source audio, reference audio, canonical wav, manifests, render outputs, internal-only artifacts, deletion evidence
- Queue/Eventing: NATS JetStream is a required P0 runtime dependency for audio/engine pipeline events. Redis/BullMQ may be used for app-facing async jobs.
- Engines: audio ingest, voice pitch, target pitch mapping, self-voice section preview, preview evaluation, safety rights
- Infra: PostgreSQL, Redis, NATS, local or object storage, Kubernetes deployments, structured logs
- Policy: terms, privacy policy, first-login required consent, job-level consent snapshot, own-voice consent, expert review consent, candidate data opt-in, contact follow-up gate policy, reference audio policy, lyrics display policy, audit retention, disallowed voice-use policy
- Music domain inputs: target song metadata, section map, provider/source metadata, lyrics handling, BPM/key source, reference audio handling, reference pre-listen rights flags, lyrics display/sync flags, target note extraction policy
- External data providers: YouTube/Spotify/music metadata domains, lyrics providers, and any licensed source required for metadata or lyric retrieval
- Music education expertise: section selection, pitch feedback interpretation, low-confidence cases, and learner-facing explanation copy

## Open Questions

### Product And User

- `Mist` section map의 timestamp를 현재 v0.10 기준으로 확정할 것인가, 아니면 reference audio asset 검수 후 조정할 것인가?
- 학습자용 화면과 교육자용 화면을 같은 결과 화면으로 시작할 것인가, 역할별로 다르게 보여줄 것인가?
- 관리자 song package의 P0 필수 metadata 외에 provider id, album/artwork 등을 언제부터 요구할 것인가?

### Safety And Policy

- 본인 음성 확인은 P0에서 separated consent로 시작하고, 이후 active voice verification이나 voice CAPTCHA 수준까지 확장할 필요가 있는가?
- reference audio source/provenance와 rights clearance status의 최종 승인자는 누구인가?
- 최소 1개월 보관 이후 reference audio와 사용자 녹음의 자동 삭제, 연장, 접근 권한 정책은 어떻게 둘 것인가?
- break-glass raw audio 접근은 누가 승인하고 어떤 incident 조건에서 허용할 것인가?
- audit log 보관 기간과 접근 권한 최종 owner는 누가 될 것인가?

### Quality And Metrics

- "이 preview가 내 목소리처럼 들린다" 평가 문항의 visual placement와 microcopy를 결과 화면에서 어떻게 다듬을 것인가?
- P0 confirmed failure tag 목록을 결과 화면에서 어떤 문구로 보여주고, `other` 자유입력의 개인정보/민감정보 유입을 어떻게 제한할 것인가?
- 결과가 나쁘더라도 job은 `completed`로 볼 것인가, quality threshold 미달이면 `failed` 또는 `needs_review`로 볼 것인가?

### Technical Scope

- NATS engine event envelope, StageResult normalization schema, and app-facing job projection schema를 어떤 versioning 정책으로 관리할 것인가?
- Redis/BullMQ를 app-facing async job에 사용할 경우, NATS engine pipeline과 중복 retry/timeout ownership이 생기지 않도록 어떤 boundary test를 둘 것인가?

### Launch

- 실패한 작업의 원본 오디오는 보관할 것인가, 즉시 삭제할 것인가?
- provider 자동화가 P0에서 제외될 경우, 관리자 수동 등록 운영 비용을 누가 감당할 것인가?

## Assumptions

- 이 PRD는 코드와 문서 기반의 제품 초안이며, 실제 고객 인터뷰나 analytics baseline은 아직 반영되지 않았다.
- MVP의 장기 제품 목표는 J-POP/우타이테 노래 한 곡 기준이지만, P0 내부 운영은 `Mist` target section으로 제한한다.
- 주요 사용자는 음악 학습자와 음악 교육자다.
- P0 내부 운영 사용자 규모는 학습자 10명과 교육자 2명이다.
- P0 내부 운영은 Ken Kamikita - `Mist` section map, `intro` default target section, 관리자 수동 song package 등록, pitch-first processing, section-first preview를 기준으로 build한다.
- 외부 provider 기반 metadata/lyrics 자동화, lyrics sync, full-song merge는 P1 실험 범위로 둔다.
- 사용자 본인 음성만 허용한다.
- 1차 가치는 내 목소리가 보정/생성된 노래를 들어보는 self-voice preview다.
- self-voice preview의 P0 primary rating은 "내 목소리처럼 들림"이다.
- P0 내부 운영에서는 구간 단위 preview/분석 성공도 P0 success로 인정할 수 있다.
- 2차 가치는 현재 내 목소리 음정과 노래에 맞는 목표 음정의 비교다.
- 목표 음정은 원곡 음원 분석과 엔진 추정 note sequence를 함께 사용한다.
- 목표 음정 source 간 충돌은 confidence 기반으로 처리하고, 신뢰도 낮은 구간은 강제 판정하지 않는다.
- P0 내부 운영의 원곡/목표곡 reference audio는 관리자가 song package로 등록한다.
- 관리자 song package는 곡명/아티스트, 원곡/reference audio, 언어, 가사, BPM, key, section map, source/provider metadata, rights clearance status를 포함하는 방향으로 설계한다.
- 곡 metadata와 가사는 외부 provider 또는 AI 보조 수집을 사용할 수 있지만, provider/API/license 검토 후 확정한다.
- 3차 가치인 진성, 비성, 두성 등 발성/공명 유형 후보 확인은 P0에서 제외하고 P1 이후 실험으로 검토한다.
- 교사/전문가 검토 라벨은 P0 필수 데이터가 아니며, P1 이후 candidate data로 저장할 때 consent, provenance, confidence, inter-rater agreement를 함께 기록한다.
- 일본어 가사/음절 정렬과 lyrics sync는 P1 실험 범위로 두고, P0는 pitch 중심 처리를 우선한다.
- P0 내부 운영 결과물은 다운로드하지 않고 앱 안에서만 재생한다.
- 보관 기간은 최소 1개월을 기준으로 한다. raw audio는 1년 보관 대상에서 제외하고, raw audio 없는 분석/audit/label 데이터만 최대 1년 보관 후보로 둔다.
- P0 내부 운영은 처리 속도보다 안정적 구간 성공을 우선한다. 추후에는 안정성과 속도를 모두 최적화한다.
- 출시 형태는 P0 내부 운영이다.
- Safety Rights는 feature가 아니라 launch gate로 취급한다.
- 앱은 모바일 first이지만, Expo Web에서도 핵심 flow를 유지한다.
- 현재 구현 상태를 기준으로 auth 연동, recorder/upload job API, section result UI, canonical P0 job state owner, minimal engine path가 주요 신규 개발 범위다.
- PRD 승인 전에는 section map 검수, reference audio rights clearance, self-voice preview 평가 문항 UI, failure reason tag 목록, song package 필수 metadata, 저장/삭제 정책을 반드시 확정해야 한다. P0 job state owner는 worker `conversion-job-state` bounded module로 결정했다.
