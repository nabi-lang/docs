# 4. 신호와 스펙트럼

기록한 신호는 일정한 속도로 잰 표본의 벡터입니다. `signal`은 그 안의 주파수를 찾고(FFT와 스펙트럼 추정), 원하지 않는 주파수를 없애고(필터), 피크 같은 특징을 찾습니다. 표본은 배열 비슷한 무엇이든 들어오고, `Vector<f64>`로, 스펙트럼이면 `Vector<Complex<f64>>`로 나갑니다.

## 시험용 신호

50 Hz와 120 Hz 두 음을 1 kHz로 1초 동안 표본화합니다. `signal`은 속도와 시간에 쓰는 수 접미사를 정의합니다. `1khz`는 1000.0, `50hz`는 50.0, `5ms`는 0.005로, 뜻을 드러내는 평범한 `f64` 값입니다. `Array::from_fn`은 인덱스로부터 표본을 만듭니다.

```nabi
use core::array::{Array, Vector};
use signal::units::*;

let fs = 1khz;
let x: Vector<f64> = Array::from_fn([1000], |i: [usize; 1]| -> f64 {
    let t = (i[0] as f64) / fs;
    (2.0 * f64::PI * 50hz * t).sin() + 0.5 * (2.0 * f64::PI * 120hz * t).sin()
});
println(f"{x.len()} samples, {x.len() as f64 / fs} s, peak value {x.max() ?? 0.0:.3}");
```

```output
1000 samples, 1 s, peak value 1.476
```

## FFT

`signal::rfft`는 실수 신호의 이산 푸리에 변환입니다. 0부터 표본화 속도의 절반까지 주파수마다 복소수 하나를 줍니다. `signal::fft::rfftfreq`는 각 구간의 주파수를 줍니다. 크기에 `2 / n`을 곱하면 각 음의 진폭이 됩니다.

```nabi
use core::array::{Array, Vector};
use signal::units::*;

let fs = 1khz;
let n = 1000;
let x: Vector<f64> = Array::from_fn([n], |i: [usize; 1]| -> f64 {
    let t = (i[0] as f64) / fs;
    (2.0 * f64::PI * 50hz * t).sin() + 0.5 * (2.0 * f64::PI * 120hz * t).sin()
});

let spec = try signal::rfft(x);
let freq = try signal::fft::rfftfreq(n, d: 1.0 / fs);
let amp = signal::abs(spec) .* (2.0 / (n as f64));
for k in 0..amp.len() {
    if amp[k] > 0.1 { println(f"{freq[k]} Hz: amplitude {amp[k]:.3}"); }
}
```

```output
50 Hz: amplitude 1.000
120 Hz: amplitude 0.500
```

`fft`와 `ifft`는 2의 거듭제곱뿐 아니라 어떤 길이의 복소 신호든 변환하며, `fft2`와 `fftn`은 이미지와 3차원 데이터에 씁니다.

## 잡음과 파워 스펙트럼

실제 측정에는 잡음이 얹혀 있고, 그 FFT 한 번도 잡음투성이입니다. Welch 방법은 겹치는 구간들의 스펙트럼을 평균합니다. 주파수 해상도는 낮아지지만 잡음은 훨씬 줄어듭니다. `welch`는 주파수 `f`, 파워 밀도 `p`, 그리고 가장 큰 값의 주파수인 `peak`를 갖춘 `Spectrum`을 돌려줍니다.

```nabi
use core::array::{Array, Vector};
use signal::units::*;
use stats::{Distribution, Normal, Sampler};

let fs = 1khz;
let g = Sampler::seed(3);
let noise = (try Normal::new(0.0, 1.0)).samples(ref g, 4000);
let x: Vector<f64> = Array::from_fn([4000], |i: [usize; 1]| -> f64 {
    let t = (i[0] as f64) / fs;
    0.8 * (2.0 * f64::PI * 120hz * t).sin() + noise[i[0]]
});

let s = try signal::welch(x, fs: fs, nperseg: 256);
println(f"{s.f.len()} bins, {s.f[1] - s.f[0]:.4} Hz apart");
println(f"strongest frequency {s.peak():.1} Hz");
```

```output
129 bins, 3.9062 Hz apart
strongest frequency 121.1 Hz
```

음은 표본 하나하나에서는 잡음보다 약하지만(표준편차 1에 대해 0.8), 스펙트럼에서는 여전히 뚜렷하게 보입니다.

## 신호 거르기

필터는 어떤 주파수는 남기고 다른 주파수는 없앱니다. `butter`는 Butterworth 필터(남기는 대역에서 평평함)를 2차 구간들로 설계하는데, 차수가 높아도 정확하게 유지됩니다. `sosfiltfilt`는 필터를 앞으로 한 번, 뒤로 한 번 돌리므로 출력에 지연이 없습니다. 여기서는 80 Hz 저역 통과 필터가 120 Hz 음을 없애고 50 Hz 음을 남깁니다.

```nabi
use core::array::{Array, Vector};
use signal::filter::{Band, butter, sosfiltfilt};
use signal::units::*;

let fs = 1khz;
let tone = |f: f64, t: f64| -> f64 { (2.0 * f64::PI * f * t).sin() };
let x: Vector<f64> = Array::from_fn([1000], |i: [usize; 1]| -> f64 {
    let t = (i[0] as f64) / fs;
    tone(50hz, t) + 0.5 * tone(120hz, t)
});
let want: Vector<f64> = Array::from_fn([1000], |i: [usize; 1]| -> f64 { tone(50hz, (i[0] as f64) / fs) });

let sos = try butter(8, Band::Lowpass(80hz), fs: fs);
let y = try sosfiltfilt(sos, x);

// Away from the ends, the filtered signal is the 50 Hz tone alone.
let err = (y[100:900] .- want[100:900]).map(|v: f64| -> f64 { v.abs() }).max() ?? 0.0;
println(f"largest difference from the pure tone: {err:.4}");
```

```output
largest difference from the pure tone: 0.0009
```

`Band::Highpass`, `Bandpass`, `Bandstop`은 다른 종류를, `cheby1`, `cheby2`, `bessel`은 다른 모양을, `firwin`은 유한 임펄스 응답 필터를 만듭니다. 설계에 문제가 있으면 에러가 무엇을 넘겨야 하는지 알려 줍니다. `fs:`를 빠뜨리면 그때는 대역 경계가 표본화 속도 단위라고 설명해 줍니다.

## 피크 찾기

`find_peaks`는 신호의 극댓값을 찾고, `height`, `distance`, `prominence`로 중요한 것만 남깁니다. 여기서는 합성한 심장 박동 기록의 박동을 찾습니다.

```nabi
use core::array::{Array, Vector};

// A spike every 0.8 s on a slow wander, sampled at 100 Hz.
let x: Vector<f64> = Array::from_fn([500], |i: [usize; 1]| -> f64 {
    let t = (i[0] as f64) / 100.0;
    let phase = t % 0.8;
    0.2 * (2.0 * f64::PI * 0.1 * t).sin() + (-(phase - 0.4) * (phase - 0.4) / 0.0005).exp()
});

let p = try signal::find_peaks(x, height: 0.5, distance: 30);
println(f"{p.indices.len()} beats at samples {p.indices:?}");
let gaps = p.indices.to_list().pairwise().map(|(a, b)| b - a).to_list();
println(f"gaps {gaps:?}: {60.0 / 0.8} beats per minute");
```

```output
6 beats at samples [40, 120, 200, 280, 360, 440]
gaps [80, 80, 80, 80, 80]: 75 beats per minute
```

## 합성곱

평활화, 경계 검출, 그리고 많은 필터가 짧은 커널과의 합성곱입니다. `⊛`(`signal::ops`에서 옴)는 완전 합성곱, `⋆`는 상관이며, `signal::convolve`는 출력 길이를 입력과 같게 하는 `mode:`를 받습니다.

```nabi
use core::array::Vector;
use signal::ops::*;

let x: Vector<f64> = [0, 0, 1, 1, 1, 0, 0];
let k: Vector<f64> = [0.25, 0.5, 0.25];
println(x ⊛ k);
println(try signal::convolve(x, k, mode: signal::conv::Mode::Same));
```

```output
[0.0, 0.0, 0.25, 0.75, 1.0, 0.75, 0.25, 0.0, 0.0]
[0.0, 0.25, 0.75, 1.0, 0.75, 0.25, 0.0]
```

## 다음으로

- [피팅과 최적화](05-fitting.html): 측정한 데이터에 모델 맞추기.
- [`signal` API](https://packages.nabi-lang.org/packages/signal): 창 함수, STFT와 스펙트로그램, 힐베르트 변환, 리샘플링.
