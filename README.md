# Zapline-plus
Zapline-plus is a wrapper for Zapline that automatically removes spectral peaks like line noise from your data while ensuring minimal negative impact, preserving both the non-noise spectrum and the data rank. It searches for noise frequencies, divides the data into spatially stable chunks, and adapts the cleaning strength automatically. A detailed plot is created.

# NOTES 

This is a fork of the repository by Marius Klug. Building on their excellent work, this version offers a few improvements.

## Motivation and new features

Data contaminated by both line noise (~50 Hz) and tVNS (~25 Hz). These are not exact harmonics of each other (line noise usually slightly above 50 Hz, tVNS just below 25 Hz), resulting in two sharp peaks in the spectrum very close to each other (line and second harmonic of tVNS). In addition, we notices pronounced peaks near 35 Hz, 15 Hz and, in a few data sets, 5 Hz, which are probably related to the tVNS stimulation. 

These artefacts were reduced but not entirely eliminated by applying the published version of zapline plus, probably due suboptimal implementation of noise removal and adaptive procedure. For instance:

1) Noise removal was sometimes too broad, resulting in wider troughs in the "cleaned" spectrum, instead of just removing the specific frequency peak. This is probably due to the limited default spectrum resolution used by `nt_zapline_plus`(nfft = 1024). At a sampling rate of 500 Hz, this would result in a frequency resolution for the spectrum of about 0.5 Hz, which is imprecise both compared to the usually very distinct noise peaks and also in relation to some of the default parameters for post-cleaning checking, in particular `detailedFreqBoundsUpper = [-0.05, 0.05]` and `detailedFreqBoundsLower = [-0.4, 0.1]`. 

We address this by introducing two features, specified by two parameters, 

- `nfft`: window size to use for FFT in `nt_zapline_plus` and `nt_bias_fft` (set to 4096 for data downsampled to 250 Hz, corresponding to a frequency resolution of ~ 1/16 = 0.0625)
- `noiseFreqWindow`: window size around each line frequency and harmonice to use in `nt_zapline_plus` and `nt_bias_fft`. This allows filtering not just for a single frequency but a small window (by default the same as `detailedFreqBoundsUpper = [-0.05, 0.05]`). We suppose that this allows to target noise peaks more specifically.

2) Both individual peak detection and post-cleaning checks (in the adaptive procedure) compare the spectrum near the targeted noise frequency to average and inter-quantile range (from median to lowest 5%) to the average on both sides of the spectrum. This works well when the spectrum is "flat" around the target frequency (same average power to left and right); however, when average power changes linearly  (e.g. decreases or increases from lower to higher frequencies) or non-linearly (e.g., higher power on both sides compared to immediate neighborhood), both the estimated average and the variability are strongly affected. 

We address this with a new feature and parameters: 

- `detrendSpectrum`: Set to 0 to use original version; set to > 0 to use polynomial fit of that order to estimate "slow trend" and residuals of spectrum around the target frequencies. These are then used to define thresholds both for peak detection and post-cleaning checking. 
- `minNoiseDelta`: If the spectral power varies very little across frequencies around the target frequency, it may be desirable to introduce a minimal deviation for computing the "below" and "above" thresholds. This can be done by setting this parameter > 0 (e.g., a value of 0.5 dB corresponds to a minimal power ratio for thresholds of about $10 ^{0.05} \approx 1.12$, i.e., around +/- 10%).

3) It may be desirable to use different parameters for different noise frequencies. This is now possible by providing a vector of values (same length as `noisefreqs`) rather than a single value for the following parameters: `searchIndividualNoise, fixedNremove, adaptiveNremove, detectionWinsize, nHarmonics`

4) We introduce two additional features that may be useful in some cases

- `nHarmoncis` allows specifying the number of harmonics to include in the noise removal procedure implemented in `nt_zapline_plus`. The default option is to include all harmonics (up to Nyquist frequency), but it may be desirable to apply the cleaning for specifically. [TODO: we may add the option to not only specify the number of harmonics, but also a list of harmonics to use, e.g. only odd: `1:2:11`].
- `chunkIndices`: Instead of relying on the automatich chunking, it may be desirable to specify chunks to correspond to experimental blocks, ignoring breaks between blocks, or excluding DCC artefacts from the analysis. This can be dones using this parameter, provided either as a list (with chunk breaks) or a Nx2 matrix (chunk start and end in each row). [TODO: this could be extended by allowing automated chunking respecting the suppolied chunk boundaries; also, exclusion of parts segments is not yet handled by the post-cleaning check, which always computes the spectrum on the entire data set]


## Further ideas

- Refactor `clean_data_with_zapline_plus.m`, to clarify structure and remove redundancies 
- Respect epoch structure, allow ignoring parts, etc. (take into account also for "whole-dataset FFT")
- Replace `nHarmonics` by alternative option to specify as list of harmonics to take into account (e.g., only odd ones: [1, 3, 5, ...]).
- Quite often, removing the 25 Hz-assocoiated components with zapline, also reduces power in the typical artifact components (odd harmonics of 5 Hz). Would it make sense to either enforce a minimal number of components for 25 Hz, or have an additional criterion, centered at, say, 5 and 35 Hz?
- Or might it make sense to combine 24.96... and 50 Hz in one removal operation in `nt_zapline_plus` and `nt_bias_fft`? With the hope to capture also the interaction noise between the two?

# Quick start

If you just want to download the package and use Zapline-plus with any data matrix, you can feed that data and the sampling rate directly in like this:
```matlab
cleanedData = clean_data_with_zapline_plus(data,srate);
```

Or if you live in the EEGLAB universe, you can install Zapline-plus via the EEGLAB plugins manager and then run the cleaning either via the `Tools` GUI or in the command line like this:

```matlab
EEG = clean_data_with_zapline_plus_eeglab_wrapper(EEG,struct('noisefreqs','line')) % specifying the config is optional
```
# Detailed user guide
Please check out the wiki articles for a detailed guide on [how to use Zapline-plus](https://github.com/MariusKlug/zapline-plus/wiki/Zapline-plus-user-guide) and [how to interpret the plot](https://github.com/MariusKlug/zapline-plus/wiki/Zapline-plus-plot).

# Please cite

Klug, M., & Kloosterman, N. A. (2022). Zapline-plus: A Zapline extension for automatic and adaptiveremoval of frequency-specific noise artifacts in M/EEG. Human Brain Mapping,1–16. https://doi.org/10.1002/hbm.25832

de Cheveigne, A. (2020). ZapLine: a simple and effective method to remove power line artifacts. NeuroImage, 1, 1-13. https://doi.org/10.1016/j.neuroimage.2019.116356

Dependencies of Noisetools are provided with permission by Alain de Cheveigné. Please visit the original repository for more info and additional noise removal tools: http://audition.ens.fr/adc/NoiseTools/

# Requirements
- Signal Processing Toolbox
- Statistics Toolbox
