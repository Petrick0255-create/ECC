---
name: auto-foley
description: MMAudio처럼 영상 장면에 맞는 효과음을 자동으로 입힐 때 사용. 컷·움직임·등장 요소를 분석해 큐 시트를 짜고, 녹음이나 코드 합성음으로 싱크를 맞춰 넣는다(GPU·모델 불필요). (MMAudio alternative, foley, video sound effects sync)
metadata:
  origin: personal
---

# auto-foley — 장면에 맞춰 효과음 입히기 (MMAudio 대체)

MMAudio(github.com/hkchengrex/MMAudio)는 영상을 보고 소리를 생성하는 AI 모델이지만 NVIDIA GPU와 수 GB의 모델 파일이 필요해 GPU가 없는 환경(claude.ai 채팅, 일반 노트북·맥)에서는 돌릴 수 없다. 이 스킬은 같은 목표(장면과 어울리고 싱크가 맞는 효과음)를 분석과 배치로 해낸다. 사용자에게는 "AI가 소리를 생성한 것"이 아니라 "장면을 분석해 효과음을 골라 맞춰 넣은 것"이라고 정확히 말한다.

## When to Use
- 무음이거나 효과음이 빈약한 영상에 장면에 맞는 효과음을 한 번에 입히고 싶을 때
- MMAudio 같은 영상→소리 생성 모델을 쓸 GPU가 없을 때
- 효과음 배치 근거(큐 시트)를 남겨 수정하기 쉽게 하고 싶을 때

## How It Works

### 준비
- earshot-lite의 `sfxkit.py`가 작업 폴더에 있어야 한다. 없으면 earshot-lite 스킬을 불러 코드를 `sfxkit.py`로 저장한다.
- 실제 녹음이 필요하면 openverse-sfx 스킬을 쓴다(네트워크가 허용될 때만).

### 순서
1. **분석**: `python3 sfxkit.py sheet VIDEO -o sheet.png`(Read로 본다), `cuts`, `gaps`. 컷 사이 장면은 프레임을 몇 장 뽑아 직접 본다: `ffmpeg -v error -ss <초> -i VIDEO -frames:v 1 -vf scale=960:-2 f_<초>.jpg`
2. **이벤트 분류**: 각 시각에 화면에서 무슨 일이 일어나는지 보고 소리를 정한다.

| 화면 | 소리 (sfxkit 레시피) |
|---|---|
| 하드 컷·와이프·슬라이드 전환 | whoosh (최고점 = 컷) |
| 타이틀·로고가 쾅 등장 | impact, 앞에 얕은 whoosh |
| 카드·텍스트·아이콘이 톡 나타남 | pop, 작은 건 click |
| 타이핑·키보드·마우스 클릭 화면 | keypress / click |
| 클라이맥스 직전 고조 | riser (끝 = 정점) |
| 완료·정답·체크 표시 | ding |
| 장면 시작·부드러운 페이드 | swell |
| 오류·노이즈 연출 | glitch |
| 묵직한 착지·박스 내려앉음 | thud |
| 실제 사물(종이, 문, 박수, 자연음) | openverse-sfx로 녹음 찾기 |

3. **큐 시트**: `시각 | 화면 | 소리 | 게인 | 팬`을 표로 먼저 정리한다.
   - 같은 소리가 반복되면 파라미터를 조금씩 바꾸고(`whoosh@d=0.55,f1=4200`), 팬은 좌우로 번갈아(±0.3) 준다.
   - 1초 안에 3개 이상 몰리면 덜 중요한 것을 뺀다. 효과음은 장면을 돕는 정도로만.
   - 내레이션이 나오는 구간은 효과음 게인을 3~6 dB 더 낮춘다.
4. **믹스**: `python3 sfxkit.py mix VIDEO -o out.mp4 <큐들>`. 원본에 소리가 있으면 효과음이 -4 dB 더 낮게 섞이고 전체는 -14 LUFS로 맞춰진다. 컷 위주의 빠른 초안이면 `kit --style soft`(또는 punchy)로 시작해 큐 시트로 다듬는다.
5. **검수**: `gaps`(남은 무음 컷이 의도된 것인지), `sheet`(효과음이 컷 선에 붙어 있는지), `describe`(LUFS·트루피크·클리핑).
6. **전달**: 결과 영상과 함께 큐 시트를 보여 주고, 직접 들을 수 없으니 어색한 곳을 알려 달라고 말한다.

## Examples
큐 시트 예시와 그대로 옮긴 명령:

| 시각 | 화면 | 소리 | 게인 | 팬 |
|---|---|---|---|---|
| 0.6 | 타이틀 카드 등장 | pop | -22 | 0 |
| 4.53 | 슬라이드 전환 | whoosh@d=0.55 | -16 | -0.3 |
| 12.93 | 결과 화면 쾅 | impact | -14 | 0 |
| 20.1 | 체크 표시 | ding | -18 | 0.3 |

```bash
python3 sfxkit.py mix demo.mp4 -o demo_foley.mp4 0.6:pop:-22 4.53:whoosh@d=0.55:-16:-0.3 12.93:impact:-14 20.1:ding:-18:0.3
python3 sfxkit.py gaps demo_foley.mp4
```

## 진짜 MMAudio가 필요할 때 (안내만)
NVIDIA GPU가 있는 리눅스 PC에서, CUDA용 PyTorch를 먼저 설치한 뒤:
```bash
git clone https://github.com/hkchengrex/MMAudio.git && cd MMAudio && pip install -e .
python demo.py --duration=8 --video=input.mp4 --prompt "footsteps on gravel"
```
결과는 `./output`에 저장된다. 개발자는 우분투에서만 테스트했다고 밝히고 있다. GPU가 없는 환경에서는 실행할 수 없다.