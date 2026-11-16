# capture — 회의 녹음을 녹취록으로

회의록 시연 앞에 붙는 단계다. 회의 소리를 녹음하고 받아 적어 `inbox/` 에 녹취록으로 넣는다. 그 뒤는 기존 흐름(수집 → 작성 → 검수 → 전달)이 그대로 맡는다.

강의의 시연 샘플은 **Google Meet**이다. **Teams·Webex 등으로 확장**할 때도 녹음 → 받아 적기 → 회의록 생성이라는 흐름은 같다. 회의 도구별 참가·녹음 조건을 확인하고, 완료된 음성을 공통 입력 형식으로 넘기면 뒤의 받아 적기와 회의록 생성을 재사용할 수 있다. 현재 강의용 앞단의 실제 회의 시험은 Meet에서 했으며, Teams·Webex 연동은 아직 시험하지 않았다.

```text
Meet 탭 ─(확장)→ receive.py ─→ jobs/<id>/audio.wav ─(transcribe.py, Ollama)→ transcript.json ─(publish.py)→ inbox/<id>-v1.txt
```

| 파일 | 하는 일 |
|---|---|
| `extension/` | Chrome 확장. 회의 탭에서 아이콘을 누르면 그 탭 소리를 16 kHz 로 1초마다 수신기로 보낸다. 다시 누르면 멈춘다 |
| `receive.py` | 127.0.0.1:8765 수신기. job 폴더를 열고 WAV 를 이어 쓰고, 멈추면 곧바로 받아 적는다. `--meeting` 을 주면 inbox 에 넣기까지 — 그 뒤는 `scripts/watch.sh` 가 받는다 |
| `pcm_writer.py` | 받은 PCM 을 WAV 로 이어 쓴다. 소리가 실제로 들어온 뒤에야 녹음 중으로 표시 |
| `transcribe.py` | 쉼에서 15초 이하로 잘라 로컬 Ollama(`gemma4:e2b`)로 받아 적는다 |
| `publish.py` | 받아 적기가 끝난 것만 `[HH:MM:SS] 화자: 발화` 형식으로 inbox 에 넣는다 |

준비물은 Chrome, Ollama(`ollama pull gemma4:e2b`), uv 다. 표준 라이브러리만 쓰고 가상 오디오 장치·ffmpeg 는 필요 없다.

## Windows + WSL 에서 준비하기 (2026-11-16 실측)

Chrome 은 Windows 에, 수신기·받아 적기·Ollama 는 WSL(Ubuntu) 에 둔다. **Ollama 와 `transcribe.py` 는 같은 쪽에 둔다** — WSL 에서 Windows 의 `localhost:11434` 는 기본으로 닿지 않는다(다른 쪽이면 `OLLAMA_HOST=http://<그 주소>:11434`).

1. Ubuntu 터미널 창(Claude Code 의 `!` 가 아니라 따로 연 창 — sudo 가 비밀번호를 물어야 한다)에서:
   ```bash
   sudo apt-get install -y zstd                      # Ubuntu 26.04 WSL 에는 없어 Ollama 설치가 멈춘다
   curl -fsSL https://ollama.com/install.sh | sh     # 서비스로 켜진다(systemctl is-active ollama → active)
   ollama pull gemma4:e2b                            # 4.6 GB
   ```
2. WSL 에서 수신기를 켠다.
   ```bash
   cd ~/harness-practice/demo/minutes/capture
   uv run python receive.py --meeting "회의 이름" --attendees "이름, 이름"
   ```
3. Windows Chrome 에서 `chrome://extensions` → 개발자 모드 → 「압축해제된 확장 프로그램 로드」 → `\\wsl.localhost\Ubuntu\home\<아이디>\harness-practice\demo\minutes\capture\extension`. 회의 탭에서 아이콘을 누르면 REC, 다시 누르면 멈추고 받아 적어 `inbox/` 에 넣는다.

| 확인한 것 | 결과 |
|---|---|
| Ollama 공식 설치 | `zstd` 없이는 `ERROR: This version requires zstd for extraction` 로 멈춤 → 설치 뒤 0.40.16 · 서비스 active |
| 모델 | `gemma4:e2b` = Q4_K_M 4.6B(맥은 nvfp4 5.2B — 같은 태그라도 양자화가 다르다), 강사 PC 의 GPU(RTX 3060)에서 실행 |
| Windows → WSL `127.0.0.1:8765` | 닿는다(WSL2 NAT · 기본 localhost 전달). 확장과 같은 요청(`/start` → 1초 `/chunk` → `/stop`)을 Windows PowerShell 에서 보내 33.6초가 `audio.wav` 에 바이트까지 같게 들어감 |
| 받아 적기 → inbox | 33.6초 · 세 조각 · 109.9초(첫 모델 올리기 포함). 「글 반응」→「결반응」, 「글 조회수」→「결 조회수」 오독 |
| Ollama 가 꺼져 있을 때 | 조각마다 `Connection refused` → 전사 `failed`, inbox 에 넣지 않음 |
| **실제 Windows Chrome 확장** | `\\wsl.localhost\Ubuntu\…\capture\extension` 로 로드됨. 시험 음성 탭에서 아이콘 → REC → 멈춤 세 번 모두 `via: extension` · 전사 `done` · inbox 넣기. 39.9초 녹음은 세 조각 · 처리 1.4초(모델이 이미 올라간 뒤). 오독은 「글 반응」→「회의 반경」, 「글 조회수」→「회의 조회수」 — 같은 문장도 녹음마다 다르게 틀린다 |
| 8765 를 Windows 프로그램이 쓰고 있을 때 | 확인하지 않음 — 그때는 녹음이 WSL 수신기가 아니라 그 Windows 프로그램으로 갈 수 있다 |

## 이렇게 정한 까닭 (2026-11-16 실측)

- **녹음 계정은 사람이 클릭해 들어간다.** Playwright 로 띄운 Chrome 은 로그인한 계정이어도 Meet 가 「이 화상 통화에 참여할 수 없습니다」 로 바로 막았다. 같은 프로필을 보통 Chrome 으로 열면 참가 화면이 나왔다. 이 차단은 피하지 않는다.
- **녹음은 브라우저 안에서 한다.** OS 오디오 장치(BlackHole 등)는 macOS 전용이라 Windows·WSL 에서 따로 맞춰야 한다.
- **15초 조각.** 78초를 한 번에 넣으면 문장을 조용히 빠뜨렸고, 25초 조각에서는 한 문단을 세 번 되풀이했다.
- **영어 지시문.** 한국어 지시문을 쓰니 「처음부터 끝까지 빠짐없이 받아 적은 문장만 출력한다」 가 받아 적은 결과에 섞여 나왔다. 이 문구가 다시 보이면 `prompt_echo` 로 표시만 하고 지우지는 않는다.
- **확장의 녹음 상태는 offscreen 문서가 기억한다.** MV3 service worker 는 쉬는 동안 내려가면서 변수를 잃는다. 그래서 첫 판에서 멈춤 버튼이 듣지 않았다.

## 보장하지 않는 것

- 받아 적은 글이 원음과 같은지. 사람 이름이 바뀌었고(「정 대리」→「정태리」), 같은 음성을 다시 받아 적으면 결과가 달라졌다.
- 화자. 이 전사에는 화자 구분이 없어 모두 `미확인` 으로 쓴다. 검수 M3 는 `미확인` 을 참석자로 받지 않고, 참석자를 명단(`--attendees`)으로만 확인한 뒤 사람이 볼 메모를 남긴다.
- Windows·WSL 에서 실행 — 수신기·받아 적기·inbox 넣기는 WSL(Ubuntu 26.04)에서, 실제 Windows Chrome 확장을 WSL 경로에서 올려 녹음 → inbox 까지 확인했다(위 「Windows + WSL 에서 준비하기」, 합성 음성 한 사람). 여러 사람 실제 목소리, Windows 단독(WSL 없이)은 돌려 보지 않았다.
