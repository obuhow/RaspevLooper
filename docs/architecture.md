# DAW MVP — архитектура

Статус: согласовано для реализации

## Архитектурные цели

1. Сохранить realtime-аудио работоспособным без сети и без GUI-зависимостей.
2. Спрятать сложность transport, looping, DSP, MIDI и export за глубоким интерфейсом AudioEngine.
3. Сохранить заменяемость аудиобэкенда через один реальный seam — AudioBackend.
4. Сделать offline engine тестируемым без PipeWire, JACK и физического устройства.
5. Не смешивать проектное состояние, UI-модель и callback state.

## Высокоуровневая схема

```text
┌──────────────────────────────────────────────┐
│ Qt 6 / QML / Quick Controls                  │
│ one ApplicationWindow                         │
└───────────────────┬──────────────────────────┘
                    │ commands + immutable snapshots
┌───────────────────▼──────────────────────────┐
│ Application / Document Model                 │
│ timeline, project bundle, undo/redo, AI flow │
└───────────────────┬──────────────────────────┘
                    │ bounded command/event queues
┌───────────────────▼──────────────────────────┐
│ Pure C++ AudioEngine                         │
│ transport · tracks · loops · overdub layers  │
│ MIDI scheduler · synth · sampler · mixer    │
└───────────────────┬──────────────────────────┘
                    │ small AudioBackend interface
┌───────────────────▼──────────────────────────┐
│ JACK C API adapter                           │
└───────────────────┬──────────────────────────┘
                    │ JACK libraries / pw-jack
┌───────────────────▼──────────────────────────┐
│ PipeWire graph / WirePlumber                 │
└──────────────────────────────────────────────┘
```

Исследование обоснования стека: [`../research/audio-stack.md`](../research/audio-stack.md).

## Modules and seams

### AudioEngine

**Interface:** принимает музыкальные команды и выдаёт snapshot состояния; принимает/публикует realtime audio/MIDI buffers через AudioBackend.

**Responsibilities:**

- sample-position transport и BPM/bar quantisation;
- состояние play/record/overdub;
- четыре audio tracks и четыре MIDI tracks;
- loop buffers, source take и overdub layers;
- mixer, mute/solo и stem routing;
- MIDI scheduling по sample offset;
- subtractive synth и WAV sampler;
- realtime-safe render path;
- observable состояния xruns, reconnect и ошибки backend.

**Invariant:** callback никогда не выполняет file I/O, allocation, блокирующий lock, UI-вызов или сетевой запрос.

### AudioBackend

Первый и единственный внешний realtime seam.

**Минимальная ответственность interface:**

- открыть/закрыть аудиосессию;
- сообщить sample rate, buffer size, входы и выходы;
- передавать audio buffers в callback;
- передавать MIDI events с sample offsets;
- уведомлять о reconfigure, reconnect и ошибках;
- предоставить диагностику фактического backend и latency.

**Adapter:** JACK C API, работающий через PipeWire JACK libraries или `pw-jack`.

**Future adapter:** native PipeWire, если integration tests покажут необходимость прямого graph control, metadata или свойств PipeWire.

### Document Model

Хранит не realtime-состояние проекта:

- Project settings: BPM, 4/4, grid, sample format;
- Track metadata;
- audio source takes and layer references;
- MIDI events and instrument parameters;
- timeline loop instances;
- AI suggestion records and applied/rejected state;
- undo/redo history metadata;
- recovery snapshot metadata.

Document Model публикует immutable render snapshots для AudioEngine и не мутирует объект, который читает callback.

### Timeline

Timeline работает с экземплярами лупов, а не с raw audio buffers. Copy/move/trim/delete создают новую версию документа и новый render snapshot. Scene matrix отсутствует.

### Realtime render path

В callback разрешены только:

- чтение immutable state;
- bounded iteration по четырём audio tracks, четырём MIDI tracks и активным layers;
- DSP synth/sampler;
- mix, mute/solo, loop wrap;
- запись в заранее выделенные ring/preallocated buffers;
- постановка bounded events для non-realtime workers.

Запрещены:

- malloc/free, исключения, mutex, condition wait;
- printf/logging, filesystem, SQLite/JSON serialization;
- network, Qt object calls, QML signals;
- неизвестная по времени декомпрессия или waveform analysis.

### Thread model

- **UI thread:** QML, controls, user commands, snapshots.
- **Audio thread:** JACK callback и deterministic render.
- **Worker threads:** WAV writer, offline export, waveform analysis, autosave, crash recovery, AI HTTP requests.
- **MIDI ingress:** backend доставляет events в audio callback; mapping commands приходят через bounded queue.

Обмен UI → engine выполняется через bounded command queue. Обмен engine → UI выполняется через атомарно публикуемые immutable snapshots и bounded event queue.

## Musical timing

- Внутренний transport хранит абсолютную sample frame position.
- BPM и 4/4 определяют длину такта в sample frames.
- Start/stop/record/overdub requests получают ближайшую допустимую bar boundary.
- Первый loop не меняет BPM.
- Сетка поддерживает 1/4, 1/8, 1/16 и триоли.
- JACK transport можно синхронизировать позднее, но состояние проекта не зависит от внешнего transport master.

## Audio tracks and overdub

Каждая audio track состоит из:

```text
Track
├── source take(s)
├── base loop reference
├── overdub layers[]
├── gain/pan/mute/solo
└── stem output
```

Новый overdub добавляется как отдельный слой. Удаление слоя — изменение Document Model, не destructive rewrite исходного WAV. Callback читает уже подготовленный слой state; audio writer сохраняет новый материал вне callback.

## MIDI and instruments

MIDI events нормализуются к внутреннему sample timeline. Внешний JACK MIDI и внутренние generated events объединяются в deterministic scheduler.

Subtractive synth MVP:

- oscillator;
- filter;
- envelope;
- LFO;
- preset/parameter snapshot.

WAV sampler MVP:

- loaded WAV asset;
- channel conversion;
- interpolation;
- note-on/note-off and release;
- no external plugin ABI.

## AI seam

AI provider — отдельный non-realtime adapter, не часть AudioEngine.

Поток:

1. UI выбирает vocal track и instrument track.
2. Local analyzer извлекает необходимые признаки: например, pitch range, spectral/timbre descriptors и rhythm/tempo-related features.
3. Provider adapter отправляет только feature payload и музыкальный context.
4. Ответ преобразуется в typed Suggestion.
5. UI показывает preview; preview render выполняется в изолированном snapshot.
6. Apply создаёт undoable document command.
7. Reject ничего не меняет.
8. Недоступность provider переводится в UI error state; audio session не останавливается.

API key читается из системного secret store. Project bundle содержит endpoint/provider settings без secret value.

## Persistence and export

Project bundle должен быть переносимой папкой с относительными ссылками:

```text
project/
├── project manifest
├── audio/
│   ├── source takes
│   └── overdub layers
├── midi/
├── presets/
├── recovery/
└── exports/
```

Формат manifest должен быть версионируемым. Save и autosave пишут временный snapshot и атомарно заменяют предыдущую версию. Recovery выбирает последний валидный snapshot.

Offline export создаёт stereo mix, stems и Standard MIDI File из deterministic engine state. Экспорт не блокирует audio callback.

## UI composition

Одно `ApplicationWindow` содержит:

- transport bar: BPM, tap tempo, play/stop, record, count-in;
- track lanes: audio/MIDI, mute/solo, input, loop state;
- timeline: loop instances и базовые операции copy/move/trim/delete;
- instrument panel: synth/sampler parameters;
- MIDI editor: grid и note editing;
- AI panel: track selection, Suggest, preview, Apply, Reject, Undo;
- diagnostics: backend, sample rate, buffer size, latency, xrun/reconnect status.

UI не рисует realtime buffers напрямую. Waveform/timeline preview получает worker-generated snapshots.

## Build and distribution

- CMake targets: `daw_engine`, `daw_backend_jack`, `daw_ui`, `daw_app`, `daw_tests`.
- Qt 6 QML/Quick/Controls; exact minor version pinning.
- First distribution: `.deb` with tested system dependencies.
- AppImage — после стабильного `.deb` и проверки QML imports.
- CI must run offline engine tests without audio device; hardware matrix runs backend integration separately.
- License notices and Qt/PipeWire/JACK dependency review are release gates.

## Risks and mitigations

- **JACK compatibility differs across Ubuntu setups:** startup diagnostics, fake backend tests, hardware integration matrix.
- **Realtime xrun under load:** preallocation, bounded work, no blocking, measured quantum and CPU budget.
- **Qt licensing/module drift:** pin versions and record exact modules in dependency manifest.
- **Project corruption:** atomic snapshots, schema version, recovery test after forced termination.
- **AI provider latency/failure:** worker adapter, preview snapshot, no audio dependency.
- **Overly broad timeline scope:** only loop instances and basic edit operations in MVP.
