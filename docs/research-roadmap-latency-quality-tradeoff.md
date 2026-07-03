# Research Roadmap: Latency-Quality Tradeoff in Realtime Voice Agents

이 문서는 이 저장소를 기반으로 "실시간 음성 에이전트의 latency-quality tradeoff를 체계적으로 측정하고 개선하는 연구"를 진행하기 위한 공동 작업 계획이다.

목표는 단순히 데모를 실행하는 것이 아니라, 재현 가능한 실험 플랫폼을 만들고, 여러 시스템 설계 선택이 지연시간, 품질, 비용, 사용자 경험에 어떤 영향을 주는지 분석해 논문으로 정리하는 것이다.

## 1. 연구 목표

### 1.1 핵심 질문

실시간 음성 에이전트에서 응답을 빠르게 시작하려면 STT, LLM, TTS를 가능한 한 빨리 연결해야 한다. 하지만 너무 빨리 연결하면 잘린 발화, 부정확한 인식, 어색한 TTS chunking, 불완전한 tool call, 낮은 답변 품질이 생길 수 있다.

이 연구의 핵심 질문은 다음과 같다.

```text
How do design choices in endpointing, buffering, model selection, and streaming orchestration affect the latency-quality tradeoff of realtime voice agents?
```

한국어로 풀면:

```text
실시간 음성 에이전트에서 endpointing, sentence buffering, 모델 선택, 스트리밍 오케스트레이션 방식이 응답 지연시간과 대화 품질 사이의 균형에 어떤 영향을 주는가?
```

### 1.2 우리가 만들 최종 결과물

- 브라우저에서 마이크로 대화 가능한 baseline realtime voice agent
- 실험 조건을 설정 파일로 바꾸며 반복 실행할 수 있는 benchmark runner
- turn 단위 latency/event log
- STT, LLM, TTS, end-to-end latency 통계
- 답변 품질, task success, interruption handling, cost 지표
- 실험 결과 CSV/JSONL
- 분석 notebook 또는 script
- 논문용 표와 그림
- 최종 논문 초안

## 2. 현재 레포의 역할과 한계

### 2.1 이 레포가 이미 제공하는 것

현재 저장소는 다음 구성 요소를 튜토리얼 형태로 제공한다.

- `chapters/01_streaming_stt`: Deepgram 기반 speech-to-text 예제
- `chapters/02_streaming_llm`: OpenAI-compatible LLM streaming 예제
- `chapters/03_streaming_tts`: ElevenLabs 기반 text-to-speech 예제
- `chapters/04_streaming_pipeline`: LLM token stream을 sentence buffer로 묶어 TTS에 넘기는 pipeline
- `chapters/05_websocket_server`: 브라우저와 연결되는 FastAPI WebSocket server
- `chapters/07_web_client`: 마이크 입력과 음성 재생을 담당하는 browser client
- `chapters/08_enterprise_agent`: function calling과 mock enterprise tools
- `chapters/09_production`: 기본 latency benchmark

### 2.2 연구 플랫폼으로 쓰기 전의 문제

그대로는 논문용 실험 플랫폼으로 부족하다.

- 일부 README가 실제 파일과 맞지 않는다.
- `agent_demo.py`, `server_with_vad.py`처럼 문서에 언급된 파일이 현재 없다.
- benchmark가 단발성이고 실험 조건 관리가 약하다.
- streaming STT, endpointing, interruption, tool call 성능을 통합적으로 기록하지 않는다.
- 품질 평가 지표가 없다.
- 실험 결과가 재현 가능한 형식으로 저장되지 않는다.
- baseline과 ablation 실험 구분이 없다.
- 논문용 figure/table 생성 파이프라인이 없다.

따라서 첫 단계는 "튜토리얼 코드"를 "실험 가능한 시스템"으로 바꾸는 것이다.

## 3. 연구 범위

### 3.1 연구할 시스템 경로

기본 pipeline은 아래와 같다.

```text
Browser microphone
  -> WebSocket audio stream
  -> STT
  -> endpointing / turn detection
  -> LLM response generation
  -> text buffering
  -> TTS
  -> WebSocket audio stream
  -> Browser speaker
```

### 3.2 주요 독립 변수

우리가 바꿔가며 실험할 설계 선택은 다음과 같다.

- Endpointing strategy
  - fixed silence threshold
  - VAD-based endpointing
  - streaming STT final transcript based endpointing
  - semantic endpointing

- Text buffering strategy
  - full response buffering
  - sentence-level buffering
  - phrase-level buffering
  - adaptive buffering

- LLM backend
  - OpenAI API
  - local vLLM
  - smaller model vs larger model

- TTS strategy
  - full response TTS
  - sentence-level TTS
  - phrase-level TTS
  - streaming TTS

- Conversation setting
  - simple QA
  - appointment booking with tool calls
  - interruption-heavy conversation
  - noisy audio
  - English vs Korean

### 3.3 주요 종속 변수

각 조건에서 측정할 결과는 다음과 같다.

- Latency
  - STT partial latency
  - STT final latency
  - endpoint detection latency
  - LLM time-to-first-token
  - first complete text chunk time
  - TTS time-to-first-byte
  - time-to-first-audio
  - end-to-end response completion time
  - interruption recovery time

- Quality
  - STT word error rate
  - answer relevance
  - task success rate
  - tool call correctness
  - conversation naturalness
  - TTS chunk smoothness
  - interruption handling success

- Efficiency
  - token count
  - TTS character count
  - API cost per turn
  - CPU/GPU usage
  - memory usage

## 4. 공동 작업 원칙

### 4.1 사용자의 역할

사용자는 연구 방향과 판단을 맡는다.

- 연구 질문을 좁힌다.
- 어떤 도메인으로 실험할지 정한다.
- 영어 중심으로 할지 한국어를 포함할지 정한다.
- 사용할 API key와 모델 접근 권한을 준비한다.
- 실험 결과가 논문 주장으로 충분한지 판단한다.
- 논문 기여점과 스토리를 함께 결정한다.

### 4.2 Codex의 역할

Codex는 구현, 실험 자동화, 분석 보조를 맡는다.

- 레포 구조를 정리한다.
- 실행 가능한 baseline을 만든다.
- instrumentation과 logging을 추가한다.
- benchmark runner를 만든다.
- 실험 설정 파일을 만든다.
- 결과를 CSV/JSONL로 저장한다.
- 분석 script를 만든다.
- 표와 그래프 생성을 자동화한다.
- 논문 초안의 기술 섹션, 실험 섹션, appendix 초안을 작성한다.

### 4.3 함께 결정해야 하는 것

아래 항목은 초기에 함께 정해야 한다.

- 주 연구 대상 언어: English only, Korean only, bilingual
- 주 도메인: medical appointment, customer support, general assistant
- 목표 venue: workshop, domestic conference, international conference, arXiv technical report
- 실험 budget: API 비용, GPU 사용 가능 여부, 실험 반복 횟수
- human evaluation 가능 여부

## 5. 전체 로드맵

### Phase 0. 저장소 안정화

목표: 현재 튜토리얼 코드를 신뢰 가능한 출발점으로 만든다.

작업:

- README와 실제 파일 목록의 불일치를 정리한다.
- missing demo file 여부를 문서화한다.
- Python 가상환경 실행 절차를 검증한다.
- `credentials.env` 로딩 방식을 정리한다.
- 최소 실행 command를 확정한다.
- dependency version을 고정할지 결정한다.

완료 기준:

- 새 개발자가 README만 보고 baseline을 실행할 수 있다.
- 실행 불가능한 문서 지시가 제거되거나 별도 TODO로 표시된다.
- 환경 변수 누락 시 친절한 에러가 나온다.

### Phase 1. Baseline voice agent 완성

목표: 브라우저에서 실제로 말하고 음성 응답을 듣는 최소 baseline을 완성한다.

작업:

- `chapters/05_websocket_server/server.py`가 `chapters/07_web_client` 정적 파일을 서빙하게 만든다.
- `http://localhost:8888`에서 web client가 열리도록 한다.
- `ws://localhost:8888/ws/audio` 연결을 검증한다.
- 마이크 입력, 서버 수신, STT, LLM, TTS, browser playback을 end-to-end로 확인한다.
- 에러 메시지를 UI와 server log 양쪽에 남긴다.
- 기본 대화 prompt를 실험용으로 분리한다.

완료 기준:

- `python chapters/05_websocket_server/server.py --port 8888` 실행 후 브라우저에서 대화가 가능하다.
- 한 turn의 audio input부터 audio output까지 성공한다.
- 실패 시 어떤 component에서 실패했는지 알 수 있다.

### Phase 2. Event logging 추가

목표: 한 turn 안에서 일어나는 모든 핵심 이벤트를 timestamp로 남긴다.

작업:

- turn ID와 session ID를 추가한다.
- server-side event logger를 만든다.
- browser-side event를 server로 전달하거나 local log로 저장한다.
- 각 event에 monotonic timestamp를 붙인다.
- logs를 JSONL로 저장한다.

필수 event:

```text
session_start
mic_stream_start
audio_chunk_received
speech_start_detected
speech_end_detected
stt_request_start
stt_partial_received
stt_final_received
llm_request_start
llm_first_token
llm_sentence_ready
tts_request_start
tts_first_audio
audio_chunk_sent
client_first_audio_played
agent_done
interruption_detected
interruption_recovered
error
```

권장 log schema:

```json
{
  "run_id": "2026-07-03T12-00-00_baseline",
  "session_id": "session-001",
  "turn_id": "turn-003",
  "condition_id": "sentence_buffer_openai_fixed700",
  "event": "llm_first_token",
  "timestamp_ms": 123456.78,
  "relative_ms": 842.31,
  "component": "llm",
  "metadata": {
    "model": "gpt-4.1-mini"
  }
}
```

완료 기준:

- 한 대화 turn을 replay하지 않아도 log만 보고 병목을 알 수 있다.
- latency 계산이 수동 stopwatch 없이 가능하다.

### Phase 3. Benchmark runner 구축

목표: 실험 조건을 설정 파일로 바꿔가며 반복 실행할 수 있게 한다.

작업:

- `experiments/configs/*.yaml` 구조를 만든다.
- 실험 runner를 만든다.
- predefined audio prompts를 읽어 자동으로 STT/LLM/TTS pipeline을 실행한다.
- 각 조건을 `n`회 반복한다.
- 결과를 `experiments/results/<run_id>/`에 저장한다.

예상 config:

```yaml
condition_id: sentence_buffer_openai_fixed700
language: en
domain: appointment_booking
stt:
  provider: deepgram
  model: nova-3
endpointing:
  strategy: fixed_silence
  silence_ms: 700
llm:
  provider: openai
  model: gpt-4.1-mini
  temperature: 0.2
buffering:
  strategy: sentence
  min_chars: 10
tts:
  provider: elevenlabs
  model: eleven_turbo_v2_5
  voice_id: JBFqnCBsd6RMkjVDRZzb
evaluation:
  repeats: 10
```

완료 기준:

- 실험 조건 하나를 command 한 번으로 실행할 수 있다.
- 결과 파일과 로그 파일이 자동 생성된다.
- 실험 실패도 결과에 기록된다.

### Phase 4. Dataset과 scenario 설계

목표: 실험이 우연한 한두 문장에 의존하지 않도록 평가 입력을 만든다.

작업:

- 실험용 utterance set을 만든다.
- clean audio와 noisy audio를 구분한다.
- short, medium, long utterance를 나눈다.
- simple QA와 task-oriented dialogue를 나눈다.
- interruption scenario를 별도로 만든다.
- 가능하면 reference transcript와 expected task outcome을 만든다.

권장 scenario:

- Simple response
  - "What are your clinic hours?"
  - "Can you explain what a follow-up appointment is?"

- Appointment booking
  - "I want to schedule an appointment with Dr. Johnson next Monday."
  - "Please book the 2 PM slot."

- Repair after STT error
  - "No, I said Johnson, not Jackson."

- Interruption
  - User starts speaking while the agent is still talking.

- Korean
  - "다음 주 월요일 오후에 진료 예약하고 싶어요."
  - "아니요, 김민수가 아니라 이민수예요."

완료 기준:

- 각 utterance에 ID, language, domain, difficulty, expected outcome이 있다.
- 최소 30개 이상의 turn-level test cases를 확보한다.
- 논문 실험에 쓸 최종 subset이 명확하다.

### Phase 5. Latency metric 정의와 계산

목표: 논문에서 사용할 latency metric을 엄밀히 정의한다.

주요 metric:

```text
STT Final Latency =
  stt_final_received - speech_end_detected

Endpointing Delay =
  speech_end_detected - true_user_speech_end

LLM TTFT =
  llm_first_token - llm_request_start

Text Chunk Ready Time =
  llm_sentence_ready - llm_request_start

TTS TTFB =
  tts_first_audio - tts_request_start

Server TTFA =
  audio_chunk_sent(first) - speech_end_detected

Client TTFA =
  client_first_audio_played - true_user_speech_end

End-to-End Completion Time =
  agent_done - true_user_speech_end

Interruption Recovery Time =
  new_user_speech_start - agent_audio_stopped
```

주의할 점:

- `speech_end_detected`와 실제 발화 종료 시점은 다르다.
- 논문에는 server-side latency와 user-perceived latency를 구분해야 한다.
- cloud API latency는 네트워크 상태의 영향을 받으므로 반복 실험과 분산 보고가 필요하다.

완료 기준:

- 모든 latency metric을 script로 계산한다.
- 평균뿐 아니라 p50, p90, p95, min, max를 저장한다.

### Phase 6. Quality metric 정의와 평가

목표: 빠른 시스템이 정말 좋은 시스템인지 판단할 quality metric을 만든다.

자동 평가:

- STT WER
- transcript completeness
- LLM response length
- tool call success
- task completion success
- interruption success
- response constraint violation
- TTS chunk count
- audio underrun count

LLM-as-judge 평가:

- relevance
- completeness
- conciseness
- conversational naturalness
- task helpfulness

Human evaluation 가능 시:

- perceived responsiveness
- naturalness
- frustration score
- preference between systems

권장 5점 척도:

```text
1 = unusable
2 = poor
3 = acceptable
4 = good
5 = excellent
```

완료 기준:

- 각 실험 turn마다 최소 하나 이상의 quality score가 있다.
- latency와 quality를 같은 `condition_id`로 join할 수 있다.

### Phase 7. 실험군 설계

목표: 논문에 들어갈 main experiment와 ablation을 설계한다.

#### Experiment A. Buffering strategy 비교

조건:

- full response buffering
- sentence-level buffering
- phrase-level buffering
- adaptive buffering

측정:

- TTFA
- completion latency
- TTS naturalness
- response coherence
- chunk boundary artifacts

가설:

```text
Phrase-level buffering reduces TTFA but may reduce naturalness.
Sentence-level buffering offers a stronger latency-quality balance.
Adaptive buffering can approach phrase-level latency while preserving sentence-level quality.
```

#### Experiment B. Endpointing strategy 비교

조건:

- fixed silence 300ms
- fixed silence 700ms
- fixed silence 1200ms
- VAD-based
- semantic endpointing

측정:

- endpointing delay
- false endpoint rate
- barge-in handling
- STT final accuracy
- user-perceived latency

가설:

```text
Short silence thresholds reduce latency but increase false turns.
Semantic endpointing can reduce premature cutoffs while avoiding long silence waits.
```

#### Experiment C. Model backend 비교

조건:

- OpenAI API small model
- OpenAI API larger model
- local vLLM small model
- local vLLM larger model

측정:

- LLM TTFT
- answer quality
- cost
- deployment complexity

가설:

```text
Local models can reduce variance and cost at scale but may require careful hardware tuning to match cloud quality.
```

#### Experiment D. Tool-calling voice agent

조건:

- no tool
- one tool call
- multi-step tool chain
- tool call with correction

측정:

- task success
- tool call correctness
- additional latency per tool call
- user-perceived delay

가설:

```text
Tool calls improve task success but introduce latency spikes that require conversational masking or partial verbal acknowledgments.
```

#### Experiment E. Korean and bilingual robustness

조건:

- English
- Korean
- code-switching

측정:

- STT WER/CER
- endpointing reliability
- LLM response quality
- TTS naturalness

가설:

```text
Realtime voice agent latency-quality behavior differs by language because STT finalization, punctuation, and sentence boundary detection are language-dependent.
```

완료 기준:

- 최소 2개의 main experiment와 1개의 ablation을 논문에 넣을 수 있다.
- 각 실험은 hypothesis, condition, metric, result table을 가진다.

### Phase 8. 개선 방법 개발

목표: 단순 측정 논문을 넘어서 개선 기여를 만든다.

가능한 개선 아이디어:

- Adaptive text buffering
  - 첫 chunk는 빠르게, 이후 chunk는 문장 단위로 안정화한다.
  - punctuation, token rate, expected TTS duration을 고려한다.

- Latency-aware endpointing
  - silence duration만 보지 않고 partial transcript 안정성도 본다.
  - 사용자가 질문을 끝냈는지 semantic classifier로 판단한다.

- Tool latency masking
  - tool call이 필요한 경우 "Let me check that" 같은 짧은 acknowledgement를 먼저 말한다.
  - 사용자가 체감하는 침묵 시간을 줄인다.

- Dynamic model routing
  - 쉬운 turn은 빠른 모델, 어려운 turn은 큰 모델로 보낸다.
  - latency budget에 따라 모델을 선택한다.

- Korean-aware sentence buffering
  - 영어 약어 중심 sentence buffer를 한국어 종결 어미와 punctuation에 맞게 확장한다.

권장 첫 개선안:

```text
Adaptive buffering for realtime TTS:
Start with low-latency phrase-level emission for the first response chunk,
then switch to sentence-level emission once the user has heard the agent begin speaking.
```

이유:

- 현재 `SentenceBuffer` 구조를 확장하기 쉽다.
- latency와 naturalness tradeoff를 직접 다룬다.
- 논문 실험으로 보여주기 좋다.
- API provider와 독립적인 시스템 기여가 될 수 있다.

완료 기준:

- baseline보다 TTFA가 줄어든다.
- full phrase-level 방식보다 naturalness 저하가 작다.
- ablation으로 개선 원인을 설명할 수 있다.

## 6. 코드 구조 제안

연구 플랫폼으로 바꿀 때 권장 구조는 다음과 같다.

```text
docs/
  research-roadmap-latency-quality-tradeoff.md

experiments/
  configs/
    baseline_sentence.yaml
    full_response.yaml
    phrase_buffer.yaml
    adaptive_buffer.yaml
  data/
    prompts_en.jsonl
    prompts_ko.jsonl
    audio/
  results/
    <run_id>/
      events.jsonl
      turns.jsonl
      metrics.csv
      summary.json
  scripts/
    run_benchmark.py
    compute_metrics.py
    plot_results.py

voice_agent/
  server.py
  pipeline.py
  logging.py
  metrics.py
  endpointing.py
  buffering.py
  evaluation.py
  providers/
    stt_deepgram.py
    llm_openai.py
    llm_vllm.py
    tts_elevenlabs.py
```

처음부터 대규모 리팩터링을 하지 말고, baseline 실행이 검증된 뒤 단계적으로 옮긴다.

## 7. 결과 파일 설계

### 7.1 `turns.jsonl`

각 turn의 입력, 출력, 조건, 품질 평가를 저장한다.

```json
{
  "run_id": "run_001",
  "condition_id": "sentence_buffer_openai_fixed700",
  "session_id": "s001",
  "turn_id": "t001",
  "prompt_id": "appt_en_001",
  "language": "en",
  "domain": "appointment",
  "reference_transcript": "I want to schedule an appointment with Dr. Johnson.",
  "stt_transcript": "I want to schedule an appointment with Dr. Johnson.",
  "agent_response": "I can help with that. What date works best for you?",
  "tool_calls": [],
  "success": true,
  "error": null
}
```

### 7.2 `metrics.csv`

각 turn의 metric을 wide format으로 저장한다.

```csv
run_id,condition_id,turn_id,stt_final_ms,llm_ttft_ms,tts_ttfb_ms,server_ttfa_ms,client_ttfa_ms,total_ms,wer,task_success,quality_score,cost_usd
```

### 7.3 `summary.json`

각 condition의 aggregate 결과를 저장한다.

```json
{
  "condition_id": "sentence_buffer_openai_fixed700",
  "n_turns": 100,
  "server_ttfa_ms": {
    "mean": 812.4,
    "p50": 755.1,
    "p90": 1102.9,
    "p95": 1240.0
  },
  "quality_score": {
    "mean": 4.2
  },
  "task_success_rate": 0.91
}
```

## 8. 논문 스토리라인

### 8.1 가능한 제목

```text
Measuring and Improving the Latency-Quality Tradeoff in Realtime Voice Agents
```

또는:

```text
When Should a Voice Agent Start Speaking? A Systematic Study of Latency-Quality Tradeoffs in Streaming Voice Agents
```

### 8.2 논문 기여점

기여점은 다음 중 2-3개로 정리하는 것이 좋다.

- 실시간 음성 에이전트의 latency-quality tradeoff를 측정하는 재현 가능한 benchmark harness
- endpointing, buffering, model backend, TTS strategy의 체계적 비교
- adaptive buffering 또는 semantic endpointing 기반 개선 방법
- 한국어/영어 실시간 voice agent에서의 언어별 latency-quality 분석
- tool-calling voice agent에서 latency spike와 task success tradeoff 분석

### 8.3 논문 구성

```text
1. Introduction
   - 실시간 음성 에이전트에서 빠른 응답이 중요한 이유
   - 빠르기만 하면 품질이 떨어지는 문제
   - 연구 질문과 기여점

2. Background
   - streaming STT
   - streaming LLM
   - streaming TTS
   - endpointing and turn-taking

3. System
   - baseline architecture
   - event instrumentation
   - buffering and endpointing modules

4. Metrics
   - latency metrics
   - quality metrics
   - cost metrics

5. Experiments
   - dataset/scenarios
   - experimental conditions
   - implementation details

6. Results
   - latency breakdown
   - buffering comparison
   - endpointing comparison
   - quality tradeoff
   - cost tradeoff

7. Proposed Improvement
   - adaptive buffering or selected method
   - ablation

8. Discussion
   - practical deployment lessons
   - language/domain limitations
   - provider variance

9. Limitations
   - API dependency
   - human evaluation size
   - simulated vs real user conversations

10. Conclusion
```

## 9. 논문용 표와 그림

### 9.1 필수 표

- Table 1: Experimental conditions
- Table 2: Latency breakdown by condition
- Table 3: Quality metrics by condition
- Table 4: Cost per turn by condition
- Table 5: Ablation of proposed improvement

### 9.2 필수 그림

- Figure 1: Realtime voice agent architecture
- Figure 2: Event timeline of one conversation turn
- Figure 3: Latency-quality Pareto frontier
- Figure 4: TTFA distribution by buffering strategy
- Figure 5: Endpointing delay vs false endpoint rate
- Figure 6: Tool-calling latency breakdown

### 9.3 가장 중요한 그림

논문에서 가장 중요한 그림은 Pareto frontier다.

```text
x-axis: time-to-first-audio
y-axis: quality score or task success rate
point: each system condition
```

이 그림이 연구 주장을 가장 직관적으로 보여준다.

## 10. 실행 순서 체크리스트

### Milestone 1. 실행 가능한 baseline

- [ ] `credentials.env` 준비
- [ ] dependency 설치
- [ ] server 실행
- [ ] web client 접속
- [ ] one-turn voice conversation 성공
- [ ] 실패 케이스 로그 확인

### Milestone 2. Logging

- [ ] run ID 생성
- [ ] session ID 생성
- [ ] turn ID 생성
- [ ] event logger 구현
- [ ] JSONL 저장
- [ ] latency 계산 script 구현

### Milestone 3. Benchmark

- [ ] 실험 config format 정의
- [ ] prompt/audio dataset 준비
- [ ] benchmark runner 구현
- [ ] n회 반복 실행
- [ ] results directory 자동 생성

### Milestone 4. Metrics

- [ ] latency metric 계산
- [ ] quality metric 계산
- [ ] cost metric 계산
- [ ] summary 생성
- [ ] plot 생성

### Milestone 5. Experiments

- [ ] buffering experiment
- [ ] endpointing experiment
- [ ] model backend experiment
- [ ] tool-calling experiment
- [ ] Korean/bilingual experiment

### Milestone 6. Improvement

- [ ] 개선 방법 구현
- [ ] baseline과 비교
- [ ] ablation 실행
- [ ] failure case 분석

### Milestone 7. Paper

- [ ] abstract 초안
- [ ] introduction 초안
- [ ] system section 작성
- [ ] experiment section 작성
- [ ] result table 생성
- [ ] figure 생성
- [ ] limitation 작성
- [ ] related work 정리
- [ ] 최종 revision

## 11. 첫 2주 작업안

### Week 1

목표: baseline과 logging까지 만든다.

작업:

- Day 1: 현재 레포 실행 가능성 점검
- Day 2: Web client를 server에서 서빙하도록 수정
- Day 3: end-to-end voice conversation 성공
- Day 4: event logger 추가
- Day 5: JSONL log와 latency 계산 script 추가
- Day 6: 10개 prompt로 수동/반자동 실험
- Day 7: 첫 latency breakdown 표 생성

### Week 2

목표: 첫 번째 논문용 실험 결과를 만든다.

작업:

- Day 8: buffering strategy 추상화
- Day 9: full response, sentence, phrase buffering 구현
- Day 10: benchmark runner 구현
- Day 11: buffering experiment 실행
- Day 12: quality evaluation 초안 구현
- Day 13: Pareto frontier plot 생성
- Day 14: preliminary result memo 작성

2주 후에는 다음 질문에 답할 수 있어야 한다.

```text
Sentence-level streaming은 full-response 방식보다 얼마나 빠른가?
Phrase-level streaming은 더 빠르지만 품질을 얼마나 잃는가?
우리가 제안한 adaptive strategy는 Pareto frontier를 개선하는가?
```

## 12. 리스크와 대응

### API latency variance

문제:

- cloud API는 시간대와 네트워크 상태에 따라 latency가 흔들린다.

대응:

- 반복 횟수를 늘린다.
- p50/p90/p95를 보고한다.
- 같은 시간대에 paired experiment를 실행한다.
- 가능하면 local vLLM 조건을 추가한다.

### Human evaluation 비용

문제:

- 품질 평가는 사람이 하는 것이 가장 좋지만 비용이 든다.

대응:

- 초기에는 automatic metric과 LLM-as-judge를 쓴다.
- 최종 논문용으로 작은 human preference study를 추가한다.

### Provider dependency

문제:

- Deepgram, OpenAI, ElevenLabs 특정 조합에 종속될 수 있다.

대응:

- provider-specific result와 provider-independent system insight를 분리해 서술한다.
- 가능한 경우 대체 provider 또는 local model을 일부 조건에 포함한다.

### Demo와 benchmark의 차이

문제:

- live browser demo는 재현성이 낮고 benchmark는 실제 사용자 경험을 완전히 반영하지 못한다.

대응:

- offline benchmark와 live user-perceived measurement를 모두 둔다.
- server-side metric과 client-side metric을 구분한다.

## 13. 바로 다음 작업

우리가 다음으로 할 일은 다음 순서가 가장 좋다.

1. `chapters/05_websocket_server/server.py`가 web client를 서빙하도록 수정한다.
2. `credentials.env` 존재 여부와 필수 key validation을 추가한다.
3. `http://localhost:8888`에서 end-to-end voice demo를 실행한다.
4. event logging module을 추가한다.
5. 첫 baseline log를 만들고 `server_ttfa_ms`, `llm_ttft_ms`, `tts_ttfb_ms`를 계산한다.

첫 코드 작업의 목표는 작게 잡는다.

```text
One real conversation turn, one clean JSONL event log, one latency summary.
```

여기까지 되면 이 레포는 더 이상 단순 튜토리얼이 아니라 연구 실험 플랫폼의 씨앗이 된다.

