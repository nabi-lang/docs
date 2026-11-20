# 4. Signals and Spectra

A recorded signal is a vector of samples taken at a fixed rate. `signal` finds the frequencies in it (the FFT and spectral estimates), removes the ones you do not want (filters), and finds features such as peaks. Samples come in as anything array-like and go out as `Vector<f64>`, or `Vector<Complex<f64>>` for a spectrum.

## A test signal

Two tones, 50 Hz and 120 Hz, sampled at 1 kHz for one second. `signal` defines number suffixes for rates and times: `1khz` is 1000.0, `50hz` is 50.0 and `5ms` is 0.005, plain `f64` values that say what they mean. `Array::from_fn` builds the samples from their index.

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

## The FFT

`signal::rfft` is the discrete Fourier transform of a real signal: one complex number per frequency from 0 up to half the sampling rate. `signal::fft::rfftfreq` gives the frequency of each bin. The magnitudes, scaled by `2 / n`, are the amplitudes of the tones.

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

`fft` and `ifft` transform complex signals of any length, not only powers of two; `fft2` and `fftn` work on images and volumes.

## Noise and the power spectrum

A real measurement has noise on top, and a single FFT of it is noisy too. Welch's method averages the spectra of overlapping segments: less detail in frequency, much less noise. `welch` returns a `Spectrum` with the frequencies `f`, the power density `p` and `peak`, the frequency of the largest value.

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

The tone is weaker than the noise in every single sample (0.8 against a standard deviation of 1), but the spectrum still shows it clearly.

## Filter a signal

A filter keeps some frequencies and removes others. `butter` designs a Butterworth filter (flat in the band it keeps) as second-order sections, which stay accurate at high orders; `sosfiltfilt` runs it forwards and backwards, so the output has no delay. Here a low-pass at 80 Hz removes the 120 Hz tone and keeps the 50 Hz one.

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

`Band::Highpass`, `Bandpass` and `Bandstop` make the other kinds; `cheby1`, `cheby2` and `bessel` other shapes; `firwin` a finite impulse response filter. A design error says what to pass: forget `fs:` and the message explains that the band edges are then in units of the sampling rate.

## Find peaks

`find_peaks` finds the local maxima of a signal; `height`, `distance` and `prominence` keep only the ones that matter. Here, the beats of a synthetic heartbeat trace.

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

## Convolution

Smoothing, edge detection and many filters are convolutions with a short kernel. `⊛` (from `signal::ops`) is the full convolution, `⋆` the correlation; `signal::convolve` takes a `mode:` to keep the output the length of the input.

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

## What next

- [Fitting and optimization](05-fitting.html): fit a model to measured data.
- The [`signal` API](https://packages.nabi-lang.org/packages/signal): windows, STFT and spectrograms, the Hilbert transform, resampling.
