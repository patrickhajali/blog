---
title: Analyzing local field potential recordings
published: true
permalink: neuro
---
{% include interactive.html %}

## The data 

I was recently given about ~10GB of [local field potential](https://en.wikipedia.org/wiki/Local_field_potential)(LFP) recordings from electrodes placed in the brains (region?) of awake mice. The recordings are sampled at 25kHz (collecting a sample every 40 microseconds) across 64 channels. 

| channel # | depth | channel # | depth |
| :--: | :--: | :--: | :--: |
| 12 | 1 | 15 | 9 |
| 28 | 2 | 31 | 10 |
| 19 | 3 | 0 | 11 |
| 3 | 4 | 16 | 12 |
| 14 | 5 | 1 | 13 |
| 30 | 6 | 17 | 14 |
| 2 | 7 | 13 | 15 |
| 18 | 8 | 29 | 16 |

#### Downsampling 

Given the sampling rate of 25kHz, which significantly exceeds the Nyquist frequency necessary for capturing the highest frequency oscillations observed in the brain ([fast ripples](https://onlinelibrary.wiley.com/doi/10.1111/j.1528-1157.1999.tb02065.x); 250-500Hz), I downsampled the data by a factor of 50 (new Nyquist frequency of 250Hz) to decrease the size of the data without compressing important information. Most brainwaves of interest (fast ripples are not common, occurring only during epilepsy) are well below 250Hz (more specific), so I am drastically oversampling (as desired). 

#### Anti-aliasing 

Before downsampling, I applied an anti-aliasing filter to prevent frequencies above the new Nyquist frequency (post-downsampling) from folding back into the lower-frequency range and creating aliasing errors. The ideal anti-aliasing filter should have a flat magnitude response to frequencies below the Nyquist frequency and a sharp drop off above it. This, of course, is not realizable in practice. 

Taking the moving average of the signal is a form of low-pass filtering. The frequency response of the moving average filter is shown below. The response is somewhat flat in the passable band but is 'bouncy' for frequencies higher than the cutoff. This is not ideal.  

![](assets/images/ma_freqz.png)

Because we are oversampling by a large margin, we can afford to sacrifice steepness near the cutoff frequency in exchange for a flatter response in the passband. This makes the [Butterworth filter](https://en.wikipedia.org/wiki/Butterworth_filter), which has maximally flat frequency response in the passband,  a good choice. 

![](assets/images/Butterworth.png)

I used ```scipy.signal.butter``` to generate a 6th-order Butterworth filter with a cutoff frequency at 250Hz. We can decompose the transfer function of the filter into a cascade of $N$-many second-order filters, taking the form 

$$ H(z) = \prod_{i=1}^{N} \frac{b_{0,i} + b_{1,i}z^{-1} + b_{2,i}z^{-2}}{1 - a_{1,i}z^{-1} - a_{2,i}z^{-2}}$$

Each second-order filter is referred to as a second-order section (SOS), and this method used to improve the numerical stability of the filter. High-order filters, combined with very large or small coefficients, can introduce significant rounding errors due to the finite-precision of the floating point numbers. 

The overall filter is the cascaded combination of these second-order sections. For $N=3$, applying the filter in the time-domain yields the following equations. 

$$y[n] = y_3[n]$$

$$y_3[n] = b_{0,3} \cdot y_2[n] + b_{1,3} \cdot y_2[n-1] + b_{2,3} \cdot y_2[n-2] - a_{1,3} \cdot y_3[n-1] - a_{2,3} \cdot y_3[n-2]$$
$$y_2[n] = b_{0,2} \cdot y_1[n] + b_{1,2} \cdot y_1[n-1] + b_{2,2} \cdot y_1[n-2] - a_{1,2} \cdot y_2[n-1] - a_{2,2} \cdot y_2[n-2]$$
$$y_1[n] = b_{0,1} \cdot x[n] + b_{1,1} \cdot x[n-1] + b_{2,1} \cdot x[n-2] - a_{1,1} \cdot y_1[n-1] - a_{2,1} \cdot y_1[n-2]$$


Here, $y_i$ denotes the filtered output after cascading through the $i$-th second-order section. The final result, $y[n]$, is equal to the output after the last SOS, $y_3[n]$.

 To apply the filter to the signal, I used the ```sosfiltfilt``` function which passes both in the forward direction (as shown in the equations above) and then in the reverse direction. This is done to ensure no lag in the phase of the signal is introduced. 

```python
f0 = 25000
nyq = 0.5 * f0
normalized_cutoff = cutoff / nyq
sos = butter(aif_order, normalized_cutoff, btype='low', analog=False, output='sos')
filtered_samples = sosfiltfilt(sos, samples, axis=0)
```

A raw sample from the LFP recordings is shown below (purple), along with the signal when filtered with a moving average (red) and low-pass Butterworth filter (blue). 

![](assets/images/butter_ma_og.png)


### Frequency domain

The spectral density of the signal describes the distribution of power in the signal into frequency components composing the signal. A straightforward method to estimate the spectral density is to use one of the fast Fourier Transform (FFT) algorithms, then take the squared magnitude of the output. Applying a FFT to the signal produces the spectral content of the signal. 

```python
freq_bins = np.fft.fftfreq(filtered_samples.shape[0], d=1/(fs/downsampling_factor))
freq_spectra = np.fft.fft(filtered_samples, axis = 0) 
psd_estimate = np.abs(freq_spectra)**2
```

We can do this over a number of channels individually, or average a few channels together. The result of the latter is shown below.  

![](assets/images/fft_psd.png)

Ignoring the 50Hz coming from a nearby outlet, the majority of the power of the signal resides below 15Hz, which is expected. Here's a more zoomed-in view

![](assets/images/fft_psd_zoomed.png)

The data is normalized to have zero mean, so the peak at 0Hz is not indicative of a "DC" element in the data. It may be an artifact of spectral leakage, caused by the discrete nature of the windowing used to calculate the FFT. 

| Oscillation | Frequency (Hz) |
| :--: | :--: |
| **Delta** | <5 |
| **Theta** | 5-10 |
| **Alpha** | 8-12 |

The signal frequencies mainly reside in the mid-theta and delta range. 


### The Hilbert transform

The equation for the Hilbert transform of a continuous-time signal is given below.
$$H\{x(t)\} = \frac{1}{\pi} \mathcal{P} \int_{-\infty}^{\infty} \frac{x(\tau)}{t - \tau} d\tau $$
In this equation, $\mathcal{P}$ refers to the Cauchy Principal Value of the following integral, a method used to handle integrals that are otherwise undefined due to singularities at certain points, like when $\tau = t$. 

#### What does this do? 

The Hilbert transform can be seen as a convolution with the function $\frac{1}{\pi t}$. In the frequency domain, convolving with this function applies a $-90$ degree phase shift to positive frequency components and a $+90$ degree phase shift to negative frequency components. 

Why is this useful? We can multiply the result, $H\{x(t)\}$, by complex $i$ (imparting a final $-90$ degree phase shift) to "restore" the positive frequency components (shift them back to their original phase) while simultaneously shifting the negative frequency ones an additional $+90$ degrees. Because the negative frequencies are now shifted by a full $+180$ degrees, if we add back the original signal, $x(t)$, to the result, the negative frequencies will be suppressed. This gives:

$$x_a(t) = x(t) + i * H\{x(t)\}$$

where $x_a(t)$ is called the *analytic signal*. If following the phase-shifts is confusing, hopefully the following gif will help. 

![](assets/images/analytic_signal.gif)

In practice, we apply the *discrete* Hilbert transform which is described as
$$ H\{x[n]\} = \sum_{m=-\infty}^{\infty} h[m]x[n-m]$$
where $h[m]$ can be thought of analogously to the continuous Hilbert transform's $\frac{1}{\pi t}$, but adapted for discrete signals.
$$
h[m] = \begin{cases} 
  0 & \mbox{if } m = 0 \\
  \frac{2}{\pi m} \sin\left(\frac{\pi m}{2}\right) & \mbox{if } m \mbox{ is odd} \\
  0 & \mbox{if } m \mbox{ is even}
\end{cases}
$$


## Finding Theta oscillations

To determine 

To be continued...

	- Theta / delta ratio


Notes: 
hardware only collects signals between 0.1 - 7600Hz
MNchbis 
	-L is 0-31
	-r is 32-63
Depth is 
16 channels on each side (only half of the probes are on)
Top one is 1.55mm depth from top of brain

Region: CA1 
- frequency domain , fourier transform 
- spectrogram before/after filtering 

Do signals propagate laterally in CA1? Reasoning for location of probe.

If probe is not in region of interest, how does it pick up on signals in CA1? 

Setup LFP in specific is designed to get synchronous firing, not general activity
