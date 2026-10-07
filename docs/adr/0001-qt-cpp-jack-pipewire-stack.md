# ADR-0001: Qt/QML UI, pure C++ engine and JACK over PipeWire

- Status: Accepted
- Date: 2026-10-07

## Context

The MVP is an Ubuntu/Linux desktop live-looper with four audio tracks, four MIDI tracks, fixed-bar looping, non-destructive overdub, a basic timeline, a built-in synth and sampler, and low-latency operation. The UI must be a native single-window application. The project does not include external plugin hosting in the first release.

The audio system must work with the user's PipeWire-based Ubuntu setup while retaining JACK interoperability. The application must be testable without a physical audio device and without a running audio server.

## Decision

Use:

- Qt 6/QML/Qt Quick Controls for the desktop UI;
- a pure C++ AudioEngine independent of Qt UI objects and file/network I/O;
- a small AudioBackend seam;
- JACK C API as the first backend, running through PipeWire's JACK compatibility layer;
- CMake and a first `.deb` distribution.

Keep a native PipeWire backend as a future adapter, not as an MVP dependency.

## Rationale

Qt provides the required native application window, QML controls, timeline rendering and offscreen UI testing without being made responsible for musical realtime processing. A separate C++ engine concentrates transport, DSP, MIDI, looping and export logic behind a deep interface.

The JACK client model gives a mature realtime callback, MIDI events with sample offsets and transport concepts while PipeWire supplies the system graph underneath. This avoids implementing the full native PipeWire context, registry, SPA format negotiation and session-policy lifecycle before the musical engine exists.

JUCE remains a viable alternative, but the MVP does not need its plugin-host layer and its licensing must be reviewed for the exact version and distribution model. Qt plus a separate engine keeps the architecture explicit and leaves the AudioBackend replaceable.

## Consequences

### Positive

- The engine can be tested with a fake backend.
- The UI thread cannot directly own realtime state.
- PipeWire/JACK interoperability is available without embedding an audio daemon.
- A native PipeWire adapter can be added behind the existing seam.
- The project can keep external plugin scope out of the first release.

### Negative

- The project must implement its own document model, timeline, transport, DSP, MIDI scheduling and export.
- Ubuntu installations may differ in PipeWire session-manager and JACK compatibility configuration.
- Qt licensing and exact module obligations must be reviewed before distribution.
- A future native PipeWire backend will require a separate integration and version matrix.

## Rejected alternatives

### JUCE as the whole application framework

Rejected for the MVP because external plugin hosting is out of scope and the exact JUCE licensing/distribution obligations require a separate decision. It can be reconsidered if implementation speed outweighs the licensing and framework trade-off.

### Qt Multimedia as the audio engine

Rejected because it does not provide the required JACK/PipeWire-native professional graph, sample-accurate musical transport and MIDI model for this MVP.

### RtAudio or PortAudio as the primary backend

Rejected because they provide general audio I/O but do not cover the required DAW transport, MIDI and graph semantics without additional layers.

### Native PipeWire first

Rejected for MVP sequencing because it introduces more low-level lifecycle, format negotiation, session-policy and compatibility work before the core musical workflow is proven.
