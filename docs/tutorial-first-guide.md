# Tutorial-First Guide

이 문서는 연구를 시작하기 전에 이 저장소의 튜토리얼을 먼저 통과하기 위한 실행 가이드다.

우리의 연구 목표는 "실시간 음성 에이전트의 latency-quality tradeoff를 체계적으로 측정하고 개선하는 것"이다. 하지만 그 전에 기본 부품이 실제로 동작하는지 확인해야 한다. 따라서 첫 단계는 연구용 리팩터링이 아니라 튜토리얼을 순서대로 실행하며 baseline voice agent의 동작 원리를 확인하는 것이다.

## 1. 튜토리얼을 먼저 하는 이유

연구용 실험 플랫폼은 다음 흐름이 실제로 작동해야 만들 수 있다.

```text
Speech audio
  -> STT transcript
  -> LLM response
  -> TTS audio
  -> WebSocket/browser playback
```

튜토리얼의 목적은 각 부품을 따로 검증한 다음, 마지막에 하나의 음성 대화 pipeline으로 연결하는 것이다.

튜토리얼을 끝내면 우리는 다음을 알게 된다.

- Deepgram STT가 음성을 정확히 텍스트로 바꾸는가
- LLM backend가 streaming token을 안정적으로 주는가
- ElevenLabs TTS가 빠르게 첫 audio chunk를 주는가
- sentence buffering이 언제 TTS를 시작하는가
- WebSocket server가 browser audio를 받을 수 있는가
- 현재 repo에서 문서와 실제 코드가 어긋나는 부분은 어디인가
- 연구용 logging과 benchmark를 어디에 붙여야 하는가

## 2. 전체 진행 순서

권장 순서는 아래와 같다.

```text
0. 환경 준비
1. Chapter 1: STT
2. Chapter 2: LLM
3. Chapter 3: TTS
4. Chapter 4: Streaming pipeline
5. Chapter 5: WebSocket server
6. Chapter 7: Browser web client
7. Chapter 6: VAD and interruptions
8. Chapter 8: Enterprise tool-calling agent
9. Chapter 9: Benchmarking
10. 연구용 logging/benchmark 개조 시작
```

Chapter 6, 8, 9는 현재 문서에 언급된 일부 파일이 실제 repo에 없으므로, 먼저 "있는 파일로 개념 확인"을 하고 나중에 우리가 직접 보완한다.

## 3. 환경 준비

### 3.1 Python 환경

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Python 가상환경을 켠 뒤 모든 챕터를 실행한다.

### 3.2 API key 준비

```bash
cp credentials.env.example credentials.env
```

`credentials.env`에 최소한 아래 값을 채운다.

```bash
DEEPGRAM_API_KEY=...
ELEVENLABS_API_KEY=...
OPENAI_API_KEY=...
```

각 key의 역할은 다음과 같다.

- `DEEPGRAM_API_KEY`: speech-to-text
- `OPENAI_API_KEY`: LLM response generation
- `ELEVENLABS_API_KEY`: text-to-speech

local vLLM을 쓸 경우 `OPENAI_API_KEY` 대신 `vllm serve`를 실행할 수 있지만, 튜토리얼을 빠르게 통과하는 첫 단계에서는 OpenAI API fallback을 쓰는 것이 가장 단순하다.

### 3.3 매번 실행 전 공통 명령

```bash
source .venv/bin/activate
source credentials.env
```

## 4. Chapter 1: Streaming STT

위치:

```text
chapters/01_streaming_stt/
```

핵심 질문:

```text
내 음성이 Deepgram을 통해 텍스트로 잘 바뀌는가?
partial transcript와 final transcript는 어떻게 다른가?
```

파일:

- `deepgram_basic.py`: WAV 파일을 REST API로 전사한다.
- `deepgram_streaming.py`: WAV 파일을 20ms chunk로 나눠 WebSocket streaming STT를 시뮬레이션한다.

실행:

```bash
source credentials.env
python chapters/01_streaming_stt/deepgram_basic.py
python chapters/01_streaming_stt/deepgram_streaming.py
```

주의:

- 실제 음성 파일이 없으면 synthetic audio를 사용하므로 transcript가 비어 있을 수 있다.
- 제대로 확인하려면 `chapters/01_streaming_stt/output/speech_sample.wav`에 실제 말소리가 들어간 WAV 파일을 넣는 것이 좋다.
- macOS에서는 `arecord`가 없을 수 있으므로 QuickTime, Voice Memos, ffmpeg, Audacity 등으로 16kHz mono WAV를 만들어 넣으면 된다.

확인할 것:

- Deepgram 연결 성공
- partial transcript 출력 여부
- final transcript 출력 여부
- time to first transcript
- `speech_final=True`가 언제 발생하는지

연구 관점에서 중요한 점:

- STT latency는 전체 TTFA의 첫 병목이다.
- partial transcript를 LLM에 너무 빨리 보내면 품질 문제가 생길 수 있다.
- final transcript만 쓰면 안정적이지만 지연시간이 늘 수 있다.

완료 기준:

```text
실제 음성 WAV 하나를 넣었을 때 final transcript가 출력된다.
```

## 5. Chapter 2: Streaming LLM

위치:

```text
chapters/02_streaming_llm/
```

핵심 질문:

```text
LLM이 첫 token을 얼마나 빨리 주는가?
OpenAI-compatible API를 vLLM과 OpenAI API 양쪽에서 쓸 수 있는가?
function calling은 어떤 흐름으로 일어나는가?
```

파일:

- `vllm_setup.py`: LLM backend 연결과 기본 completion 확인
- `vllm_streaming.py`: streaming token 확인
- `vllm_function_calling.py`: function calling 흐름 확인

실행 옵션 A: OpenAI API fallback

```bash
source credentials.env
python chapters/02_streaming_llm/vllm_setup.py
python chapters/02_streaming_llm/vllm_streaming.py
python chapters/02_streaming_llm/vllm_function_calling.py
```

실행 옵션 B: local vLLM

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct --port 8000 --gpu-memory-utilization 0.8
```

다른 터미널:

```bash
source credentials.env
python chapters/02_streaming_llm/vllm_setup.py
python chapters/02_streaming_llm/vllm_streaming.py
python chapters/02_streaming_llm/vllm_function_calling.py
```

확인할 것:

- backend가 OpenAI API인지 vLLM인지
- first token latency
- token streaming이 실제로 순차 출력되는지
- function call result를 다시 LLM에 넣어 최종 답변을 만드는지

연구 관점에서 중요한 점:

- LLM TTFT는 사용자가 느끼는 침묵 시간에 직접 영향을 준다.
- 모델이 클수록 품질은 좋아질 수 있지만 TTFT와 cost가 증가할 수 있다.
- tool calling은 task success를 높이지만 추가 latency를 만든다.

완료 기준:

```text
streaming response가 출력되고 TTFT를 관찰할 수 있다.
```

## 6. Chapter 3: Streaming TTS

위치:

```text
chapters/03_streaming_tts/
```

핵심 질문:

```text
텍스트를 얼마나 빨리 첫 audio chunk로 바꿀 수 있는가?
TTS streaming은 전체 음성이 끝나기 전에 재생을 시작할 수 있는가?
```

파일:

- `elevenlabs_basic.py`: 텍스트를 음성 파일로 저장한다.
- `elevenlabs_streaming.py`: streaming TTS chunk와 first audio timing을 확인한다.

실행:

```bash
source credentials.env
python chapters/03_streaming_tts/elevenlabs_basic.py
python chapters/03_streaming_tts/elevenlabs_streaming.py
```

확인할 것:

- ElevenLabs API 연결 성공
- WAV 또는 PCM audio 생성
- first audio byte timing
- audio chunk 크기와 개수

연구 관점에서 중요한 점:

- TTS TTFB는 agent가 "말하기 시작하는 시간"에 직접 영향을 준다.
- chunk를 너무 작게 보내면 빠르지만 자연스러움이 떨어질 수 있다.
- 긴 문장을 한 번에 보내면 자연스럽지만 첫 응답이 늦어진다.

완료 기준:

```text
텍스트 하나가 음성으로 변환되고, streaming TTS의 첫 chunk timing을 볼 수 있다.
```

## 7. Chapter 4: Streaming Pipeline

위치:

```text
chapters/04_streaming_pipeline/
```

핵심 질문:

```text
LLM token stream을 언제 TTS로 넘겨야 가장 빠르면서 자연스러운가?
```

파일:

- `sentence_buffer.py`: LLM token을 모아 문장 경계를 감지한다.
- `pipeline.py`: LLM streaming output을 sentence buffer로 묶고 TTS로 넘긴다.

문서상 언급되지만 현재 없는 파일:

- `pipeline_demo.py`

실행:

```bash
source credentials.env
python chapters/04_streaming_pipeline/sentence_buffer.py
python chapters/04_streaming_pipeline/pipeline.py
```

확인할 것:

- abbreviation, decimal number, punctuation 처리
- 문장이 완성되는 순간
- 첫 TTS 요청 시점
- pipeline output audio 저장 여부
- LLM TTFT, TTS time, TTFA 추정치

연구 관점에서 중요한 점:

- 이 챕터가 latency-quality tradeoff 연구의 중심이다.
- full response buffering, sentence buffering, phrase buffering, adaptive buffering을 여기서 비교할 수 있다.
- 현재 `SentenceBuffer`는 영어 punctuation 중심이므로 한국어 실험에서는 확장이 필요하다.

완료 기준:

```text
LLM 응답이 문장 단위로 잘리고 각 문장이 TTS로 전달된다.
```

## 8. Chapter 5: WebSocket Server

위치:

```text
chapters/05_websocket_server/
```

핵심 질문:

```text
browser 또는 test client에서 보낸 audio를 server가 받아 STT -> LLM -> TTS로 처리할 수 있는가?
```

파일:

- `server.py`: FastAPI WebSocket server
- `test_client.py`: WAV 파일을 WebSocket으로 보내는 test client

실행:

터미널 1:

```bash
source credentials.env
python chapters/05_websocket_server/server.py --port 8888
```

터미널 2:

```bash
python chapters/05_websocket_server/test_client.py --input chapters/01_streaming_stt/output/speech_sample.wav
```

주의:

- `speech_sample.wav`가 없으면 Chapter 1에서 실제 음성 파일을 먼저 준비해야 한다.
- 현재 server는 STT를 "말이 끝난 뒤 batch 방식"으로 호출한다. 완전한 streaming STT server는 아니다.
- server가 Chapter 7 web client를 정적 파일로 서빙하도록 되어 있지 않을 수 있다. 이 부분은 튜토리얼 후 우리가 baseline 개선 작업에서 고친다.

확인할 것:

- WebSocket 연결 성공
- server가 audio bytes를 받는지
- transcript가 생성되는지
- LLM response가 생성되는지
- TTS audio bytes가 client로 돌아오는지

연구 관점에서 중요한 점:

- 여기서부터 end-to-end latency를 측정할 수 있다.
- server event logging을 붙일 주요 위치가 보인다.
- speech endpointing과 conversation state machine의 한계가 드러난다.

완료 기준:

```text
WAV test client를 통해 한 turn의 transcript와 agent response가 생성된다.
```

## 9. Chapter 7: Browser Web Client

위치:

```text
chapters/07_web_client/
```

핵심 질문:

```text
브라우저 마이크 입력과 speaker playback으로 실제 대화 UX를 만들 수 있는가?
```

파일:

- `index.html`: UI
- `app.js`: WebSocket 연결, mic capture, playback queue
- `audio-capture-worklet.js`: microphone PCM capture
- `audio-playback-worklet.js`: playback worklet

문서상 실행:

```text
http://localhost:8888
```

주의:

- README에는 server가 web client를 서빙한다고 되어 있지만, 현재 Chapter 5 server에는 정적 파일 mount가 보이지 않는다.
- 따라서 실제 browser demo를 하려면 우리가 server에 static serving을 추가하거나 별도 local static server를 띄워야 한다.
- 마이크 권한은 localhost에서는 허용된다.

확인할 것:

- Connect 버튼
- microphone permission
- WebSocket 연결
- transcript UI 업데이트
- agent audio playback

연구 관점에서 중요한 점:

- browser에서 사용자가 느끼는 latency는 server-side latency와 다를 수 있다.
- `client_first_audio_played` 같은 client-side event가 필요하다.
- 실제 UX 연구는 이 챕터를 통과해야 가능하다.

완료 기준:

```text
브라우저에서 말하고 agent 음성 응답을 들을 수 있다.
```

## 10. Chapter 6: VAD and Interruptions

위치:

```text
chapters/06_vad_and_interruptions/
```

핵심 질문:

```text
사용자가 언제 말하기 시작하고 끝냈는지 더 정확히 감지할 수 있는가?
agent가 말하는 도중 사용자가 끼어들면 멈출 수 있는가?
```

파일:

- `vad_basics.py`: Silero VAD standalone demo

문서상 언급되지만 현재 없는 파일:

- `server_with_vad.py`

실행:

```bash
python chapters/06_vad_and_interruptions/vad_basics.py
```

확인할 것:

- Silero VAD 모델 로딩
- speech probability 또는 speech segment 감지
- CPU에서 충분히 빠른지

연구 관점에서 중요한 점:

- endpointing strategy 비교 실험의 핵심이다.
- fixed silence threshold보다 VAD가 더 빠르거나 안정적인지 측정할 수 있다.
- interruption recovery time을 정의하려면 이 챕터 개념이 필요하다.

완료 기준:

```text
VAD standalone demo가 실행되고 speech detection의 입력/출력 구조를 이해한다.
```

## 11. Chapter 8: Enterprise Agent with Function Calling

위치:

```text
chapters/08_enterprise_agent/
```

핵심 질문:

```text
voice agent가 단순 답변을 넘어 tool call로 업무를 수행할 수 있는가?
```

파일:

- `tools.py`: appointment booking mock tools와 schema
- `agent.py`: LLM agent loop와 function calling 처리

문서상 언급되지만 현재 없는 파일:

- `agent_demo.py`

실행:

```bash
source credentials.env
python chapters/08_enterprise_agent/agent.py
```

확인할 것:

- LLM이 tool call을 선택하는지
- tool handler가 실행되는지
- tool result를 바탕으로 최종 답변을 만드는지
- multi-turn appointment scenario가 유지되는지

연구 관점에서 중요한 점:

- tool call은 품질과 task success를 높이지만 latency spike를 만든다.
- "Let me check that" 같은 latency masking 전략을 실험할 수 있다.
- task-oriented voice agent 논문으로 확장할 수 있다.

완료 기준:

```text
agent.py의 demo conversation에서 tool call과 최종 답변이 출력된다.
```

## 12. Chapter 9: Production and Benchmarking

위치:

```text
chapters/09_production/
```

핵심 질문:

```text
STT, LLM, TTS 각각의 latency를 어떻게 측정하고 비교할 것인가?
```

파일:

- `benchmark.py`: STT, LLM TTFT, TTS TTFB, estimated end-to-end TTFA 측정

문서상 언급되지만 현재 없는 파일:

- `multi_user_server.py`

실행:

```bash
source credentials.env
python chapters/09_production/benchmark.py
```

확인할 것:

- Deepgram STT latency
- LLM TTFT
- ElevenLabs TTS TTFB
- estimated end-to-end TTFA
- p50, mean, min, max

연구 관점에서 중요한 점:

- 여기서 시작해 연구용 benchmark runner로 확장한다.
- 단순 평균보다 p50, p90, p95가 중요하다.
- provider latency variance를 반복 실험으로 다뤄야 한다.

완료 기준:

```text
각 component latency가 출력되고, 어떤 component가 병목인지 볼 수 있다.
```

## 13. 튜토리얼 완료 체크리스트

튜토리얼을 끝냈다고 판단하려면 아래를 확인한다.

- [ ] `credentials.env`가 준비되어 있다.
- [ ] Chapter 1에서 실제 음성 WAV의 transcript를 얻었다.
- [ ] Chapter 2에서 LLM streaming response와 TTFT를 확인했다.
- [ ] Chapter 3에서 TTS audio와 TTFB를 확인했다.
- [ ] Chapter 4에서 sentence buffering pipeline을 실행했다.
- [ ] Chapter 5에서 WebSocket test client로 한 turn을 처리했다.
- [ ] Chapter 7 browser client 실행을 위해 필요한 server static serving gap을 확인했다.
- [ ] Chapter 6에서 VAD standalone demo를 이해했다.
- [ ] Chapter 8에서 function calling agent loop를 확인했다.
- [ ] Chapter 9에서 latency benchmark를 실행했다.
- [ ] 문서와 실제 코드가 안 맞는 파일 목록을 확인했다.

## 14. 튜토리얼 중 기록해야 할 것

각 챕터를 실행할 때 아래를 짧게 기록한다.

```text
date:
chapter:
command:
success/failure:
observed latency:
error message:
what we learned:
next fix:
```

연구로 넘어갈 때 이 기록이 baseline issue list가 된다.

## 15. 튜토리얼 후 바로 할 일

튜토리얼 통과 후 첫 연구용 작업은 아래 순서로 한다.

1. Chapter 5 server가 Chapter 7 web client를 직접 서빙하게 만든다.
2. browser에서 end-to-end voice conversation을 성공시킨다.
3. event logger를 추가한다.
4. `events.jsonl`을 저장한다.
5. `server_ttfa_ms`, `llm_ttft_ms`, `tts_ttfb_ms`를 계산한다.
6. Chapter 4의 buffering 전략을 실험 가능하도록 추상화한다.

첫 연구 milestone은 다음 한 줄로 정의한다.

```text
One real conversation turn, one clean event log, one latency summary.
```

