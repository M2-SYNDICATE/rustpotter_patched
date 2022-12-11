# Rustpotter

## An open source wake word spotter forged in rust.

<div align="center">
    <img src="./logo.png?raw=true" width="400px"</img> 
</div>

## Description

This project allows to detect specific wake words on a live audio stream.

 To do so it generates a set of features from some audio samples to later compare them with the features generated from a live stream, to calculate the probability of a match.

The features can be loaded from a previous generated model file or extracted from the samples before starting to process the live streaming.

## Web Demos

 This [spot demo](https://givimad.github.io/rustpotter-worklet-demo/) is available so you can quickly try out Rustpotter using a web browser.

 It includes some models generated using multiple voices from a text-to-speech service.
 You can also load your own ones.

 This [model generator demo](https://givimad.github.io/rustpotter-create-model-demo/) is available so you can quickly generate rustpotter models using your own voice.

Please note that the audio processing runs entirely on your browser, your voice is not sent anywhere.

## Audio Format

This project uses wav, it works internally with the spec 48000hz 16 bit 1 channel int, but allows to configure the detector to work with other specs.

The detector configuration is ignored when adding a keyword with samples as the wav spec is read from each file header (so raw samples are not allowed).

## Related projects

* [rustpotter-cli](https://github.com/GiviMAD/rustpotter-cli): Use rustpotter on the command line (the only that exposes model generation at the moment).
* [rustpotter-java](https://github.com/GiviMAD/rustpotter-java): Use rustpotter on java.
* [rustpotter-wasm](https://github.com/GiviMAD/rustpotter-wasm): Generator for javascript + wasm module.
* [rustpotter-worklet](https://github.com/GiviMAD/rustpotter-worklet): Ready to use package for web (runs rustpotter-web in a worklet).
* [rustpotter-web](https://www.npmjs.com/package/rustpotter-web): Npm package generated with rustpotter-wasm targeting web.

## Versioning

Rustpotter versions prior to v1.0.0 are not recommended, model compatibility was broken frequently.

Since 1.0.0 it will stick to [semver](https://semver.org), and a model compatibly break will be  marked by a MAJOR version change, same will apply for related packages (cli, wasm-wrapper, java-wrapper...).

## The Averaged Threshold 

When generating a model, it will contain one set of features for each wav sample that you use to generate it, but also will contain and extra set of features that is generated by averaging the other sets.

When configuring the averaged threshold, you are configuring the minimum score that the current frame should obtain against this extra set of features to continue with the detection process (which is scoring against each of the other feature sets).

You will get lower scores against the averaged feature set and more false positives, but with the correct value you will avoid running the whole detection process in most of the audio frames (you will be running the comparison once instead of one per sample). This reduces a lot the cpu usage when you are talking or watching tv near the detector, as the comparison step is the more cpu expensive.

## Some examples:

### Create wakeword model:
```rust
let mut detector_builder = detector::FeatureDetectorBuilder::new();
    let mut word_detector = detector_builder.build();
    let name = String::from("hey home");
    let path = String::from("./hey_home.rpw");
    word_detector.add_wakeword(
        name.clone(),
        false,
        None,
        None,
        vec!["./<audio sample path>.wav", "./<audio sample path>2.wav", ...],
    );
    match word_detector.create_wakeword_model(name.clone(), path) {
        Ok(_) => {
            println!("{} created!", name);
        }
        Err(message) => {
           panic!(message);
        }
    };
```


### Spot wakeword:
```rust
    let mut detector_builder = detector::FeatureDetectorBuilder::new();
    detector_builder.set_threshold(0.4);
    detector_builder.set_sample_rate(16000);
    let mut word_detector = detector_builder.build();
    let result = word_detector.add_wakeword_from_model(command.model_path, command.average_templates, true, None);
    if result.is_err() {
        panic!("Unable to load wakeword model");
    }
    while true {
        let mut frame_buffer: Vec<i16> = vec![0; word_detector.get_samples_per_frame()];
        // fill the buffer
        ...
        let detection = word_detector.process_pcm_signed(frame_buffer);
        if detection.is_some() {
            println!("Detected '{}' with score {}!", detection.unwrap().wakeword, detection.unwrap().score)
        }
    }

```

### References

This project started as a port of the project [node-personal-wakeword](https://github.com/mathquis/node-personal-wakeword) and uses the method described in this medium [article](https://medium.com/snips-ai/machine-learning-on-voice-a-gentle-introduction-with-snips-personal-wake-word-detector-133bd6fb568e).

### Motivation

The motivation behind this project is to learn about audio analysis and Rust.
Also to have access to an open source wakeword spotter to use in other open projects.

Feel encourage to suggest any improvements or fixes.

