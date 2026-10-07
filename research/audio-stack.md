# Исследование аудиостека для DAW MVP

Статус: исследование для архитектурной спецификации, без изменения кода. Репозиторий на момент исследования содержит только `README.md` и не имеет коммитов.

## Краткая рекомендация

Рекомендуемый стек для MVP:

- **C++-ядро аудиодвижка**, не зависящее от GUI и не выполняющее Qt/файловый ввод-вывод из realtime-пути.
- **Qt 6 + Qt Quick/QML + Qt Quick Controls** для одного главного окна, timeline и управляющей поверхности.
- **JACK C API как MVP audio/MIDI backend, работающий поверх PipeWire JACK-совместимости** (`pw-jack`/системная замена `libjack`).
- Явный интерфейс `AudioBackend`, чтобы позднее добавить **нативный PipeWire backend**, если потребуются нативное управление графом, точные PipeWire properties/metadata или ограничения JACK-совместимости.
- **CMake**; для Ubuntu в первую очередь `.deb` с системными Qt/PipeWire/JACK runtime-зависимостями. AppImage можно добавить как дополнительный portable-артефакт после проверки QML-модулей и аудиосессии.
- **Qt Test + CTest** для C++/интеграционных тестов и Qt Quick Test для QML; engine должен тестироваться через fake backend без запуска графического окна.

Это не означает, что JACK-сервер надо запускать отдельно: согласно документации PipeWire, `pw-jack` перенаправляет приложения, использующие JACK client libraries, в PipeWire, а при системной замене JACK-библиотек это происходит автоматически. Таким образом, путь «JACK API → PipeWire» даёт требуемую совместимость с JACK при существенно меньшей сложности MVP, чем непосредственная реализация всех деталей PipeWire native API.

**Ключевой риск рекомендации:** приложение зависит от корректно установленного PipeWire session manager и JACK-совместимого слоя на целевой Ubuntu. Это нужно проверять на реальных конфигурациях, а не считать гарантией одного API.

## Критерии оценки

В оценке учитывались: четыре одновременные аудиодорожки, MIDI, fixed-bar looping, неразрушающие overdub layers, базовый timeline, встроенные subtractive synth и WAV sampler, одно native desktop window, realtime low-latency, отсутствие внешних плагинов в MVP, экспорт WAV/stems/MIDI, лицензирование, Ubuntu build/distribution и тестируемость.

## Сравнение вариантов

| Вариант | Что закрывает хорошо | Что не закрывает сам | Вывод для MVP |
|---|---|---|---|
| **JUCE** | Единый C++ framework для GUI, audio devices, graph, MIDI, WAV и тестового фреймворка; быстрый путь к рабочему audio app | Текущая Linux-документация указывает ALSA и опциональный JACK, но не PipeWire native; коммерческое распространение требует отдельной проверки JUCE-лицензии | Сильная альтернатива при готовности принять AGPL/commercial licensing. Не выбран из-за лицензионного риска и отсутствия необходимости в plugin-host API для MVP |
| **Qt 6/QML + отдельный engine** | Native top-level window, QML canvas/controls, C++ bridge, Qt Test/Qt Quick Test, удобное разделение UI и realtime-ядра | Qt не является полноценным профессиональным audio/MIDI engine; Qt Multimedia не заменяет JACK/PipeWire backend и музыкальный transport | Выбранный UI/application layer; audio должен быть отдельным C++ модулем |
| **RtAudio** | Небольшой C++ callback API, ALSA/JACK на Linux, permissive MIT-подобная лицензия | В официальном перечне backend нет native PipeWire и нет MIDI/transport API; routing/graph остаются приложению | Допустим как простой portability adapter, но не как основа Linux DAW |
| **PortAudio** | Унифицированные devices/streams/callback, ALSA и JACK host APIs, permissive MIT license | Wrapper над native audio API; стандартный API не даёт DAW transport/graph/MIDI; JACK — extension | Уместен для простого I/O, но добавляет отдельный MIDI и снижает контроль над Linux graph |
| **PipeWire native API** | Нативный graph, audio/video/MIDI, stream format/buffer negotiation, low-latency model, MIT source license с указанными исключениями | Сложнее JACK client model: `pw_context`, core/registry, SPA params, stream lifecycle, session/policy и version matrix | Архитектурно перспективный второй backend; можно выбрать primary только если нужен прямой PipeWire graph control уже в MVP |
| **JACK API** | Профессиональный low-latency synchronous graph, realtime process callback, float32 audio, MIDI events, transport/timebase, client interoperability | Это client API; для PipeWire нужен JACK compatibility layer; ограничения realtime строгие | Рекомендуемый MVP audio/MIDI boundary поверх PipeWire JACK implementation |

## Факты из первичных источников

### 1. JUCE

**Факты.** Официальный README описывает JUCE как cross-platform C++ framework для desktop/mobile applications, audio plug-ins и plug-in hosts, с интеграцией через CMake. JUCE публикует API-документацию из комментариев в заголовках и предоставляет готовые CMake examples для GUI/console/audio plugin: [JUCE README](https://github.com/juce-framework/JUCE/blob/master/README.md), [JUCE CMake API](https://github.com/juce-framework/JUCE/blob/master/docs/CMake%20API.md).

`AudioDeviceManager`/`AudioIODeviceCallback` дают device lifecycle и callback на high-priority audio thread; callback должен заполнить все выходные каналы до возврата: [AudioDeviceManager](https://docs.juce.com/master/classjuce_1_1AudioDeviceManager.html), [AudioIODeviceCallback](https://docs.juce.com/master/classjuce_1_1AudioIODeviceCallback.html).

`AudioProcessorGraph` моделирует processors как nodes и умеет соединять audio/MIDI channels. `AudioFormatWriter` пишет samples в формат через конкретный `AudioFormat`, а `ThreadedWriter` выносит flush на background thread: [AudioProcessorGraph](https://docs.juce.com/master/classjuce_1_1AudioProcessorGraph.html), [AudioFormatWriter](https://docs.juce.com/master/classjuce_1_1AudioFormatWriter.html).

В текущем заголовке `juce_audio_devices` документированы `JUCE_ALSA=1` и `JUCE_JACK=0`; Linux dependency guide перечисляет `libasound2-dev` и `libjack-jackd2-dev` (если JACK включён): [audio devices header](https://docs.juce.com/master/juce__audio__devices_8h.html), [Linux Dependencies](https://github.com/juce-framework/JUCE/blob/master/docs/Linux%20Dependencies.md). Эти документы не описывают PipeWire native backend.

**Вывод.** JUCE функционально закрывает большую часть MVP быстрее, чем самостоятельное ядро: callback, graph, MIDI-составляющая graph, WAV writer и GUI primitives уже есть. Но проект не нуждается в VST/LV2/AU host/plug-in слоях в MVP, а PipeWire native не является описанным backend JUCE. JUCE остаётся хорошим запасным вариантом, если скорость прототипирования важнее лицензирования и допустимо использовать JACK/ALSA путь.

### 2. Qt 6/QML и отдельный audio engine

**Факты.** Qt QML предоставляет язык/engine infrastructure, регистрацию C++ types/modules и C++ API: [Qt QML](https://doc.qt.io/qt-6/qtqml-index.html). Qt Quick даёт visual canvas, input, models/views, animation и C++ extension points: [Qt Quick](https://doc.qt.io/qt-6/qtquick-index.html). Qt Quick Controls даёт готовые desktop-like controls, включая `ApplicationWindow`: [Qt Quick Controls](https://doc.qt.io/qt-6/qtquickcontrols-index.html), [ApplicationWindow](https://doc.qt.io/qt-6/qml-qtquick-controls-applicationwindow.html). `ApplicationWindow` является top-level `Window`, поэтому один root window прямо соответствует требованию native single-window GUI.

Qt Multimedia содержит API для playback и recording audio/video, включая `QAudioInput`, `QAudioOutput`, `QAudioSink` и `MediaRecorder`: [Qt Multimedia](https://doc.qt.io/qt-6/qtmultimedia-index.html). Документация не позиционирует этот модуль как JACK/PipeWire-native профессиональный DAW graph, MIDI transport или sample-accurate looping API.

Qt Test предоставляет C++ unit tests, benchmarking, data-driven tests и CMake/CTest integration; Qt Quick Test исполняет QML `TestCase` и поддерживает `-platform offscreen`: [Qt Test](https://doc.qt.io/qt-6/qttest-index.html), [Qt Test Overview](https://doc.qt.io/qt-6/qtest-overview.html), [Qt Quick Test](https://doc.qt.io/qt-6/qtquicktest-index.html).

**Вывод.** Qt лучше всего соответствует требованию отдельного native desktop UI и позволяет не смешивать UI thread с realtime audio thread. `Qt Multimedia` следует оставить для UI-level preview, если он вообще понадобится; основной capture/playback, musical clock, loop renderer, synth, sampler и export должны жить в отдельном C++ engine. UI получает immutable snapshots/модель состояния, а команды (record/overdub/loop/timeline/MIDI learn) передаются в engine через bounded lock-free command queue.

### 3. RtAudio и PortAudio

**Факты RtAudio.** Официальный репозиторий описывает object-oriented common realtime audio I/O API, stream callback, device enumeration, format/channel conversion и Linux backends native ALSA, JACK, PulseAudio и OSS. В тексте перечислены CMake/Meson/autotools варианты сборки; лицензия названа похожей на MIT и находится в отдельном LICENSE: [RtAudio README/repository](https://github.com/thestk/rtaudio), [RtAudio LICENSE](https://raw.githubusercontent.com/thestk/rtaudio/master/LICENSE), [RtAudio header](https://github.com/thestk/rtaudio/blob/master/RtAudio.h).

В заголовке RtAudio есть `UNIX_JACK`, `LINUX_ALSA` и stream flags вроде `RTAUDIO_MINIMIZE_LATENCY`; там же описан callback audio stream. Native PipeWire backend в перечне RtAudio не указан, MIDI/transport API в этом audio-only API не представлены.

**Факты PortAudio.** PortAudio определяет себя как uniform wrapper над platform-specific native audio APIs. Его ключевые абстракции — Host APIs, Devices и Streams; данные идут через asynchronous callback или synchronous read/write. Документация прямо говорит, что общий API ограничен возможностями конкретного native API, а специальные возможности доступны через host-specific extensions: [PortAudio API Overview](https://portaudio.com/docs/v19-doxydocs/api_overview.html). JACK extension предоставляет операции с JACK client name, но не превращает PortAudio в DAW transport/MIDI API: [PortAudio JACK extension](https://portaudio.com/docs/v19-doxydocs/pa__jack_8h.html).

PortAudio license text разрешает use/copy/modify/distribute/sublicense/sell при сохранении copyright и license notice: [PortAudio license](https://raw.githubusercontent.com/PortAudio/portaudio/master/LICENSE.txt).

**Вывод.** Оба проекта полезны как portability abstraction для обычного audio I/O, но для этого MVP они скрывают именно те Linux-specific возможности, которые важны DAW: graph routing, explicit JACK transport/timebase, MIDI event ports и контроль realtime lifecycle. Если брать RtAudio/PortAudio, MIDI придётся подключать отдельным API, а PipeWire native path всё равно не будет закрыт. Поэтому они не должны быть основным backend; максимум — дополнительный backend для будущей portability.

### 4. PipeWire native API

**Факты.** PipeWire описан как graph-based framework, ориентированный на multimedia data, включая audio, video и MIDI. Clients могут говорить с сервером по native protocol через UNIX domain socket, а graph control и node implementation выполняются через PipeWire API: [PipeWire Overview](https://docs.pipewire.org/page_overview.html). Официальный сайт дополнительно заявляет minimal-latency capture/playback, realtime multimedia processing, JACK compatibility и sandboxed application support: [pipewire.org](https://pipewire.org/).

Native `pw_stream` проходит lifecycle create/connect/format negotiation/buffer negotiation/streaming. Потоки автоматически адаптируются к формату, требуемому сервером; `pw_stream_dequeue_buffer()`/`pw_stream_queue_buffer()` используются для обмена буферами, обычно в `process` callback для снижения buffering: [PipeWire Streams](https://docs.pipewire.org/page_streams.html), [pw_stream](https://docs.pipewire.org/group__pw__stream.html), [pw_buffer](https://docs.pipewire.org/structpw__buffer.html).

У `pw_stream_events` есть `process` callback. Документация отмечает, что он обычно вызывается из mainloop, но может вызываться непосредственно realtime data thread, если клиент готов соблюдать realtime-ограничения: [pw_stream_events](https://docs.pipewire.org/structpw__stream__events.html). Latency model умеет распространять latency objects по linked graph и отдельно описывает дополнительную quantum latency для async links: [PipeWire latency support](https://docs.pipewire.org/page_latency.html).

`pw-jack` подменяет поиск JACK client libraries через `LD_LIBRARY_PATH`, направляя JACK clients в PipeWire; если PipeWire JACK libraries установлены системной заменой, отдельный `pw-jack` не нужен: [pw-jack](https://docs.pipewire.org/devel/page_man_pw-jack_1.html). `jack.conf` документирует `node.latency`, `node.force-quantum`, `node.lock-quantum` и MIDI-related JACK properties: [PipeWire jack.conf](https://docs.pipewire.org/page_man_pipewire-jack_conf_5.html).

Исходный LICENSE PipeWire указывает MIT для source files, с исключением LGPL для `libspa-alsa.so` и GPL2 для `libjackserver.so`, связывающегося с GPL2 jack/control.h: [PipeWire LICENSE](https://raw.githubusercontent.com/PipeWire/pipewire/master/LICENSE), [PipeWire README](https://raw.githubusercontent.com/PipeWire/pipewire/master/README.md).

**Вывод.** Native PipeWire даёт наиболее прямое соответствие системному audio graph и перспективен для production backend. Однако для MVP он требует больше низкоуровневого кода и тестов: context/core/registry, SPA format PODs, buffer negotiation, stream state, session policy, reconnect и version matrix. Требование JACK compatibility не требует этого сразу: JACK API поверх PipeWire уже даёт realtime callback/MIDI/transport и inter-application compatibility. Native backend следует оставить за стабильным `AudioBackend` interface и добавить после измерений.

### 5. JACK API

**Факты.** JACK описывает себя как low-latency audio server с synchronous execution всех clients и low-latency focus. Модель — client, ports, process callback; внутри JACK audio представлен как 32-bit floating point: [JACK API introduction](https://jackaudio.org/api/index.html). 

Process callback должен быть realtime-safe: официальная документация запрещает в нём операции, способные блокировать, включая I/O, `malloc/free`, `printf`, mutex lock, sleep, wait, poll/select и thread join: [JACK client callbacks](https://jackaudio.org/api/group__ClientCallbacks.html), [JACK non-callback API](https://jackaudio.org/api/group__NonCallbackAPI.html).

JACK предоставляет MIDI event ports с sample offsets, event count, reserve/write/read и lost-event count: [JACK MIDI API](https://jackaudio.org/api/group__MIDIAPI.html). Он также предоставляет transport/timebase callbacks, frame position, BBT и loop-related transport bits: [JACK transport/timebase](https://jackaudio.org/api/group__TransportControl.html).

JACK server распространяется под GPL, а client library — под LGPL, что позволяет proprietary programs link с client library: [JACK API license](https://jackaudio.org/api/index.html), [JACK headers](https://github.com/jackaudio/headers/blob/master/jack.h).

**Вывод.** Для MVP JACK client API — лучший баланс зрелости, low-latency semantics, MIDI и PipeWire interoperability. Внутренний musical transport всё равно должен быть источником истины для fixed-bar loop: JACK transport можно синхронизировать/экспортировать, но не следует делать проектное состояние зависимым от внешнего transport master.

## Предлагаемая граница компонентов

```text
Qt 6 / QML / Quick Controls
        |  commands + immutable state snapshots
Application / document / undo-redo / timeline model
        |
Pure C++ AudioEngine
  - sample-position transport + BPM/bar quantisation
  - 4 track mixers
  - loop buffers and non-destructive overdub layers
  - MIDI scheduler and MIDI Learn mapping
  - subtractive synth
  - WAV sampler
  - realtime-safe command/event queues
        |
AudioBackend interface
        |
JACK C API backend  ---> PipeWire JACK libraries / pw-jack ---> PipeWire graph
        |
(optional later) PipeWire native backend
```

### Realtime rules

- Audio callback делает только bounded work: чтение заранее подготовленных immutable snapshots, synth/sampler DSP, track mix, loop wrap и запись в preallocated buffers.
- Запись WAV, flush, stems, project save/autosave, waveform analysis, undo history и экспорт MIDI выполняются вне callback.
- Межпоточные изменения — через заранее выделенные SPSC/MPSC queues или атомарную публикацию snapshot; UI никогда не держит mutex, который нужен audio thread.
- Любой buffer size/sample-rate change от backend проходит через явный reconfigure barrier; после него engine пересоздаёт preallocated state.
- Ошибки PipeWire/JACK, reconnect и xruns переводятся в observable engine state и не вызывают allocation/logging из callback.

### Покрытие функциональных требований

- **4 audio tracks:** внутренний graph фиксирует максимум четыре track processors; каждая track имеет input/loop playback/overdub layer/mute/solo и stem output.
- **Fixed-bar looping:** transport хранит sample frame position и BPM; старт/stop/record/overdub quantize к следующей bar boundary при фиксированном 4/4 MVP. Первая запись не меняет global BPM.
- **Overdub:** новый слой добавляется неразрушающе, исходный layer остаётся отключаемым/удаляемым; callback смешивает активные layers без обращения к диску.
- **MIDI:** внешний MIDI input/output — JACK MIDI ports; внутренние events — sample-offset events, чтобы ноты synth/sampler были sample-scheduled. MIDI export — отдельный non-RT Standard MIDI File writer.
- **Synth/sampler:** оба инструмента — engine processors; внешний plugin ABI не включается в MVP.
- **Timeline:** UI-модель отображает clip/layer metadata; copy/move/trim/delete меняют документ и публикуют новый render snapshot, а не mutate-ят объект, который читает callback.
- **WAV/stems:** mix и каждый track/stem рендерятся offline из того же deterministic engine state; файл пишется non-RT в WAV 48 kHz/24-bit согласно границам README.

## Лицензирование и compliance

1. **Наиболее низкий риск:** Qt modules, которые реально используются, под LGPLv3/GPL terms соответствующего модуля; RtAudio и PortAudio под permissive MIT-подобными лицензиями; PipeWire source в основном MIT; JACK client library LGPL. Нужно поставлять notices и не смешивать license status отдельных PipeWire components.
2. **Qt:** официальная licensing page прямо говорит, что open-source Qt доступен под LGPLv3 или GPLv3, но некоторые модули доступны только под GPLv3. Runtime QML/Quick/Controls страницы указывают LGPLv3 или GPLv2 для соответствующих модулей. Нужно pin-ить ровно используемые Qt modules/version, выполнить LGPL obligations для динамических библиотек и проверить, не включён ли GPL-only module (например, QML compiler) в production package: [Qt Licensing](https://doc.qt.io/qt-6/licensing.html), [Qt QML](https://doc.qt.io/qt-6/qtqml-index.html), [Qt Quick](https://doc.qt.io/qt-6/qtquick-index.html), [Qt Quick Controls](https://doc.qt.io/qt-6/qtquickcontrols-index.html).
3. **JUCE:** текущий JUCE README и LICENSE описывают dual licensing под AGPLv3 и commercial JUCE licence. Официальная JUCE 9 EULA имеет revenue/funding limits, licence types и отдельные обязанности для distribution; JUCE 6/7/8 имеют свои EULA. Для закрытого или коммерческого DAW нельзя выводить лицензионную пригодность из старых «JUCE is ISC» утверждений: нужен review точной версии и EULA: [JUCE LICENSE](https://github.com/juce-framework/JUCE/blob/master/LICENSE.md), [JUCE 9 EULA](https://juce.com/legal/juce-9-licence/), [JUCE licensing FAQ](https://juce.com/get-juce/).
4. **JACK/PipeWire:** linking к JACK client library отличается от использования GPL JACK server; приложение не должно встраивать GPL server. PipeWire LICENSE отдельно выделяет `libjackserver.so` и `libspa-alsa.so`, поэтому package SBOM/notice должен учитывать фактически linked/runtime-loaded components.
5. **Аудиоданные:** лицензии фреймворков не покрывают права на bundled samples, soundfonts, demo WAV/MIDI и trademarks. MVP должен либо не поставлять copyrighted content, либо иметь отдельный provenance/license manifest.

Это технический risk assessment, а не юридическое заключение.

## Ubuntu build и distribution

**Факты.** JUCE CMake API требует CMake 3.22+ и описывает `add_subdirectory`/`find_package`; его Ubuntu dependency guide перечисляет compiler, ALSA/JACK dev package и графические зависимости: [JUCE CMake API](https://github.com/juce-framework/JUCE/blob/master/docs/CMake%20API.md), [JUCE Linux Dependencies](https://github.com/juce-framework/JUCE/blob/master/docs/Linux%20Dependencies.md). Qt modules документируют CMake `find_package(Qt6 REQUIRED COMPONENTS ...)` и `target_link_libraries`: [Qt Quick CMake](https://doc.qt.io/qt-6/qtquick-index.html), [Qt QML CMake](https://doc.qt.io/qt-6/qtqml-index.html).

Ubuntu packaging documentation требует `debian/` metadata и описывает сборку `.deb` через `dpkg-buildpackage`; binary `.deb` — стандартный формат `dpkg`/APT: [Ubuntu: create a package](https://ubuntu.com/project/docs/contributors/new-package/create-a-new-package/), [Ubuntu: package binaries as .deb](https://documentation.ubuntu.com/hardware-support/image-cookbook/howto/packaging/package/). Ubuntu desktop documentation сравнивает snap/deb и указывает, что deb привязан к Ubuntu release, а зависимости обычно берутся из системы: [Ubuntu snap/deb comparison](https://ubuntu.com/desktop/docs/en/26.04/explanation/snap-and-deb-packages/).

**Рекомендация.**

- Pin compiler standard, CMake, Qt minor, PipeWire/JACK headers и exact CI image; не собирать приложение против случайных `master` headers.
- Build targets разделить на `daw_engine`, `daw_backend_jack`, `daw_ui`, `daw_app`, `daw_tests`; test target не должен требовать audio device.
- Для первого Ubuntu distribution выпускать `.deb`, который использует системный PipeWire/WirePlumber/JACK compatibility runtime, а не встраивает audio daemon в приложение. В `debian/control` явно указывать только проверенные release-specific runtime dependencies.
- AppImage рассматривать как дополнительный portable build: он уменьшает зависимость от версии Qt, но усложняет bundling QML imports, platform plugins и диагностику PipeWire/JACK session. Это решение нужно принимать после работающего `.deb` и integration matrix.
- Не обещать low-latency только фактом установки пакета: при запуске проверять доступность PipeWire graph, JACK compatibility, sample rate/quantum, MIDI ports и сообщать пользователю фактический backend/latency.

## Тестируемость и acceptance checks

### Offline C++ tests

Использовать Qt Test/CTest либо другой отдельный C++ test runner для engine без GUI/backend. Минимальный набор:

- bar quantisation при разных BPM и buffer sizes, включая переход через buffer boundary;
- запись первого loop, wrap ровно на bar boundary, overdub layer add/disable/delete;
- четырёхтрековый mix, mute/solo и stem isolation;
- sample-offset ordering MIDI events, note-on/off, sustain и MIDI export/import round-trip;
- deterministic subtractive synth при фиксированном seed/parameter snapshot;
- WAV sampler interpolation, channel conversion и end-of-file behavior;
- WAV 48 kHz/24-bit header/sample validation, mix/stem equality;
- undo/redo и crash-recovery document snapshots;
- отсутствие allocations/locks/file calls в тестируемом realtime render function (instrumented allocator/TSAN-oriented test).

Qt официально поддерживает data-driven tests, benchmarking, CMake/CTest и QML offscreen tests: [Qt Test Overview](https://doc.qt.io/qt-6/qtest-overview.html), [Qt Quick Test](https://doc.qt.io/qt-6/qtquicktest-index.html). Если выбран JUCE, его `UnitTest`/`UnitTestRunner` дают аналогичный framework с reproducible random seed: [JUCE UnitTest](https://docs.juce.com/master/classjuce_1_1UnitTest.html), [JUCE UnitTestRunner](https://docs.juce.com/master/classjuce_1_1UnitTestRunner.html).

### Backend integration tests

В CI и на поддерживаемом Ubuntu hardware profile проверять:

1. PipeWire running, JACK client library resolves to PipeWire (`pw-jack` path и system-replacement path).
2. Open/close/reconnect, sample-rate and quantum changes.
3. Four-track full-duplex capture/playback at target 48 kHz and selected quantum.
4. External JACK audio client connection and JACK MIDI input/output.
5. no-xrun run of a fixed-duration loop session; отдельно собирать callback deadline misses, xruns, CPU and memory.
6. export while playback/recording is active; export must not block callback.
7. headless/offscreen UI tests and project reopen after forced process termination.

PipeWire README documents `PIPEWIRE_LATENCY`, `PIPEWIRE_QUANTUM`, JACK redirection via `pw-jack` and `pw-top`/`pw-dump` diagnostics; эти параметры стоит использовать в reproducible integration harness, но acceptance thresholds (например, допустимый quantum/xrun rate) должны быть измерены на целевых аудиоинтерфейсах: [PipeWire README](https://raw.githubusercontent.com/PipeWire/pipewire/master/README.md).

## Итоговое решение

Для данного MVP не стоит начинать с JUCE, если проект хочет сохранить максимально предсказуемую permissive/LGPL-compatible лицензионную поверхность и уже зафиксировал отсутствие внешних plugins. Рекомендуется **Qt 6/QML UI + отдельное pure-C++ engine + JACK C API backend поверх PipeWire**. Это покрывает native single-window GUI, low-latency callback, MIDI, JACK interoperability и PipeWire deployment без написания полного PipeWire native integration слоя в первой итерации. `AudioBackend` должен сразу быть абстракцией: native PipeWire backend остаётся технически возможным, а RtAudio/PortAudio — только optional portability adapters, не основой timeline/transport.

## Сводка первичных источников

- JUCE README: https://github.com/juce-framework/JUCE/blob/master/README.md
- JUCE LICENSE: https://github.com/juce-framework/JUCE/blob/master/LICENSE.md
- JUCE 9 EULA: https://juce.com/legal/juce-9-licence/
- JUCE CMake API: https://github.com/juce-framework/JUCE/blob/master/docs/CMake%20API.md
- JUCE Linux dependencies: https://github.com/juce-framework/JUCE/blob/master/docs/Linux%20Dependencies.md
- JUCE audio callback: https://docs.juce.com/master/classjuce_1_1AudioIODeviceCallback.html
- JUCE graph: https://docs.juce.com/master/classjuce_1_1AudioProcessorGraph.html
- JUCE WAV writer: https://docs.juce.com/master/classjuce_1_1AudioFormatWriter.html
- Qt Licensing: https://doc.qt.io/qt-6/licensing.html
- Qt QML: https://doc.qt.io/qt-6/qtqml-index.html
- Qt Quick: https://doc.qt.io/qt-6/qtquick-index.html
- Qt Quick Controls/ApplicationWindow: https://doc.qt.io/qt-6/qml-qtquick-controls-applicationwindow.html
- Qt Multimedia: https://doc.qt.io/qt-6/qtmultimedia-index.html
- Qt Test: https://doc.qt.io/qt-6/qttest-index.html
- Qt Quick Test: https://doc.qt.io/qt-6/qtquicktest-index.html
- RtAudio repository/license: https://github.com/thestk/rtaudio ; https://raw.githubusercontent.com/thestk/rtaudio/master/LICENSE
- PortAudio API: https://portaudio.com/docs/v19-doxydocs/api_overview.html
- PortAudio JACK extension: https://portaudio.com/docs/v19-doxydocs/pa__jack_8h.html
- PortAudio license: https://raw.githubusercontent.com/PortAudio/portaudio/master/LICENSE.txt
- PipeWire overview: https://docs.pipewire.org/page_overview.html
- PipeWire streams: https://docs.pipewire.org/page_streams.html
- PipeWire stream API: https://docs.pipewire.org/group__pw__stream.html
- PipeWire stream events: https://docs.pipewire.org/structpw__stream__events.html
- PipeWire latency: https://docs.pipewire.org/page_latency.html
- PipeWire `pw-jack`: https://docs.pipewire.org/devel/page_man_pw-jack_1.html
- PipeWire JACK config: https://docs.pipewire.org/page_man_pipewire-jack_conf_5.html
- PipeWire LICENSE/README: https://raw.githubusercontent.com/PipeWire/pipewire/master/LICENSE ; https://raw.githubusercontent.com/PipeWire/pipewire/master/README.md
- JACK API: https://jackaudio.org/api/index.html
- JACK callbacks: https://jackaudio.org/api/group__ClientCallbacks.html
- JACK MIDI: https://jackaudio.org/api/group__MIDIAPI.html
- JACK transport: https://jackaudio.org/api/group__TransportControl.html
- JACK headers/license: https://github.com/jackaudio/headers/blob/master/jack.h
- Ubuntu `.deb` packaging: https://ubuntu.com/project/docs/contributors/new-package/create-a-new-package/
- Ubuntu binary `.deb`: https://documentation.ubuntu.com/hardware-support/image-cookbook/howto/packaging/package/
