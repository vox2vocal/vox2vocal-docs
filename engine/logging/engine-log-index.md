# Engine Log Event Index

이 문서는 각 엔진을 담당하는 에이전트가 구현해야 할 로그 이벤트 인덱스다.

각 엔진의 `README.md` 또는 `AGENT.md`에는 이 문서의 해당 섹션을 기준으로 필수 event를 반영한다.

## 공통 필수 이벤트

모든 엔진은 아래 event를 가진다.

| Event | Level | 설명 |
| --- | --- | --- |
| `<engine>.startup.started` | INFO | 프로세스 시작 |
| `<engine>.startup.completed` | INFO | 설정 로드 및 의존성 준비 완료 |
| `<engine>.startup.failed` | CRITICAL | 시작 실패 |
| `<engine>.health.checked` | DEBUG/INFO | health check |
| `<engine>.dependency.connected` | INFO | NATS, storage, DB, model runtime 연결 |
| `<engine>.dependency.failed` | ERROR/CRITICAL | 필수 dependency 연결 실패 |
| `<domain>.<capability>.started` | INFO | 작업 시작 |
| `<domain>.<capability>.completed` | INFO | 작업 완료 |
| `<domain>.<capability>.failed` | ERROR | 작업 실패 |

오류 event 필수 필드:

```txt
error_code
error_type
retryable
duration_ms
trace_id
job_id
audio_asset_id
```

## engine-audio-ingest

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `audio.ingest.requested` | INFO | optional | 요청 이벤트 수신 |
| `audio.ingest.source.resolved` | INFO | no | source object key 해석 완료 |
| `audio.ingest.probe.started` | INFO | no | ffprobe 시작 |
| `audio.ingest.probe.completed` | INFO | no | format/codec/duration 추출 |
| `audio.ingest.probe.failed` | ERROR | no | ffprobe 실패 |
| `audio.ingest.conversion.started` | INFO | no | canonical WAV 변환 시작 |
| `audio.ingest.conversion.completed` | INFO | no | canonical WAV 변환 완료 |
| `audio.ingest.quality.warned` | WARN | no | clipping, low volume, silence-only |
| `audio.ingest.manifest.written` | INFO | no | manifest 저장 완료 |
| `audio.ingest.completed` | INFO | optional | completed event 발행 |
| `audio.ingest.failed` | ERROR | optional | failed event 발행 |

## engine-voice-analysis

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `voice.analysis.started` | INFO | no | 분석 시작 |
| `voice.analysis.segment.detected` | INFO | no | active segment 탐지 |
| `voice.analysis.energy.completed` | INFO | no | energy curve 생성 |
| `voice.analysis.speech_rate.completed` | INFO | no | 발화 속도 추정 |
| `voice.analysis.quality.warned` | WARN | no | 무음/저에너지/과도한 잡음 |
| `voice.analysis.completed` | INFO | no | 분석 완료 |
| `voice.analysis.failed` | ERROR | no | 분석 실패 |

## engine-voice-pitch

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `voice.pitch.extraction.started` | INFO | no | F0 추출 시작 |
| `voice.pitch.extraction.completed` | INFO | no | F0 frame 생성 완료 |
| `voice.pitch.confidence.warned` | WARN | no | low confidence frame 비율 높음 |
| `voice.pitch.smoothing.completed` | INFO | no | pitch contour smoothing 완료 |
| `voice.pitch.note_candidate.completed` | INFO | no | note candidate 생성 |
| `voice.pitch.completed` | INFO | no | 피치 분석 완료 |
| `voice.pitch.failed` | ERROR | no | 피치 분석 실패 |

## engine-phoneme-alignment

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `phoneme.alignment.started` | INFO | no | 정렬 시작 |
| `phoneme.alignment.text.normalized` | INFO | no | 텍스트 정규화 |
| `phoneme.alignment.g2p.completed` | INFO | no | grapheme-to-phoneme 완료 |
| `phoneme.alignment.completed` | INFO | no | 음절/음소 정렬 완료 |
| `phoneme.alignment.confidence.warned` | WARN | no | 낮은 confidence |
| `phoneme.alignment.failed` | ERROR | no | 정렬 실패 |

## engine-rhythm-timing

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `rhythm.timing.started` | INFO | no | 타이밍 보정 시작 |
| `rhythm.timing.grid.generated` | INFO | no | BPM grid 생성 |
| `rhythm.timing.quantized` | INFO | no | 음절 시작점 quantize |
| `rhythm.timing.stretch.warned` | WARN | no | 과도한 stretch 감지 |
| `rhythm.timing.completed` | INFO | no | 타이밍 보정 완료 |
| `rhythm.timing.failed` | ERROR | no | 타이밍 보정 실패 |

## engine-melody-mapping

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `melody.mapping.started` | INFO | no | 멜로디 매핑 시작 |
| `melody.mapping.note.segmented` | INFO | no | pitch contour note 분절 |
| `melody.mapping.scale.corrected` | INFO | no | key/scale 보정 |
| `melody.mapping.reference.applied` | INFO | no | reference melody 적용 |
| `melody.mapping.completed` | INFO | no | note sequence 생성 완료 |
| `melody.mapping.failed` | ERROR | no | 매핑 실패 |

## engine-singing-synthesis

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `singing.synthesis.started` | INFO | optional | 합성 시작 |
| `singing.synthesis.model.loaded` | INFO | no | 모델 로드 |
| `singing.synthesis.feature.generated` | INFO | no | acoustic feature 생성 |
| `singing.synthesis.quality.warned` | WARN | no | low confidence 또는 artifact 위험 |
| `singing.synthesis.completed` | INFO | optional | 합성 완료 |
| `singing.synthesis.failed` | ERROR | optional | 합성 실패 |

## engine-vocoder-render

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `vocoder.render.started` | INFO | no | 렌더 시작 |
| `vocoder.render.model.loaded` | INFO | no | vocoder 로드 |
| `vocoder.render.waveform.generated` | INFO | no | waveform 생성 |
| `vocoder.render.clipping.warned` | WARN | no | clipping 감지 |
| `vocoder.render.completed` | INFO | no | 렌더 완료 |
| `vocoder.render.failed` | ERROR | no | 렌더 실패 |

## engine-expression

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `expression.control.started` | INFO | no | 표현 제어 시작 |
| `expression.vibrato.generated` | INFO | no | vibrato instruction 생성 |
| `expression.dynamics.generated` | INFO | no | dynamics curve 생성 |
| `expression.breath.generated` | INFO | no | breath placement 생성 |
| `expression.control.completed` | INFO | no | 표현 제어 완료 |
| `expression.control.failed` | ERROR | no | 표현 제어 실패 |

## engine-mix-master

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `mix.master.started` | INFO | no | 믹스/마스터 시작 |
| `mix.master.eq.applied` | INFO | no | EQ 적용 |
| `mix.master.compressor.applied` | INFO | no | compressor 적용 |
| `mix.master.limiter.applied` | INFO | no | limiter 적용 |
| `mix.master.loudness.completed` | INFO | no | loudness 측정 |
| `mix.master.completed` | INFO | optional | 최종 오디오 생성 |
| `mix.master.failed` | ERROR | optional | 믹스 실패 |

## engine-evaluation

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `evaluation.started` | INFO | no | 평가 시작 |
| `evaluation.pitch.completed` | INFO | no | pitch deviation 측정 |
| `evaluation.timing.completed` | INFO | no | timing deviation 측정 |
| `evaluation.artifact.completed` | INFO | no | artifact 검사 |
| `evaluation.quality.warned` | WARN | no | 품질 기준 미달 |
| `evaluation.completed` | INFO | no | 평가 완료 |
| `evaluation.failed` | ERROR | no | 평가 실패 |

## engine-safety-rights

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `rights.check.requested` | INFO | yes | 권한 판단 요청 |
| `rights.voice.ownership.verified` | INFO | yes | 보이스 소유권 확인 |
| `rights.consent.verified` | INFO | yes | 동의 확인 |
| `rights.decision.allowed` | INFO | yes | 허용 판단 |
| `rights.decision.denied` | WARN | yes | 거부 판단 |
| `rights.policy.changed` | WARN | yes | 정책 변경 |
| `rights.check.failed` | ERROR | yes | 권한 판단 실패 |

## engine-voice-conversion

| Event | Level | Audit | 설명 |
| --- | --- | --- | --- |
| `voice.conversion.requested` | INFO | yes | 변환 요청 |
| `voice.conversion.rights.checked` | INFO | yes | 권한 확인 완료 |
| `voice.conversion.started` | INFO | yes | 변환 시작 |
| `voice.conversion.model.loaded` | INFO | no | target model 로드 |
| `voice.conversion.feature.generated` | INFO | no | converted feature 생성 |
| `voice.conversion.similarity.warned` | WARN | optional | speaker similarity 경고 |
| `voice.conversion.completed` | INFO | yes | 변환 완료 |
| `voice.conversion.failed` | ERROR | yes | 변환 실패 |

## 엔진별 문서 반영 규칙

새 엔진 또는 새 event를 추가하면 아래를 갱신한다.

1. 이 문서의 해당 엔진 섹션
2. 해당 엔진 저장소의 `README.md`
3. 해당 엔진 저장소의 `AGENT.md`
4. audit event가 필요하면 [Audit Data Guide](./audit-data-guide.md)
5. 운영 알림이 필요하면 [Operations Guide](./operations-guide.md)
