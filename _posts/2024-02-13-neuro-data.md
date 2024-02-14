---
title: Analyzing local field potential recordings
published: true
permalink: neuro
---
{% include interactive.html %}

I was recently given about ~10GB of [local field potential](https://en.wikipedia.org/wiki/Local_field_potential)(LFP) recordings from electrodes placed in the brains (region?) of awake mice. The recordings are sampled at 25kHz (collecting a sample every 40 microseconds) across 64 channels. 
 
Given the sampling rate of 25kHz, which significantly exceeds the Nyquist frequency necessary for capturing the highest frequency oscillations observed in the brain ([fast ripples](https://onlinelibrary.wiley.com/doi/10.1111/j.1528-1157.1999.tb02065.x); 250-500Hz), I downsampled the data by a factor of 50 (new Nyquist frequency of 250Hz) to decrease the size of the data without compressing important information. Most brainwaves of interest (fast ripples are not common, occurring only during epilepsy) are well below 250Hz (more specific), so I am drastically oversampling (as desired). 

Before downsampling, I applied an anti-aliasing filter to prevent frequencies above the new Nyquist frequency (post-downsampling) from folding back into the lower-frequency range and creating aliasing errors. The ideal anti-aliasing filter should have a flat magnitude response to frequencies below the Nyquist frequency and a sharp drop off above it. This, of course, is not realizable in practice. 

Taking the moving average of the signal is a form of low-pass filtering. The frequency response of the moving average filter is shown below. The response is somewhat flat in the passable band but is 'bouncy' for frequencies higher than the cutoff. This is far from ideal.  

![](assets/images/ma_freqz.png)

Because we are oversampling by a large margin, we can afford to sacrifice steepness near the cutoff frequency in exchange for a flatter response in the passband. This makes the [Butterworth filter](https://en.wikipedia.org/wiki/Butterworth_filter), which has maximally flat frequency response in the passband,  a good choice. 

![](assets/images/Butterworth.png)

I applied a 6th-order Butterworth filter with a cutoff frequency at 250Hz. 

```python
f0 = 25000
nyq = 0.5 * f0
normalized_cutoff = cutoff / nyq
sos = butter(aif_order, normalized_cutoff, btype='low', analog=False, output='sos')
filtered_samples = sosfiltfilt(sos, samples, axis=0)
```

A raw sample from the LFP recordings is shown below (purple), along with the signal when filtered with a moving average (red) and low-pass Butterworth filter (blue). 
![](assets/images/butter_ma_og.png)

To be continued...