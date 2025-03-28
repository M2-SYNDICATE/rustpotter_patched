# Rustpotter (This is a patch that improves the operation and assembly of the module)

## An open source wakeword spotter forged in rust.

<div align="center">
    <img src="./logo.png?raw=true" width="400px"</img> 
</div>

## Overview

The aim of this project is to detect specific wakewords in a live audio stream.

Rustpotter allows two detection methods, both based on using the [mel frequency cepstral coeﬃcients (mfccs)](https://en.wikipedia.org/wiki/Mel-frequency_cepstrum) of the audio, exposed as the two kind of wakewords:

- Wakeword References: When using this kind of wakeword rustpotter does the detection by measuring the similarity of the live stream audio mfccs with the mfccs of the records used to build the wakeword reference, using the [dynamic time warping](https://en.wikipedia.org/wiki/Dynamic_time_warping) algorithm.
Creating a functional wakeword reference requires 3 to 8 wav records.
Computation increases based on the number of records used to create the wakeword reference.
It has less accuracy than a wakeword model trained with enough data.

- Wakeword Models: When using these files rustpotter will pass the live stream audio mfccs to a classification neural network and rely on its predictions to do the detections.
Training a functional wakeword model requires a training and testing sets of wav records which need to be tagged (contains [label] in its file name, where 'label' is the tag the network should predict for that audio segment) or untagged (equivalent to contain [none] on the filename).
The size of the wakeword model is based on the model type you choose, and the audio duration it was trained on (which is defined by the max audio duration found on the training set).
When trained with enough data, all models types offer a pretty good experience (very low missing or false detections).


## Libraries

You can use rustpotter on several platforms through different programming interfaces.

* [rustpotter-cli](https://github.com/GiviMAD/rustpotter-cli): Use Rustpotter on the `command line`. (pre-build binaries for Window, macOs and Linux)
* [rustpotter-java](https://github.com/GiviMAD/rustpotter-java): Use Rustpotter on `Java`. (Available on Maven with support for Window, macOs and Linux)
* [rustpotter-worklet](https://github.com/GiviMAD/rustpotter-worklet): Use Rustpotter in the `browser` as a Web Audio API processor node.

## Audio Format Support

Rustpotter works internally in PCM 32bit float mono audio at 16000Hz.

It handles pcm audio at any sample rate, only uses the first channel data and support following sample encodings:

- PCM signed int samples of 8, 16 or 32 bits.
- PCM float samples of 32 bits.

For creating the wakeword files you can use wav records matching the supported format, the audio format is read from the wav header.

On detection time the library consumes raw data, the available audio format options need to be configured correctly.

## Detection Mechanism Overview

Rustpotter process the audio data and keeps a window of mfccs vectors (can be seen as a matrix of mfccs) that grows until the length needed by the loaded wakewords.

The input length requested by Rustpotter varies depending on the audio format, it's equivalent to 30ms of audio. Internally, it generates a vector of mfccs for each 10ms of audio, meaning the audio window is updated 3 times each time you call the process method.

From the moment the window has enough size, Rustpotter starts scoring the window on each update in order to find a successful detection (score is over the defined `threshold`).

A detection is considered a `partial detection` (not emitted) until n more updates are processed (half of the length of the feature window). If in the meantime a detection with a higher score is found, it replaces the current partial detection, and this countdown is reset.

### The Score

The score is a numeric value in range 0 - 1 that represents the accuracy of the detection.

When using a wakeword model the score represents the inverse similarity between the predicted label and the prediction for the `none` label.

When using a wakeword reference the score represents the aggregated similarity against the mfccs of each of the records used on creations.
Calculated in base to the score mode option.

#### Score Mode

When using a wakeword reference rustpotter needs to unify the scores against the mfccs of each of the records used on creations.

You can configure how this is done using the `score_mode` option. The following modes are available:

* Avg: Use the averaged value (mean).
* Max: Use the maximum value.
* Median: Use the median. Equivalent to P50.
* P25, P50, P75, P80, P90, P95: Use the indicated percentile value. Linear interpolation between the values is used on non-exact matches.

### The Averaged Score

Another numeric value in range 0 - 1 calculated on detection.

When using a wakeword reference the average threshold represents the similarity of the current audio mfccs against a single mfccs matrix generated by averaging the mfccs of the records used on creations. The averaged threshold can be used to reduce cpu usage as it aborts the detection when the averaged score is not enough.

When using a wakeword model it the inverse similarity between the predicted label and the prediction for next matched label. It will match the score unless you are using a model trained for detect more that one label.

Remember you can set the `avg_threshold` config to zero to disable using this score.

### Partial detections

To discard false detections, you can require a certain number of partial detections to occur.
This is configured through the `min_scores` config option.

### Detection

A successful Rustpotter detection provides you with some relevant information about the detection process so you know how to configure the detector to achieve a good configuration (minimize the number of misses/false detections).

It looks like this when using a wakeword reference:

```rust
RustpotterDetection {
    /// Detected wakeword name.
    name: "hey home",
    /// Detection score against the averaged features matrix. (zero if disabled)
    avg_score: 0.41601, 
    /// Detection score. (calculated from the scores using the selected score mode).
    score: 0.6618781, 
    /// Detection score against the mfccs of each record used on creation.
    scores: {
        "hey_home_g_5.wav": 0.63050425, 
        "hey_home_g_3.wav": 0.6301979, 
        "hey_home_g_4.wav": 0.61404395, 
        "hey_home_g_1.wav": 0.6618781, 
        "hey_home_g_2.wav": 0.62885964
    },
    /// Number of partial detections.
    counter: 40,
    /// Gain applied by the gain-normalizer or 1.
    gain: 1.,
}
```

It looks like this when using a wakeword model:

```rust
RustpotterDetection {
    /// Detected label.
    name: "hey home",
    /// Inverse similarity against the seconds more probable label. (zero if disabled)
    avg_score: 0.9994159,
    /// Inverse similarity against the 'none' label probability.
    score: 0.9994159,
    /// Label probabilities.
    scores: {
        "hey home": 7.999713,
        "none": -10.5787945
    }
    /// Number of partial detections.
    counter: 28,
    /// Gain applied by the gain-normalizer or 1.
    gain: 1.,
}
```

Rustpotter exposes a reference to the current partial detection that allows read access to it for debugging purposes.

## Model Types

The following sizes are for models files trained on 1950ms of audio.
Those are: 

- Tiny: 2 linear layer network (320K).
- Small: 3 linear layer network (768K).
- Medium: 3 linear layer network (2.1M).
- Large: 3 linear layer network (3.1M).

## Audio Filters

Rustpotter includes two audio filter implementations: a `gain-normalizer filter` and a `bass-pass filter`.

These filters are disabled by default, and their main purpose is to improve the detector's performance in the presence of noise.

## Web Demos

 The [spot demo](https://givimad.github.io/rustpotter-worklet-demo/) is available so you can quickly try out Rustpotter using a web browser.

 It includes some models generated using multiple voices from a text-to-speech service.
 You can also load your own ones.

 The [wakeword reference generator demo](https://givimad.github.io/rustpotter-create-model-demo/) is available so you can quickly record samples and generate Rustpotter wakeword references using your own voice.

Please note that `both run entirely on your browser, your voice is not sent anywhere`, they are hosted using Github Pages.


## Changelog overview

A minimal overview of the changes introduced on each major version.

v3:

- Introduce wakeword models and refactor previous functionality as wakeword references.
- Allow configurable mfccs number by wakeword (does not support adding wakewords of different sizes).

v2:

- Rebuild the library, incompatible with v1.
- Add audio filters.

v1:

- Initial version.

## Basic Usage

```rust
use rustpotter::{Rustpotter, RustpotterConfig, Wakeword};
// assuming the audio input format match the rustpotter defaults
let mut rustpotter_config = RustpotterConfig::default();
// Configure format/filters/detection options
...
// Instantiate rustpotter
let mut rustpotter = Rustpotter::new(&rustpotter_config).unwrap();
// load a wakeword
rustpotter.add_wakeword_from_file("./tests/resources/hey_home.rpw").unwrap();
// You need a buffer of size `rustpotter.get_samples_per_frame()` when using samples.
// You need a buffer of size `rustpotter.get_bytes_per_frame()` when using bytes.
let mut samples_buffer: Vec<i16> = vec![0; rustpotter.get_samples_per_frame()];
// while true { Iterate forever
    // fill the buffer with the required samples
    ...
    let detection = rustpotter.process(samples_buffer);
    if let Some(detection) = detection {
        println!("{:?}", detection);
    }
// }
```

## References

This project started as a port of the project [node-personal-wakeword](https://github.com/mathquis/node-personal-wakeword) and it's based on public available articles, and uses a ton of amazing crates.

## Motivation

The motivation behind this project is to learn about audio analysis and the Rust language/ecosystem.

As such, this is not intended to be a production-grade tool, but with a well trained wakeword model it achieves the quality expected.

## Contributing

Feel free to suggest or contribute any improvements that you have in mind, either to the code or the detection process.

If you need any assistance, please feel free to open an issue.

Best regards!

