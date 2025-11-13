# Explainer for the WebCodecs VideoEncoder Manual Scalability Mode API

This proposal is an early design sketch by Chrome Media to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Chrome Media
- Chrome WebRTC

## Participate
- https://github.com/explainers-by-googlers/webcodecs-manual-scalability/issues
- https://github.com/w3c/webcodecs/issues/285

## Introduction

The WebCodecs API specifies a [scalabilityMode](https://www.w3.org/TR/webcodecs/#dom-videoencoderconfig-scalabilitymode) field in the [VideoEncoder](https://www.w3.org/TR/webcodecs/#video-encoder-config) object. This allows applications to enable support for a pre-defined set of basic scalability modes (e.g. *temporal layers*) as defined by the [SVC Extension](https://www.w3.org/TR/webrtc-svc/) for WebRTC.

However, since the scalability modes are somewhat limited, and by their nature, they are fixed at configuration time, meaning they are not useful for features that need reactive real-time updates, such as long-term references.

By evolving the WebCodecs VideoEncoder API with more advanced features, we make it possible to build more capable video conferencing and streaming applications that are better able to scale and adapt to varying network conditions.

## Goals

The main goal is to allow a large amount of flexibility for applications, by allowing them to configure the reference structure dynamically per frame. This simple fact will solve most of our use cases in one fell swoop:
* Support arbitrary temporal scaling factors
* Support quality layers
* Support per-layer rate control using external rate control
* Support flexible and reactive reference structures for variable fps scenarios
* Support LTR

In addition, we want this feature to be exposed in a manner which is as codec and implementation agnostic as possible.

## Non-goals

To constrain the problem to a reasonable scope, we propose that we at least initially not support:
* Internal (CBR/VBR) Rate Control in combination with manual scalability mode
  - Quantizer mode only, make both API surface and implementation easy
* Reference frame scaling
  - While spatial layers can technically be used, only 1:1 scaling factors.
  - Planned as follow-up work.
* Extensive capability querying
  - E.g. differences in max number of reference buffers vs max number of references per frame.
  - Planned as follow-up work.

## Use cases

The current scalability modes cover the basic temporal and spatial layering modes, but they still lack some functionality. Manual scalability unlocks new use cases:

* Arbitrary temporal scaling factors
  - Not restricted to a 2:1 factor per layer index. Allows scaling to lower bitrates.
* Quality layers
  - SVC layers without a spatial scaling factor. Allows quality scaling without sacrificing FPS where resizing is not feasible.
* Custom bitrate allocation
  - Allows per-layer target bitrates that better match the current content as well as remote receive endpoint conditions.
* Varying layer count dynamically based on content
  - E.g. screensharing that may swing wildly from ~0 fps to 60fps depending on user actions and content currently on screen.
* Long-term references
  - Allows quick recovery and reduced frame dropping in lossy conditions

## [Potential Solution]

[For each related element of the proposed solution - be it an additional JS method, a new object, a new element, a new concept etc., create a section which briefly describes it.]

```js
// Example implementing L1T3 scalability using manual scalability mode.

const pattern_index = frame_num % 4;
const temporal_pattern = [0, 2, 1, 2];
const temporal_index = temporal_pattern[pattern_index];

let videoEncoder = new VideoEncoder(...);
let config = {
  codec: 'av01.0.04M.08',
  width: 640,
  height: 360,
  bitrateMode: 'quantizer',
  scalabilityMode: 'manual'  // New mode using per-frame references
};

const encoder_support = await VideoEncoder.isConfigSupported(config);
if (encoder_support.supported) {
  await videoEncoder.configure(config);
  // Get all available reference buffers for this implementation.
  const buffers = videoEncoder.getAllFrameBuffers();
  if (buffers.length < 2) {
    // Error handling...
  }
}

// For each new `frame`:
let encode_options = {
  keyFrame: (frame_num == 0),
  av1: { quantizer: getQp(temporal_index) },
};

switch (pattern_index++) {
  case 0:
    encode_options.referenceBuffers = [buffers[0]];  // Reference T0
    encode_options.updateBuffer = buffers[0];        // Update T0
    break;
  case 1:
    encode_options.referenceBuffers = [buffers[0]];  // Reference T0, no update
    break;
  case 2:
    encode_options.referenceBuffers = [buffers[0]];  // Reference T0
    encode_options.updateBuffer = buffers[1];        // Update T1
    break;
  case 3:
    // Reference T1 and T0, no update.
    encode_options.referenceBuffers = [buffers[1], buffers[0]]; 
    break;
}

videoEncoder.encode(frame, encode_options);

```

See also discussion on [WebCodecs GitHub](https://github.com/w3c/webcodecs/issues/285#issuecomment-2125761556)

## Considered alternatives

The alternative to the above is to implement the desired use cases explicitly.
This would likely entail a large extension to the already very long list of scalability modes.
In addition, explicit LTR functionality would need to be exposed - with detailed life cycles for marking, acknowledging or retiring reference stages. This work would all be more implementation work and more spec work, for an end result that is still less flexible, not to mention harder to maintain and test.

## Security and Privacy Considerations

Exposing support for the new manual mode and the number of reference buffers in `isConfigSupported()` does technically expose a small amount of fingerprinting data. However, this data is already exposed in other ways. E.g., which browser vendor and version for SW codecs, in combination with the GPU type used for HW codecs. The latter is available via e.g. WebGPU, WebGL or just from inspecting the output of the WebCodecs video encoder.

For privacy and accessibility there is no difference.

As for security, it might seem scary that we expose some new “low-level knobs”. However, the state space here is small enough that it can be exhaustively covered (codec * reference buffer selection * qp). The same is not true for the existing CBR modes, where references may do weird things at certain bitrates or during transient events like scene changes. Thus, we actually think this is less of a security risk than traditional CBR encoders.

## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- Chrome : Positive
- Edge : No public signal
- Firefox : No public signal
- Safari : No public signal

## References & acknowledgements

https://github.com/w3c/webcodecs/issues/285

Many thanks for valuable feedback and advice from:

- Eugene Zemtsov
- Philip Eliasson
