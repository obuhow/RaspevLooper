# DAW MVP — план реализации

Статус: согласованный план

Цель плана — получить проверяемый MVP, а не сразу полноценную DAW. Каждая фаза должна оставлять запускаемый и тестируемый vertical slice.

## Phase 0 — Репозиторий и воспроизводимая сборка

**Результат:** пустое Qt/C++ приложение и CMake/CTest pipeline.

- зафиксировать compiler standard, CMake и Qt 6 minor;
- создать target layout: engine, backend, UI, app, tests;
- добавить dependency manifest и license notices;
- настроить offline engine tests и QML offscreen tests;
- добавить `.gitignore`, CI configuration и developer README.

**Готово, когда:** чистая сборка и тесты проходят на Ubuntu без аудиоустройства.

## Phase 1 — Глубокий AudioEngine seam

**Результат:** deterministic engine с fake backend.

- определить небольшой AudioBackend interface;
- определить command queue и immutable state snapshots;
- реализовать sample-position transport, BPM, 4/4 и bar quantisation;
- добавить диагностику backend state, reconnect и xrun;
- написать тесты buffer-boundary и sample-position invariants.

**Готово, когда:** transport и quantisation тестируются без JACK/PipeWire.

## Phase 2 — Реальный JACK/PipeWire audio path

**Результат:** приложение слышит и записывает аудио через Ubuntu audio stack.

- реализовать JACK C API adapter;
- открыть аудио-входы/выходы и сообщать sample rate/buffer size;
- подключить MIDI ingress/egress;
- добавить startup diagnostics и reconnect handling;
- проверить путь JACK API → PipeWire с `pw-jack` и системными JACK libraries.

**Готово, когда:** фиксированная тестовая сессия работает на целевом интерфейсе без xruns в оговорённом quantum.

## Phase 3 — Четыре аудиодорожки и looping

**Результат:** первый музыкальный vertical slice.

- реализовать input monitoring, metronome и one-bar count-in;
- записывать первый loop к ближайшим bar boundaries;
- сохранять source take;
- добавлять non-destructive overdub layers;
- реализовать mute/solo, clear layer и stem routing;
- подключить keyboard shortcuts и базовый MIDI Learn.

**Готово, когда:** пользователь может создать четыре дорожки, наложить слои и безопасно удалить отдельный слой.

## Phase 4 — MIDI и встроенные инструменты

**Результат:** MIDI-дорожки воспроизводят встроенный звук.

- реализовать до четырёх MIDI-дорожек;
- добавить sample-offset MIDI scheduler;
- реализовать subtractive synth: oscillator/filter/envelope/LFO;
- реализовать WAV sampler и channel conversion;
- добавить MIDI Learn для транспорта и управления инструментом;
- проверить deterministic render при фиксированном snapshot.

**Готово, когда:** MIDI events корректно воспроизводятся, записываются и экспортируются без внешних plugins.

## Phase 5 — Document Model, timeline и project bundle

**Результат:** проект можно сохранить, перенести и восстановить.

- определить версионируемый manifest;
- хранить source takes, overdub layers, MIDI и presets в self-contained folder;
- реализовать timeline loop instances с copy/move/trim/delete;
- добавить command-based Undo/Redo;
- реализовать autosave, atomic snapshots и crash recovery;
- запретить абсолютные пути в переносимом manifest.

**Готово, когда:** проект копируется на другой путь, открывается с тем же результатом и восстанавливается после принудительного завершения.

## Phase 6 — Offline export

**Результат:** пользователь получает итоговые файлы.

- offline render stereo mix;
- offline render четырёх stems;
- Standard MIDI File export;
- проверка WAV 48 кГц/24-bit и MIDI round-trip;
- export worker не блокирует audio callback.

**Готово, когда:** mix, stems и MIDI воспроизводят тот же deterministic project state.

## Phase 7 — Native UI

**Результат:** рабочее single-window приложение.

- transport bar и session defaults;
- track lanes с audio/MIDI state;
- timeline с базовыми edit operations;
- MIDI editor;
- synth/sampler controls;
- diagnostics panel;
- keyboard shortcuts и MIDI Learn UI;
- offscreen QML tests.

**Готово, когда:** весь основной workflow выполняется без debug CLI и без ручного редактирования project files.

## Phase 8 — AI recommendations

**Результат:** AI помогает, но не управляет сессией.

- local voice feature extractor;
- typed provider-neutral request/response;
- secret-store integration;
- MIDI motif suggestion 1–8 bars;
- synth parameter suggestions based on vocal features;
- preview render from isolated snapshot;
- Apply/Reject/Undo;
- offline/error state without stopping audio.

**Готово, когда:** исходная вокальная запись не отправляется в API, рекомендацию можно прослушать и отменить, а отсутствие сети не ломает базовый workflow.

## Phase 9 — Packaging and acceptance

**Результат:** воспроизводимый Ubuntu MVP release candidate.

- `.deb` with explicit tested runtime dependencies;
- startup backend/latency diagnostics;
- integration matrix for PipeWire/JACK, reconnect, MIDI and full-duplex audio;
- forced-crash recovery test;
- xrun/CPU/memory measurements;
- license and dependency notice review;
- AppImage decision only after `.deb` is stable.

**Готово, когда:** acceptance suite passes on supported Ubuntu profile and known limitations are documented.

## Acceptance checklist

- [ ] Four audio tracks record, loop, overdub, mute/solo and stem-render.
- [ ] Four MIDI tracks drive the built-in synth/sampler.
- [ ] Fixed-bar quantisation preserves source takes.
- [ ] Timeline supports copy/move/trim/delete of loop instances.
- [ ] BPM/tap tempo, 4/4 grid, count-in and metronome work.
- [ ] Keyboard shortcuts and MIDI Learn control transport.
- [ ] Project bundle is self-contained and recoverable.
- [ ] Stereo mix, stems and MIDI export validate.
- [ ] AI suggestions are previewable, confirmable and undoable.
- [ ] AI outage does not stop audio.
- [ ] No source audio is sent to provider; only required features are sent.
- [ ] Fake-backend, offline engine, QML and backend integration tests exist.
- [ ] `.deb` installs and reports actual PipeWire/JACK backend state.

## Dependencies and ordering

- Phase 1 precedes every realtime feature.
- Phase 2 is needed before hardware acceptance, but fake backend keeps engine work parallelizable.
- Phase 3 precedes AI preview because AI must render against stable project state.
- Phase 5 precedes safe AI Apply/Undo and crash recovery.
- Phase 6 precedes release packaging.
- Phase 7 may progress in parallel with Phases 3–6 once snapshot interfaces are stable.
- Phase 8 should not introduce provider logic into AudioEngine.

## Explicit non-goals for the first release

Do not add plugins, scene matrix, full audio editing, full-song generation, local fallback models or native PipeWire graph control while closing the MVP acceptance checklist.
