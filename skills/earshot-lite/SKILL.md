---
name: earshot-lite
description: 영상·오디오의 소리를 측정하고 컷에 맞는 효과음을 코드로 만들어 입힐 때 사용. 모델·네트워크 없이 LUFS·컷·무음 컷 분석, whoosh·impact 등 10종 합성, 믹스까지. (SFX, sound effects, audio analysis, LUFS, earshot alternative)
metadata:
  origin: personal
---

# earshot-lite — 에이전트의 귀 (claude.ai 채팅·PC 공용)

Leonxlnx/earshot의 아이디어(소리를 듣고, 평가하고, 코드로 만든다)를 네트워크·GPU 없이도 돌아가게 직접 구현한 도구다. 원본 earshot의 청취 모델(소리를 말로 찾는 CLAP 등)은 없고, 수치 측정과 스펙트로그램 시트로 판단한다. 원본과 다르다는 점은 사용자에게 숨기지 않는다.

## When to Use
- 코드로 만든 영상(모션그래픽, 시연 영상, Remotion·HyperFrames 결과물)에 효과음을 입힐 때
- 영상의 음량(LUFS)·트루피크·소리 없는 컷을 수치로 점검할 때
- 녹음 효과음 후보 중 원하는 성격(whoosh 등)에 가까운 것을 고를 때

## How It Works

### 준비
- 작업 폴더에 `sfxkit.py`가 없으면 맨 아래 '코드'를 그대로 `sfxkit.py`로 저장한다(Write 도구).
- 필요한 것: python3, numpy, scipy, matplotlib, ffmpeg. claude.ai 채팅 환경에는 기본으로 있다. 네트워크가 막힌 환경에서는 설치를 시도하지 않고, PC에서 빠진 것이 있으면 사용자에게 알린 뒤 설치한다.

### 명령 (`python3 sfxkit.py <명령>`)
| 명령 | 하는 일 |
|---|---|
| `describe FILE` | 길이, LUFS·트루피크·LRA, 밝기(Hz), 음정감(tonality), 어택·감쇠, 클리핑, 한 줄 요약 |
| `events FILE` | 소리 시작점(onset) 목록 [초, 강도] |
| `cuts VIDEO` | 화면 컷 시각 |
| `gaps VIDEO` | 소리가 없는 컷(컷 ±0.15초 안에 onset 없음) |
| `sheet VIDEO -o sheet.png` | 프레임 8장 + 스펙트로그램 + 파형 + 컷(하늘색)·onset(주황) 표시. Read로 직접 본다 |
| `render RECIPE -o out.wav [--param d=0.8]` | whoosh, impact, click, pop, keypress, riser, ding, swell, glitch, thud |
| `kit VIDEO -o out.mp4 [--style soft/punchy] [--no-motion]` | 컷마다 whoosh(punchy는 impact+whoosh), 큰 화면 변화엔 pop/작은 whoosh 자동 배치, -14 LUFS |
| `mix VIDEO -o out.mp4 시각:소리[:게인dB[:팬]] ...` | 원하는 시각에 직접 배치. 소리는 레시피(`whoosh@d=0.5,f1=5000`) 또는 파일 경로 |
| `rank REF FILES...` | REF(레시피 이름이나 파일)와 얼마나 비슷한지 순위 + 길이·클리핑 |

싱크 규칙: whoosh는 최고점, riser는 끝, swell은 가운데, impact·click·pop·ding·thud와 녹음 파일은 시작점이 지정 시각에 맞춰진다. 녹음 파일은 앞 무음을 잘라 두고 써야 정확하다.

레시피 주요 파라미터: whoosh(d, f0, f1, peak), impact(d, low), click(d, f), pop(d, f0, f1), riser(d, f0, f1), ding(f, d), swell(d, f), glitch(d), thud(d, f).

### 작업 순서
1. 영상을 받으면 먼저 `sheet`와 `gaps`로 지금 소리 상태를 본다. 원본 오디오가 있으면 `describe`로 LUFS를 확인한다.
2. 컷과 등장 요소를 보고 큐 시트(시각, 소리, 게인, 팬)를 표로 짠다. 단순하면 `kit`, 의도가 있으면 `mix`.
3. 결과를 다시 `sheet`·`gaps`·`describe`로 확인한다: 의도하지 않은 무음 컷이 없는지, -14±1 LUFS, 트루피크 -1 dBFS 이하, 클리핑 없음.
4. 직접 들을 수 없으므로 전달할 때 "수치로 확인했으니 들어보고 어색한 곳을 말해 달라"고 솔직히 말한다.

게인 기준(효과음 트랙): whoosh -16, 작은 whoosh -19, pop·click -22, impact -14, ding -18 dB. 원본에 소리가 있으면 효과음 전체가 -4 dB 더 낮게 섞이고, 전체는 -14 LUFS로 맞춰진다. 같은 소리를 반복할 때는 파라미터를 조금씩 바꾸고 팬을 좌우로 번갈아(±0.3) 반복감을 줄인다.

## Examples
```bash
python3 sfxkit.py gaps demo.mp4                     # 소리 없는 컷 찾기
python3 sfxkit.py kit demo.mp4 -o demo_sfx.mp4      # 컷마다 whoosh 자동 배치
python3 sfxkit.py mix demo.mp4 -o out.mp4 4.53:whoosh@d=0.5:-16:-0.3 12.9:impact:-14 20.1:ding:-18
python3 sfxkit.py describe out.mp4                  # LUFS·트루피크 확인
```

## 원본 earshot이 필요하면
github.com/Leonxlnx/earshot (MIT). Python 3.10+, ffmpeg, 약 1.1GB 모델(CPU)이 필요하고 Claude Code에는 MCP로 연결한다. 네트워크가 열린 PC의 Claude Code라면 원본 설치를 권하고, 네트워크가 막힌 채팅 환경에서는 안내만 한다.

## 코드 (sfxkit.py)
```python
#!/usr/bin/env python3
"""sfxkit — earshot-lite. Listen to, measure and make sound effects with numpy + scipy + ffmpeg.
No models, no network. Subcommands: describe, events, cuts, gaps, sheet, render, kit, mix, rank."""
import sys, json, re, subprocess, argparse
import numpy as np
from scipy import signal
from scipy.io import wavfile

SR = 48000
RNG = np.random.default_rng(7)

# ---------- io ----------
def load(path, sr=SR):
    raw = subprocess.run(['ffmpeg', '-v', 'error', '-i', path, '-vn', '-ac', '1', '-ar', str(sr), '-f', 'f32le', '-'],
                         capture_output=True).stdout
    return np.frombuffer(raw, np.float32).copy()

def save(path, x, sr=SR):
    x = np.asarray(x, np.float32)
    wavfile.write(path, sr, (np.clip(x, -1, 1) * 32767).astype(np.int16))

def has_audio(path):
    r = subprocess.run(['ffprobe', '-v', 'error', '-select_streams', 'a', '-show_entries', 'stream=index', '-of', 'csv=p=0', path],
                       capture_output=True, text=True).stdout.strip()
    return bool(r)

def duration(path):
    return float(subprocess.run(['ffprobe', '-v', 'error', '-show_entries', 'format=duration', '-of', 'csv=p=0', path],
                                capture_output=True, text=True).stdout.strip() or 0)

def loudness(path):
    e = subprocess.run(['ffmpeg', '-hide_banner', '-nostats', '-i', path, '-filter_complex', 'ebur128=peak=true', '-f', 'null', '-'],
                       capture_output=True, text=True).stderr
    s = e[e.rfind('Summary:'):]
    g = lambda k: (re.search(k + r':\s+(-?[\d.]+|-inf)', s) or [None, 'nan'])[1]
    return {'lufs': float(g('I').replace('-inf', '-99')), 'lra': float(g('LRA')), 'true_peak_dbfs': float(g('Peak').replace('-inf', '-99'))}

# ---------- analysis ----------
def stft_mag(x, n=1024, hop=256):
    f, t, Z = signal.stft(x, SR, nperseg=n, noverlap=n - hop, boundary=None)
    return f, t, np.abs(Z)

def onsets(x, k=4.0, min_gap=0.1):
    if len(x) < 2048: return []
    f, t, M = stft_mag(x)
    L = np.log1p(100 * M)
    flux = np.maximum(np.diff(L, axis=1), 0).sum(0)
    flux = np.convolve(flux, np.hanning(5) / np.hanning(5).sum(), 'same')
    med = np.median(flux); mad = np.median(np.abs(flux - med)) + 1e-9
    z = (flux - med) / (1.4826 * mad)
    pk, _ = signal.find_peaks(z, height=k, distance=max(1, int(min_gap / (t[1] - t[0]))))
    return [(round(float(t[i + 1]), 3), round(float(z[i]), 1)) for i in pk]

def describe(path):
    x = load(path); d = len(x) / SR
    out = {'file': path, 'duration_s': round(d, 3)}
    out.update(loudness(path))
    if d == 0: return out
    rms = 20 * np.log10(np.sqrt(np.mean(x ** 2)) + 1e-12)
    f, t, M = stft_mag(x); P = M ** 2 + 1e-12; spec = P.mean(1)
    centroid = float((f * spec).sum() / spec.sum())
    from scipy.ndimage import median_filter  # 음정감: 주변보다 뾰족하게 솟은 주파수 성분이 차지하는 에너지 비율
    flat = float(spec[spec > 8 * median_filter(spec, 31)].sum() / spec.sum())
    env = np.sqrt(signal.lfilter([1], [1, -0.995], x ** 2) * 0.005 + 1e-12)
    pk = int(np.argmax(env)); thr = env[pk] * 0.1
    attack_ms = 1000 * (pk - np.argmax(env > thr)) / SR
    after = np.where(env[pk:] < thr)[0]; decay_ms = 1000 * (after[0] if len(after) else len(env) - pk) / SR
    silent = float(np.mean(env < 10 ** (-50 / 20)))
    ev = onsets(x)
    kind = 'build-up (riser/swell)' if attack_ms > 400 else 'short transient' if d < 0.6 or (decay_ms < 300 and len(ev) <= 2) else ('sustained' if len(ev) < max(3, d) else 'busy/rhythmic')
    tone = 'tonal' if flat > 0.6 else ('noisy' if flat < 0.25 else 'mixed')
    out.update({'rms_dbfs': round(rms, 1), 'centroid_hz': round(centroid), 'tonality': round(flat, 3), 'attack_ms': round(attack_ms),
                'decay_ms': round(decay_ms), 'silence_share': round(silent, 2), 'onsets': len(ev), 'clipping': bool(np.mean(np.abs(x) > 0.999) > 1e-4),
                'summary': f'{d:.2f} s {kind}, {tone}, brightness {centroid:.0f} Hz, attack {attack_ms:.0f} ms, decays in {decay_ms:.0f} ms'})
    return out

def cuts(video, thr=0.3):
    e = subprocess.run(['ffmpeg', '-hide_banner', '-i', video, '-vf', f"scale=320:-2,select='gt(scene,{thr})',showinfo", '-an', '-f', 'null', '-'],
                       capture_output=True, text=True).stderr
    return [round(float(m), 3) for m in re.findall(r'pts_time:([\d.]+)', e)]

def motion(video, fps=15):
    raw = subprocess.run(['ffmpeg', '-v', 'error', '-i', video, '-vf', f'fps={fps},scale=160:90,format=gray', '-f', 'rawvideo', '-'],
                         capture_output=True).stdout
    F = np.frombuffer(raw, np.uint8).reshape(-1, 90, 160).astype(np.float32)
    m = np.r_[0, np.abs(np.diff(F, axis=0)).mean((1, 2))]
    return np.arange(len(m)) / fps, m

def gaps(video, win=0.15):
    c = cuts(video)
    if not has_audio(video): return {'cuts': c, 'silent_cuts': c, 'note': 'video has no audio track'}
    ev = onsets(load(video))
    silent = [t for t in c if not any(abs(t - o) <= win for o, _ in ev)]
    return {'cuts': c, 'audio_onsets': len(ev), 'silent_cuts': silent}

def sheet(video, out, n=8):
    import matplotlib; matplotlib.use('Agg'); import matplotlib.pyplot as plt
    d = duration(video); x = load(video) if has_audio(video) else np.zeros(int(d * SR), np.float32)
    c = cuts(video); ev = onsets(x)
    fig = plt.figure(figsize=(16, 9), dpi=110); gs = fig.add_gridspec(3, n, height_ratios=[0.55, 1.4, 0.8])
    for i in range(n):
        t = d * (i + 0.5) / n
        img = subprocess.run(['ffmpeg', '-v', 'error', '-ss', str(t), '-i', video, '-frames:v', '1', '-vf', 'scale=320:-2', '-f', 'rawvideo', '-pix_fmt', 'rgb24', '-'],
                             capture_output=True).stdout
        ax = fig.add_subplot(gs[0, i]); ax.axis('off')
        if img:
            h = len(img) // (320 * 3); ax.imshow(np.frombuffer(img, np.uint8)[:h * 960].reshape(h, 320, 3))
        ax.set_title(f'{t:.1f}s', fontsize=8)
    ax = fig.add_subplot(gs[1, :])
    if np.any(x):
        ax.specgram(x + 1e-6 * RNG.standard_normal(len(x)), NFFT=1024, Fs=SR, noverlap=768, cmap='magma'); ax.set_yscale('symlog', linthresh=500); ax.set_ylim(40, 20000)
    ax.set_xlim(0, d); ax.set_ylabel('Hz')
    for t in c: ax.axvline(t, color='cyan', lw=1, alpha=.8)
    ax2 = fig.add_subplot(gs[2, :]); tt = np.arange(len(x)) / SR
    ax2.plot(tt[::50], x[::50], lw=.4, color='k'); ax2.set_xlim(0, d)
    for t in c: ax2.axvline(t, color='c', lw=1)
    for t, s in ev: ax2.axvline(t, color='orange', lw=.6, alpha=.6)
    ax2.set_xlabel('s  (cyan = picture cuts, orange = sound onsets)')
    fig.tight_layout(); fig.savefig(out); return {'sheet': out, 'cuts': len(c), 'onsets': len(ev)}

# ---------- synthesis recipes: each returns (mono signal, sync offset in s) ----------
def _t(d): return np.arange(int(d * SR)) / SR
def _noise(d): return RNG.standard_normal(int(d * SR))
def _bp(x, lo, hi, order=2): b, a = signal.butter(order, [lo / (SR / 2), min(hi, SR / 2 - 100) / (SR / 2)], 'band'); return signal.lfilter(b, a, x)
def _lp(x, fc, order=2): b, a = signal.butter(order, fc / (SR / 2)); return signal.lfilter(b, a, x)
def _hp(x, fc, order=2): b, a = signal.butter(order, fc / (SR / 2), 'high'); return signal.lfilter(b, a, x)
def _norm(x, peak=0.9): return x / (np.max(np.abs(x)) + 1e-9) * peak

def whoosh(d=0.7, f0=250, f1=3500, peak=0.6, **_):
    t = _t(d); n = _noise(d); u = t / d; out = np.zeros_like(n); seg = 512
    fc = f0 * (f1 / f0) ** np.minimum(u / peak, 1) * (1 - 0.4 * np.maximum(u - peak, 0) / (1 - peak))
    for i in range(0, len(n), seg):
        c = fc[i]; out[i:i + seg] = _bp(n[max(0, i - 2048):i + seg], c * 0.6, c * 1.6)[-len(n[i:i + seg]):]
    env = np.where(u < peak, (u / peak) ** 2.2, np.exp(-(u - peak) / (1 - peak) * 4))
    return _norm(out * env, .8), peak * d

def impact(d=1.0, low=48, **_):
    t = _t(d); f = low + 90 * np.exp(-t * 18); body = np.sin(2 * np.pi * np.cumsum(f) / SR) * np.exp(-t * 4.5)
    crack = _hp(_noise(d), 1500) * np.exp(-t * 60); tail = _lp(_noise(d), 900) * np.exp(-t * 6) * .3
    return _norm(body + .5 * crack + tail, .95), 0.0

def click(d=0.05, f=2600, **_):
    t = _t(d); x = _bp(_noise(d), f * .6, f * 1.8) * np.exp(-t * 180) + .4 * np.sin(2 * np.pi * f * 1.3 * t) * np.exp(-t * 250)
    return _norm(x, .7), 0.0

def pop(d=0.14, f0=500, f1=1300, **_):
    t = _t(d); f = f0 + (f1 - f0) * (1 - np.exp(-t * 60)); x = np.sin(2 * np.pi * np.cumsum(f) / SR) * np.exp(-t * 32) * (1 - np.exp(-t * 900))
    return _norm(x, .8), 0.0

def keypress(**_):
    a, _ = click(.04, 3200); b, _ = click(.035, 2200); body = _bp(_noise(.08), 300, 900) * np.exp(-_t(.08) * 70) * .5
    x = np.zeros(int(.16 * SR)); x[:len(a)] += a; x[:len(body)] += body; o = int(.07 * SR); x[o:o + len(b)] += .6 * b
    return _norm(x, .75), 0.0

def riser(d=2.0, f0=200, f1=2400, **_):
    t = _t(d); u = t / d; f = f0 * (f1 / f0) ** (u ** 1.6)
    saw = signal.sawtooth(2 * np.pi * np.cumsum(f) / SR) * .25 + signal.sawtooth(2 * np.pi * np.cumsum(f * 1.005) / SR) * .25
    x = _lp(saw, 4000) + _hp(_noise(d), 2000) * .4 * u
    return _norm(x * u ** 2.5, .8), d

def ding(f=1320, d=1.4, **_):
    t = _t(d); x = sum(a * np.sin(2 * np.pi * f * r * t) * np.exp(-t * k) for r, a, k in [(1, 1, 3), (2.76, .45, 6), (5.4, .25, 10), (8.9, .1, 16)])
    return _norm(x * (1 - np.exp(-t * 2000)), .7), 0.0

def swell(d=1.6, f=220, **_):
    t = _t(d); x = sum(np.sin(2 * np.pi * f * m * (1 + dt) * t) for m in (1, 1.5, 2) for dt in (-.003, .003))
    env = np.sin(np.pi * np.minimum(t / d, 1)) ** 2
    return _norm(x * env, .5), d * 0.5

def glitch(d=0.3, **_):
    n = _bp(_noise(d), 800, 6000); g = (np.sin(2 * np.pi * 28 * _t(d)) > 0).astype(float)
    x = np.round(n * g * 6) / 6
    return _norm(x * np.exp(-_t(d) * 6), .6), 0.0

def thud(d=0.35, f=68, **_):
    t = _t(d); x = np.sin(2 * np.pi * (f + 40 * np.exp(-t * 40)) * t) * np.exp(-t * 14)
    return _norm(x + .2 * _lp(_noise(d), 300) * np.exp(-t * 30), .9), 0.0

RECIPES = dict(whoosh=whoosh, impact=impact, click=click, pop=pop, keypress=keypress, riser=riser, ding=ding, swell=swell, glitch=glitch, thud=thud)

# ---------- placing and mixing ----------
def place(track, x, t_event, sync, gain_db=0.0, pan=0.0):
    s = int(round((t_event - sync) * SR)); g = 10 ** (gain_db / 20)
    a, b = max(0, s), min(track.shape[0], s + len(x))
    if b <= a: return
    seg = x[a - s:b - s] * g; l, r = np.sqrt(.5 * (1 - pan)), np.sqrt(.5 * (1 + pan))
    track[a:b, 0] += seg * l; track[a:b, 1] += seg * r

def mux(video, track, out, sfx_gain_db=0.0, target_lufs=-14.0):
    wav = out.rsplit('.', 1)[0] + '_sfx.wav'
    from scipy.io import wavfile as wf
    wf.write(wav, SR, (np.clip(track, -1, 1) * 32767).astype(np.int16))
    if has_audio(video):
        fc = f'[1:a]volume={sfx_gain_db}dB[s];[0:a][s]amix=inputs=2:normalize=0,loudnorm=I={target_lufs}:TP=-1.5[a]'
    else:
        fc = f'[1:a]loudnorm=I={target_lufs}:TP=-1.5[a]'
    subprocess.run(['ffmpeg', '-v', 'error', '-y', '-i', video, '-i', wav, '-filter_complex', fc, '-map', '0:v', '-map', '[a]',
                    '-c:v', 'copy', '-c:a', 'aac', '-b:a', '192k', '-ar', str(SR), '-shortest', '-movflags', '+faststart', out], check=True)
    return {'video': out, 'sfx_track': wav}

def kit(video, out, style='soft', motion_sfx=True):
    d = duration(video); track = np.zeros((int(d * SR) + SR, 2), np.float32); cues = []
    c = cuts(video)
    for i, t in enumerate(c):
        if style == 'punchy':
            x, s = impact(.8); place(track, x, t, s, -14); cues.append((t, 'impact'))
            w, ws = whoosh(.45, 300, 4200, .8); place(track, w, t, ws, -18, (-1) ** i * .4)
        else:
            w, ws = whoosh(.55 + .1 * RNG.random(), 220 + 80 * RNG.random(), 3000 + 1500 * RNG.random(), .7); place(track, w, t, ws, -16, (-1) ** i * .35)
            cues.append((t, 'whoosh'))
    if motion_sfx:
        # 화면 전체 평균 밝기 변화(0~255)가 큰 순간만: 4 이상 = 요소 등장(pop), 10 이상 = 큰 움직임(작은 whoosh)
        tm, m = motion(video); pk, _ = signal.find_peaks(m, height=4.0, distance=15)
        for i in pk:
            t = float(tm[i])
            if all(abs(t - u) > .5 for u in c):
                big = m[i] >= 10
                x, s = whoosh(.4, 400, 5000, .7) if big else pop()
                place(track, x, t, s, -19 if big else -22); cues.append((round(t, 2), 'small whoosh' if big else 'pop'))
    res = mux(video, track, out, sfx_gain_db=-4 if has_audio(video) else 0)
    res['cues'] = sorted(cues); return res

def mix(video, out, cues):
    d = duration(video); track = np.zeros((int(d * SR) + SR, 2), np.float32)
    for cue in cues:
        t, src, *rest = cue.split(':'); g = float(rest[0]) if rest else -12; p = float(rest[1]) if len(rest) > 1 else 0
        name, _, prm = src.partition('@')  # 예: whoosh@d=0.5,f1=5000
        if name in RECIPES: x, s = RECIPES[name](**{k: float(v) for k, v in (q.split('=') for q in prm.split(',') if q)})
        else: x, s = load(src), 0.0
        place(track, x, float(t), s, g, p)
    return mux(video, track, out, sfx_gain_db=-4 if has_audio(video) else 0)

def logmel(x, n_mels=40):
    f, t, M = stft_mag(x, 2048, 512); P = M ** 2
    mel = lambda hz: 2595 * np.log10(1 + hz / 700); edges = 700 * (10 ** (np.linspace(mel(40), mel(16000), n_mels + 2) / 2595) - 1)
    fb = np.array([np.interp(f, [edges[i], edges[i + 1], edges[i + 2]], [0, 1, 0], left=0, right=0) for i in range(n_mels)])
    S = np.log(fb @ P + 1e-10); z = lambda v: (v - v.mean()) / (v.std() + 1e-9)
    env = S.max(0); env = np.interp(np.linspace(0, len(env) - 1, 24), np.arange(len(env)), env)
    return np.r_[z(S.mean(1)), 0.7 * z(S.std(1)), 0.7 * z(env)]

def rank(ref, files):
    r = logmel(RECIPES[ref]()[0] if ref in RECIPES else load(ref)); out = []
    for fp in files:
        x = load(fp)
        if len(x) < 2048: continue
        v = logmel(x); sim = float(np.dot(r, v) / (np.linalg.norm(r) * np.linalg.norm(v) + 1e-9))
        out.append({'file': fp, 'similarity': round(sim, 3), 'duration_s': round(len(x) / SR, 2), 'clipping': bool(np.mean(np.abs(x) > .999) > 1e-4)})
    return sorted(out, key=lambda o: -o['similarity'])

if __name__ == '__main__':
    ap = argparse.ArgumentParser(); sp = ap.add_subparsers(dest='cmd', required=True)
    for c in ('describe', 'events', 'cuts', 'gaps'): sp.add_parser(c).add_argument('file')
    p = sp.add_parser('sheet'); p.add_argument('video'); p.add_argument('-o', default='sheet.png')
    p = sp.add_parser('render'); p.add_argument('recipe', choices=RECIPES); p.add_argument('-o', required=True)
    p.add_argument('--param', action='append', default=[], help='key=value, e.g. d=0.8')
    p = sp.add_parser('kit'); p.add_argument('video'); p.add_argument('-o', required=True); p.add_argument('--style', default='soft', choices=['soft', 'punchy']); p.add_argument('--no-motion', action='store_true')
    p = sp.add_parser('mix'); p.add_argument('video'); p.add_argument('-o', required=True); p.add_argument('cues', nargs='+', help='time:recipe[@k=v,..]_or_file[:gain_db[:pan]]')
    p = sp.add_parser('rank'); p.add_argument('ref'); p.add_argument('files', nargs='+')
    a = ap.parse_args()
    if a.cmd == 'describe': r = describe(a.file)
    elif a.cmd == 'events': r = onsets(load(a.file))
    elif a.cmd == 'cuts': r = cuts(a.file)
    elif a.cmd == 'gaps': r = gaps(a.file)
    elif a.cmd == 'sheet': r = sheet(a.video, a.o)
    elif a.cmd == 'render':
        kw = {k: float(v) for k, v in (s.split('=') for s in a.param)}; x, s = RECIPES[a.recipe](**kw); save(a.o, x); r = {'file': a.o, 'sync_s': round(s, 3), 'duration_s': round(len(x) / SR, 3)}
    elif a.cmd == 'kit': r = kit(a.video, a.o, a.style, not a.no_motion)
    elif a.cmd == 'mix': r = mix(a.video, a.o, a.cues)
    elif a.cmd == 'rank': r = rank(a.ref, a.files)
    print(json.dumps(r, ensure_ascii=False, indent=1, default=float))
```