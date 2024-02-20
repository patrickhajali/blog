---
title: Analyzing local field potential recordings
published: true
permalink: neuro
---
All code accessible [here](https://github.com/patrickhajali/neuro).

I was given about ~10GB of [local field potential](https://en.wikipedia.org/wiki/Local_field_potential)(LFP) recordings from electrodes placed in the brains of awake mice. The recordings are sampled at 25kHz (collecting a sample every 40 microseconds) across 32 channels (half in the left hemisphere, half in the right). 

The start of the probes, which contain the electrodes, are placed at a depth of 1.55mm from the outermost layer of the brain. They are positioned to rest directly above the CA1 region (which I believe is in hippocampus) to record the oscillatory activity in the region. 

![](assets/images/p56_coronal.jpg){:.centered}

The goal here is to get a closer look at the data, learn about some relevant signal processing methods, and see if I can locate (in time) theta (5-10Hz in mice) oscillations. 

## The data 

Below is a plot of a few channels of a sample of the data. Note that the compression used to publish the interactive plot here introduced a lot of noise, so don't try to zoom too closely.  

{% include interactive.html %}{:.centered}

#### Downsampling 

Given the sampling rate of 25kHz, which significantly exceeds the Nyquist frequency necessary for capturing the highest frequency oscillations observed in the brain ([fast ripples](https://onlinelibrary.wiley.com/doi/10.1111/j.1528-1157.1999.tb02065.x); 250-500Hz), I downsampled the data by a factor of 50 (new Nyquist frequency of 250Hz) to decrease the size of the data without compressing important information. Ranging from 5-10Hz in mice, theta oscillations occur well below 250Hz, so I am drastically oversampling, which is good. 

#### Anti-aliasing 

Before downsampling, I applied an anti-aliasing filter to prevent frequencies above the new Nyquist frequency (post-downsampling) from folding back into the lower-frequency range and creating aliasing errors. The ideal anti-aliasing filter should have a flat magnitude response to frequencies below the Nyquist frequency and a sharp drop off above it. This, of course, is not realizable in practice. 

Taking the moving average of the signal is a form of low-pass filtering. The frequency response of the moving average filter is shown below. The response is somewhat flat in the passable band but is 'bouncy' for frequencies higher than the cutoff. This is not ideal.  

![](assets/images/ma_freqz.png){:.centered}

Because we are oversampling by a large margin, we can afford to sacrifice steepness near the cutoff frequency in exchange for a flatter response in the passband. This makes the [Butterworth filter](https://en.wikipedia.org/wiki/Butterworth_filter), which has maximally flat frequency response in the passband,  a good choice. 

![](assets/images/Butterworth.png){:.centered}

I used ```scipy.signal.butter``` to generate a 6th-order Butterworth filter with a cutoff frequency at 250Hz. We can decompose the transfer function of the filter into a cascade of $N$-many second-order filters, taking the form 

$$ H(z) = \prod_{i=1}^{N} \frac{b_{0,i} + b_{1,i}z^{-1} + b_{2,i}z^{-2}}{1 - a_{1,i}z^{-1} - a_{2,i}z^{-2}}$$

Each second-order filter is referred to as a second-order section (SOS). This method is used to improve the numerical stability of the overall filter. High-order filters, combined with very large or small coefficients, can introduce significant rounding errors due to the finite-precision of floating point numbers, so it makes sense to decompose them into lower-order filters. 

The overall filter is the cascaded combination of these second-order sections. For $N=3$, applying the filter in the time-domain yields the following equations. 

$$y_1[n] = b_{0,1} \cdot x[n] + b_{1,1} \cdot x[n-1] + b_{2,1} \cdot x[n-2] - a_{1,1} \cdot y_1[n-1] - a_{2,1} \cdot y_1[n-2]$$

$$y_2[n] = b_{0,2} \cdot y_1[n] + b_{1,2} \cdot y_1[n-1] + b_{2,2} \cdot y_1[n-2] - a_{1,2} \cdot y_2[n-1] - a_{2,2} \cdot y_2[n-2]$$


$$
y_3[n] = b_{0,3} \cdot y_2[n] + b_{1,3} \cdot y_2[n-1] + b_{2,3} \cdot y_2[n-2] - a_{1,3} \cdot y_3[n-1] - a_{2,3} \cdot y_3[n-2]
$$


$$y[n] = y_3[n]$$

Here, $y_i$ denotes the filtered output after cascading through the $i$-th second-order section. The final result, $y[n]$, is equal to the output after the last SOS, $y_3[n]$.

 To apply the filter to the signal, I used the ```sosfiltfilt``` function which passes both in the forward direction (as shown in the equations above) and then in the reverse direction. This is done to ensure no lag in the phase of the signal is introduced. 

```python
fs = 25000
nyq = 0.5 * fs
normalized_cutoff = cutoff / nyq
sos = butter(aif_order, normalized_cutoff, btype='low', analog=False, output='sos')
filtered_samples = sosfiltfilt(sos, samples, axis=0)
```

A raw sample from the LFP recordings is shown below (purple), along with the signal when filtered with a moving average (red) and low-pass Butterworth filter (blue). 

![](assets/images/butter_ma_og.png){:.centered}


### Frequency domain

The power spectral density of the signal describes the distribution of power in the signal into frequency components composing the signal. A straightforward method to estimate the spectral density is to use one of the fast Fourier Transform (FFT) algorithms, then take the squared magnitude of the output. Applying a FFT to the signal produces the spectral content of the signal. 

```python
freq_bins = np.fft.fftfreq(filtered_samples.shape[0], d=1/(fs/downsampling_factor))
freq_spectra = np.fft.fft(filtered_samples, axis = 0) 
psd_estimate = np.abs(freq_spectra)**2
```

We can do this over a number of channels individually, or average a few channels together. The result of the latter is shown below.  

![](assets/images/fft_psd.png){:.centered}

Ignoring the 50Hz coming from a nearby outlet, the majority of the power of the signal resides below 15Hz, which is expected. Here's a more zoomed-in view

![](assets/images/fft_psd_zoomed.png){:.centered}

The data is normalized to have zero mean, so the peak at 0Hz is not indicative of a "DC" element in the data. It may be an artifact of spectral leakage, caused by the discrete nature of the windowing used to calculate the FFT. 

| Oscillation | Frequency (Hz) |
| :--: | :--: |
| **Delta** | <5 |
| **Theta** | 5-10 |
| **Alpha** | 8-12 |

The signal frequencies mainly reside in the mid-theta and delta range. Before I can determine specifically when theta oscillations occur, it's essential to introduce a key operation: the Hilbert transform.

### The Hilbert transform

The equation for the Hilbert transform of a continuous-time signal is given below.

$$H\{x(t)\} = \frac{1}{\pi} \mathbf{P} \int_{-\infty}^{\infty} \frac{x(\tau)}{t - \tau} d\tau $$

$\mathbf{P}$ refers to the Cauchy Principal Value of the following integral, a method used to handle integrals that are otherwise undefined due to singularities at certain points, like when $\tau = t$. 

On discrete data, like ours, we apply the *discrete* Hilbert transform (DHT) which is described as

$$ H\{x(n)\} = \sum_{m=-\infty}^{\infty} h(m)x(n-m)$$


where $h(m)$ can be thought of analogously to the continuous Hilbert transform's $\frac{1}{\pi t}$ filter (see below), but adapted for discrete signals.

$$
h(m) = \begin{cases} 
  0 & \mbox{if } m = 0 \\
  \frac{2}{\pi m} \sin\left(\frac{\pi m}{2}\right) & \mbox{if } m \mbox{ is odd} \\
  0 & \mbox{if } m \mbox{ is even}
\end{cases}
$$

#### What does it do? 

The Hilbert transform can be viewed as a convolution with the function  $c(t) = \frac{1}{\pi t}$. In the frequency domain, convolving with this function applies a $-90$ degree phase shift to positive frequency components and a $+90$ degree phase shift to negative frequency components. 

Why is this useful? We can multiply the result of the Hilbert transform by $i$ (imparting a final $-90$ degree phase shift) to "restore" the positive frequency components while simultaneously shifting the negative frequency ones an additional $+90$ degrees. Because the negative frequencies are now shifted by a full $+180$ degrees, if we add back the original signal, $x(t)$, to the result, the negative frequencies will be suppressed. 


![](assets/images/analytic_signal.gif){:.centered}

The end result is referred to as the *analytic signal*.

$$x_a(t) = x(t) + i * H\{x(t)\}$$

From the analytic signal, $x_a(t)$, we can extract the amplitude of the signal

$$ A(t) = |x_a(t)| = \sqrt{\Re\{x_a(t)\}^2 + \Im\{x_a(t)\}^2}$$


and the instantaneous phase

$$\phi(t) = \arg(x_a(t)) = \tan^{-1}\left(\frac{\Im\{x_a(t)\}}{\Re\{x_a(t)\}}\right)$$

which we can use to get the instantaneous frequency

$$f(t) = \frac{1}{2\pi} \frac{d\phi(t)}{dt}
$$


If negative frequencies were not suppressed in the analytic signal, the instantaneous frequency would be misleading, as negative frequencies can cause the phase to advance or retreat in a manner that does not correspond to the physical reality of the signal's frequency content.

Using ```scipy```'s [```signal.hilbert```](https://docs.scipy.org/doc/scipy/reference/generated/scipy.signal.hilbert.html) function and some ```numpy``` operations:

```python
analytic_signal = signal.hilbert(data)
amplitude_envelope = np.abs(analytic_signal)
instantaneous_phase = np.unwrap(np.angle(analytic_signal))
instantaneous_frequency = (np.diff(instantaneous_phase) / 
			(2.0*np.pi) * (fs/downsampling_factor))
```

Note that we multiply by the post-downsampling sampling rate ```fs/downsampling_factor``` to ensure the instantaneous frequency is in Hz. 

The amplitude envelope (orange) and instantaneous frequency (bottom plot) for a 10s sample are shown below. Note that the original signal below was bandpassed for theta frequencies. More on this in the next section. 

![](assets/images/hilbert.png){:.centered}

## Finding Theta oscillations

An objective of mine is to get an estimate of when, in the recordings, there are theta oscillations occurring. The first step is to apply a bandpass filter with a passband range of 5 to 10 Hz (theta) to the recording. 

```python
nyq = 0.5 * fs
lowcut = 5
highcut = 10
low = lowcut / nyq
high = highcut / nyq
sos = signal.butter(order, [low, high], btype='band', analog=False, output='sos')
theta = signal.sosfiltfilt(sos, samples, axis=0)
```

Initially, analyzing the signal filtered in the theta band seemed sufficient. But after digging deeper (specifically, [Király et. al.](https://www.nature.com/articles/s41467-023-41746-0) and [Kocsis et. al.](https://www.cell.com/cell-reports/fulltext/S2211-1247(22)00958-5?_returnURL=https://linkinghub.elsevier.com/retrieve/pii/S2211124722009585?showall%3Dtrue)), I found there is more nuance. Specifically, I found that the ratio of the amplitude envelopes of theta-filtered and delta-filtered signals, rather than the theta signal alone, is commonly used to determine the presence of theta oscillations. 

My guess is that dividing by the delta amplitude is used to normalize for different levels of background activity across different mice/experimental setups. Normalizing by delta makes sense, as delta and theta oscillations are thought to be associated with different states of brain activity. Hypothetically, this means that, when theta is present, delta should be low (and vice-versa), thus amplifying the signal we are trying to capture. 

The full method consists of the following: 
1. Extract a 'theta' signal by bandpassing the original signal with a passband in the theta range.
2. Extra a 'delta' signal through the same method.
3. Get the analytic signal of both the theta and delta filtered signals.
4. Compare the ratio of the magnitude (amplitude envelope) of the two analytic signals. 
5. If the ratio exceeds a set threshold for a given number of seconds, consider that time-window positive for theta. 

Code for steps 4 and 5 is shown below. The hyperparameters ```ratio_threshold``` and ```min_seconds``` were hand selected. 

```python
ratio = theta_amplitude / delta_amplitude
ratio_threshold = 1
above_threshold = np.where(ratio > ratio_threshold, 1, 0)

min_seconds = 2
window_size = int(min_seconds * fs_new)
convolved = np.convolve(above_threshold, np.ones(window_size)/window_size, mode='same')

theta_on = np.zeros_like(above_threshold)
for i in range(len(convolved)):
# if ratio is met at a certain point, mark the entire window as a theta cycle
if convolved[i] >= 1:
	start_index = max(i - window_size // 2, 0)
	end_index = min(i + window_size // 2, len(theta_on))
	theta_on[start_index:end_index] = 1
```

The results are plotted below. The original signal is shown in blue. The red bands are present when the mice was running (this was recorded during the experiment). Running is a behavioral state thought to be associated with theta oscillations, so the correlation is expected.

![](assets/images/theta_on.png){:.centered}

Pretty cool! 