# DAW Domain Glossary

## Project

A self-contained folder containing the musical document, source audio, overdub layers, MIDI data, instrument settings and recovery snapshots.

## Session

The active playback and recording context: BPM, musical grid, transport state, track state and current project snapshot.

## Transport

The musical clock and playback/recording state. It owns sample-position time, BPM, 4/4 bar boundaries, count-in and quantised start/stop/record operations.

## Audio Track

A track that receives audio input and renders a base loop plus zero or more overdub layers. The MVP supports at most four audio tracks.

## MIDI Track

A track containing timed MIDI events routed to an internal instrument. The MVP supports at most four MIDI tracks.

## Source Take

The original recorded audio retained before boundary quantisation or later overdub operations. A source take is never silently replaced by an overdub.

## Loop

A fixed-bar musical segment with a start, end and transport relationship. A loop has audio or MIDI content and can be instantiated on the timeline.

## Overdub Layer

A non-destructive audio recording added on top of an existing loop. A layer can be enabled, disabled or deleted without rewriting the source take.

## Timeline Instance

A reference to a loop placed at a position on the basic linear timeline. Copy, move, trim and delete operate on instances, not on the original source take.

## Instrument

An internal sound generator used by a MIDI track. MVP instruments are one subtractive synthesizer and one WAV sampler.

## Suggestion

A typed, previewable AI result containing either MIDI material or parameters for an internal instrument. A suggestion is not applied until the user confirms it.

## Voice Features

Locally extracted characteristics of a vocal recording used for a synth recommendation. Voice features are not the original vocal recording and are the only audio-derived data sent to the provider.

## AudioEngine

The realtime-capable module that owns transport rendering, looping, overdub mixing, MIDI scheduling, instruments and track output behind a small command/snapshot interface.

## AudioBackend

The interface and adapter through which AudioEngine exchanges realtime audio/MIDI buffers and backend status with PipeWire/JACK. It is not the musical transport or project model.

## Project Bundle

The portable project folder containing the manifest, audio assets, MIDI, presets, recovery snapshots and exports using relative references.

## Stem

An exported audio render for one track or logical source, separate from the stereo mix.

## Scene Matrix

A grid of clips/scenes for launching combinations of loops. It is explicitly outside the MVP.
