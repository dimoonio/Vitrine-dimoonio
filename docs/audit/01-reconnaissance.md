# 01. Разведка репозитория (этап A)

Дата исследования: 2026-10-25. Базовый коммит: `7342933` («Update README.md», автор dimoonio). Код не изменялся.

## 1. Состояние Git

| Параметр | Значение | Как установлено |
|---|---|---|
| Рабочая ветка | `claude/sharp-wright-kiah5b`, совпадает с `main` (`7342933`) | `git status`, `git branch -a` |
| Рабочее дерево | чистое | `git status` |
| Удалённый репозиторий | `origin` → `https://github.com/dimoonio/Vitrine-dimoonio` | `git remote -v` |
| Теги на remote | только `Input-Main` (указывает на `7342933`) | `git ls-remote --tags origin` |
| История | 7 коммитов: 6 от Glongq (2026-10-20…23), последний — dimoonio | `git log` |

Коммит `7342933` удалил всё содержимое `README.md` (61 строка), в файле осталась одна пустая строка. Описание проекта для пользователей теперь есть только в `BUILD.md`, `SOURCE.md`, `CHANGELOG.md` и в истории Git (`git show 7342933^:README.md`).

## 2. Состав репозитория

В Git отслеживается 119 файлов (`git ls-files | wc -l`). Хешированный манифест `source-manifest.json` перечисляет 118 из них (все, кроме себя).

| Группа | Файлы | Объём | Роль |
|---|---|---|---|
| Документация | `README.md` (пуст), `BUILD.md`, `SOURCE.md`, `CHANGELOG.md`, `LICENSE` (GPL-3.0) | — | инструкции и происхождение кода |
| Манифест | `source-manifest.json` | 118 записей SHA-256 | контроль целостности публикации |
| Патч upstream | `patches/vitrine-pixie.patch` | 592 строки, 16 файлов Vitrine | изменения к Vitrine `ebc1cbf6…` |
| Исходники Pixie | `NativeDownloads/src/*.m`, `*.c` (25 файлов) | 7 072 строки | библиотека `SpotifyAudioProbe.dylib` |
| Заголовки | `NativeDownloads/src/*.h` (36 файлов) | 842 строки | интерфейсы, сгенерированные константы, встроенный PNG |
| Ассеты | `NativeDownloads/assets/GitHub-Invertocat.{png,svg}` | — | иконка; PNG продублирован массивом в `src/GitHubIcon.h` |
| Сборка/патчеры | `NativeDownloads/tools/*.py`, `tools/key_mode_trampoline.S` | 19 файлов | сборка IPA/DEB, бинарные патчеры, тест-раннеры |
| Тесты/фикстуры | `NativeDownloads/tests/*` | 33 файла | хост-тесты C/ObjC/Python/ASM, UI-фикстуры |

Файлы, затронутые `patches/vitrine-pixie.patch`: `harness/album/{build.sh,main.m}`, `harness/local-files/{README.md,build.sh,cover.m}`, `harness/playlist/{build.sh,main.m}`, `scripts/extract-flags.py`, `tweak/Sources/App/About/{About.h,AboutSettings.m,Backup.m}`, `tweak/Sources/Redesigned/Album/AlbumHeader.x`, `tweak/Sources/Redesigned/Kit/SGRHeaderInfo.{h,m}`, `tweak/Sources/Redesigned/Playlist/PlaylistHeader.x`, `tweak/Sources/Shared/LocalFiles/LocalCover.m`. Патч добавляет в Vitrine функцию `SGWriteDiagnosticsFile` (`patches/vitrine-pixie.patch:327`, `:352`), которую Pixie вызывает через `dlsym` (`NativeDownloads/src/AudioProbeUI.m:307`).

### 2.1. Исходный код, сгенерированные файлы, сборка, тесты

| Категория | Файлы | Примечание |
|---|---|---|
| Рукописный исходный код | все `src/*.m`, `src/*.c`, большинство `src/*.h` | — |
| Сгенерированные заголовки (в репозитории) | `src/NativeRestrictionsPatch.h`, `src/OfflineLimits.h` (оба — `tools/patch_native_restrictions.py`), `src/NativeKeyModePatch.h` (`tools/patch_key_request_mode.py`), `src/NativeTracePatch.h` (`tools/patch_native_trace.py`) | сборка сверяет их с генератором и падает при расхождении (`tools/build_vitrine_9190.py:51-58`). ⛔ Содержимое байтовых массивов не разбирается. |
| Сгенерированный при сборке DEB | `NativeKeyPatch.h` | создаётся `generate_header()` в `tools/build_debs.py:28-47` в каталоге сборки; подключается `src/JailbreakPatches.c:4`. В репозитории его нет — **это ожидаемо, а не потеря** |
| Встроенный ресурс | `src/GitHubIcon.h` (339 строк, массив PNG) | соответствует `assets/GitHub-Invertocat.png`; генератора в репозитории нет |
| Сборка | `tools/build_vitrine_9190.py` (IPA), `tools/build_debs.py` (DEB), `tools/build.py` (устаревший, см. §6) | — |
| Бинарные патчеры | `tools/patch_imports.py`, `patch_native_restrictions.py`, `patch_key_request_mode.py`, `patch_native_trace.py`, `patch_offline.py`, `patch_offline_key_observer.py`, `fix_sideload_dependencies.py`, `key_mode_trampoline.S` | ⛔ для патчеров ограничений и режима ключей — только роль в сборке |
| Тест-раннеры | `tools/test.py`, `tools/test_patchers.py`, `tools/test_native_restrictions.py`, `tools/test_settings_ui.py`, `tools/test_vitrine_settings_ui.py`, `tools/verify_debs.py` | — |

## 3. Языки и технологии

| Язык | Где | Особенности, найденные в коде |
|---|---|---|
| Objective-C (ARC, `-fobjc-arc`) | 20 файлов `src/*.m` | swizzling через `class_replaceMethod`/`class_addMethod`/`imp_implementationWithBlock` (33 места), `objc_msgSend` с ручным приведением типов, associated objects, `__weak`-глобалы, GCD-очереди, `os_unfair_lock`, KVO, `NSURLSession` |
| C11 (`-std=gnu11`) | `src/AudioProbe.c`, `src/SourceGraph.c`, `src/JailbreakPatches.c` | `<stdatomic.h>`, `pthread_once`, `pthread_mutex`, `dlopen/dlsym`, `vm_read_overwrite`, разбор Mach-O |
| Python 3 | `tools/*.py`, `tests/*.py` | разбор и модификация Mach-O/ZIP, `unittest` |
| ARM64 assembly | `tools/key_mode_trampoline.S`, `tests/native_restrictions.S`, `tests/native_trace_abi.S` | шаблоны трамплинов и тестовые бинарники. ⛔ Не разбираются |
| Swift | `tests/SettingsFixture.swift` | UI-фикстура |
| Unified diff | `patches/vitrine-pixie.patch` | изменения Vitrine (Logos `.x`, ObjC) |

## 4. Сборочные конфигурации

### 4.1. IPA (основной путь) — `NativeDownloads/tools/build_vitrine_9190.py`

- Вход: пользовательский IPA Vitrine b14 / Spotify 9.1.90; проверяются `CFBundleShortVersionString == "9.1.90"` (`:62-63`), UUID исполняемого файла `fbce22c1…943e` (`:20`, `:66-67`), наличие `spotifyglass.dylib` (`:68-69`) и встроенного CydiaSubstrate (`:70-71`).
- Опционально `--vitrine-dylib`: проверка подписи, `dyld_info` и маркера `showAlbumShuffle:play:download:add:playColor:` (`:44-49`).
- Компиляция `SpotifyAudioProbe.dylib`: 23 исходника `SOURCE_FILES` (`:19-26`), `-target arm64-apple-ios15.0 -O2 -fobjc-arc -fvisibility=hidden -Wall -Wextra -Werror=implicit-function-declaration`, фреймворки AudioToolbox, Foundation, UIKit, AVFAudio, CoreGraphics, AVFoundation, CoreMedia, Network (`:98-106`).
- Патчи исполняемого файла по порядку: `patch_audio_imports` → `patch_limits` → `patch_key_mode(runtime_selectable=False)` → `patch_native_trace` (`:75-78`). ⛔ Содержание трёх последних не разбирается.
- Проверка: `dyld_info -imports`, все пять импортов должны разрешаться через `SpotifyAudioProbe` (`:112-116`); ZIP `testzip()` (`:124-126`).
- Подпись: ad hoc (`codesign --sign -`) для `spotifyglass.dylib`, companion-библиотеки и исполняемого файла.

### 4.2. DEB (jailbreak) — `NativeDownloads/tools/build_debs.py`

- Те же `SOURCE_FILES` плюс `src/JailbreakPatches.c`, `src/JailbreakSupport.m` и `vitrine-upstream/tweak/Sources/Core/SGRebind.m` (`:104-114`).
- Флаги: `-DSAP_JAILBREAK=1 -DSG_DRIVER=0 -DSGRebindImport=SAPJBRebindImport`, `-target arm64-apple-ios16.1` (минимальная версия выше, чем у IPA — 15.0), фреймворк `CydiaSubstrate` из Theos (`:104-110`).
- Два варианта: rootful (`iphoneos-arm`, префикс пустой) и rootless (`iphoneos-arm64`, `/var/jb`) (`:96`).
- Предварительно запускается `tests/jailbreak_patches.py` на фикстуре (`:92`).
- `BUILD.md` прямо говорит, что опубликованные DEB **не пересобраны** из v1.2.1 и на устройстве с jailbreak **не тестировались**.

### 4.3. Макросы условной компиляции

| Макрос | Где задаётся | Что меняет |
|---|---|---|
| `SAP_JAILBREAK` | `build_debs.py` (`=1`) | `AudioProbe.c:78-98` (перепривязка импортов через `SAPJBRebindImport`), `AudioProbeUI.m:24`, `:159`, `:209`, `:243`, `:328`, `NativeOffline.m:289` |
| `SAP_TESTING` | `tools/test.py` | тестовые хуки (`SAPTestUseBackend`, `SAPTraceTestConfigure` и др.), каталоги в `TMPDIR`, короткие тайм-ауты (`TrackDownload.m:50-60`, `TrackJournal.m:742-746`) |
| `SAP_RUNTIME_TESTING` | нигде в репозитории | `NativeOffline.h:342`, `TrackDownload.m:105-115`, `:937-949`. Сборки с этим флагом в репозитории нет |
| `SAP_PROBE_MEDIA_CACHE` | по умолчанию `0` (`EncodedAudio.m:9-10`) | отключённая ветка исследования кэша (`EncodedAudio.m:18`, `:129`). ⛔ Не разбирается |
| `SAP_UI_FIXTURE` | UI-фикстуры | `DownloadSettingsUI.m:653` |

## 5. Внешние зависимости

| Зависимость | Тип | Как используется | Доступна здесь |
|---|---|---|---|
| Spotify 9.1.90 (UUID `fbce22c1…`) | внешнее приложение | хост процесса; классы `SPT*`, `Esperanto*` через `NSClassFromString` | нет |
| Vitrine 1.0.0-beta.14 (`ebc1cbf6…`) + патч | tweak | `spotifyglass.dylib`; символы `SGWriteDiagnosticsFile`, `SGSpclientHeaders`, `SGPageRow`, `SGActionRow`, `SGWithTile`, `SGSection` через `dlsym`; класс `SGModPage` | нет; кроме `SGWriteDiagnosticsFile`, эти символы в патче не определены — предполагается, что они есть в upstream (не проверено) |
| CydiaSubstrate | библиотека | IPA: встроена в исходный IPA; DEB: `MSHookMemory` (`JailbreakSupport.m:22`) | нет |
| Apple SDK (iOS 15+/16.1+) | системные | AudioToolbox, AVFoundation, CoreMedia, UIKit, Network, CommonCrypto | нет |
| Theos | инструмент | сборка Vitrine и DEB | нет |
| EeveeSpotify | исторический предок | упоминается в `tools/build.py`, `tools/test.py:69-80`, `AudioProbeUI.m:313`, `DownloadSettingsUI.m:180`, `:656-658` | нет |

## 6. Воспроизводимость сборки и отсутствующие компоненты

**Вывод: из содержимого форка сборку воспроизвести нельзя. Здесь она не проверялась.** Нужны macOS, Xcode, iOS SDK и пользовательский IPA — это ожидаемо и описано в `BUILD.md`. Помимо этого, найдены пробелы, которые `BUILD.md` не объясняет:

| Отсутствует / рассогласовано | Где упоминается | Что это ограничивает |
|---|---|---|
| Spotify-бинарник 9.1.90 | везде | все выводы о внутреннем поведении Spotify (смещения, классы, сигнатуры) — только по коду Pixie |
| Исходники Vitrine b14 | `BUILD.md` §1 | нельзя проверить символы `SG*`, вызываемые через `dlsym`, и поведение `SGModPage` |
| Тег `pixie-9.1.90-jailbreak-1` | `BUILD.md` («use tag … rather than current main») | в форке такого тега нет (есть только `Input-Main`). По удалённому тексту `README.md` тег находится в upstream `Glongq/Vitrine-Pixie`. Воспроизвести опубликованные DEB из форка нельзя → [SAP-004](defects.md#sap-004) |
| `provenance.json`, `provenance-ui-v7.json` | `tools/build.py:28`, `tools/test_patchers.py:64-65` | `tools/build.py` неработоспособен; один тест `test_patchers.py` падает → [SAP-001](defects.md#sap-001), [SAP-002](defects.md#sap-002) |
| `analysis/*.md`, `analysis/*.txt`, `build/device-v14/queue_breakpoint.py` | комментарии `TrackJournal.h:5`, `:34`, `TrackJournal.m:517`, `:1231`, `NativeOffline.m:232-233`, `:246`, `:327` | обоснования схемы protobuf и смещений недоступны; проверить их нельзя |
| Фикстура «exact Spotify binary» | `tests/key_mode_runtime.py` | три теста всегда пропускаются (`skipped`) |

## 7. Точки входа

| Точка входа | Где | Когда срабатывает |
|---|---|---|
| `__attribute__((constructor)) SAPLoad` | `NativeDownloads/src/AudioProbeUI.m:324-349` | при загрузке dylib (до `main` Spotify). Последовательность — в [02-architecture-preliminary.md §3](02-architecture-preliminary.md#3-последовательность-инициализации) |
| Пять экспортов AudioToolbox: `AudioUnitSetProperty`, `AudioUnitGetProperty`, `AudioOutputUnitStart`, `AudioOutputUnitStop`, `AudioComponentInstanceDispose` | `src/AudioProbe.c:270`, `:331`, `:346`, `:357`, `:361` | IPA: импорты Spotify переадресованы на dylib (`tools/patch_imports.py:12-16`); DEB: перепривязка через `SAPJBRebindImport` (`AudioProbe.c:82-97`) |
| Публичный C-API (`SAP_EXPORT`) | `SpotifyAudioProbeInitialize`, `SpotifyAudioProbeSetObserver`, `SpotifyAudioProbeReadSnapshot`, `SpotifyAudioProbeMuteForSeconds` (`src/AudioProbe.h:13`, `:48-50`); `SAPDiagnosticsSetEnabled` (`src/Diagnostics.h:8`) | внешние потребители; внутри репозитория `SpotifyAudioProbeSetObserver` не вызывается |
| Экспорты для патченного кода Spotify | `SAPCurrentKeyRequestMode` (`NativeOffline.m:49`), `SAPTraceNativeBoundary` (`NativeKeyTrace.m:132`), `SpotifyAudioProbeObserveOfflineKey` (`EncodedAudio.m:108`) | вызываются из трамплинов в патченном исполняемом файле. ⛔ Механика не разбирается. Для `SpotifyAudioProbeObserveOfflineKey` патчера под 9.1.90 нет → [SAP-008](defects.md#sap-008) |
| ObjC-хуки на классы Spotify/Vitrine | см. [02 §4](02-architecture-preliminary.md#4-хуки-на-чужой-код) | при вызове соответствующих методов |
| UI-действия | `SAPSettingsActions` (`AudioProbeUI.m:153-322`), страницы `DownloadSettingsUI.m` | по нажатию пользователя |
| DEB-бутстрап | `SAPInstallJailbreakSupport` (`JailbreakSupport.m:24`) | из `SAPLoad` при `SAP_JAILBREAK` (`AudioProbeUI.m:328-332`) |

## 8. Проверки, выполненные в Linux

Все команды запускались из `NativeDownloads/`. Вывод сохранён в `NativeDownloads/build/linux-checks/` (каталог в `.gitignore`).

| Проверка | Команда | Результат |
|---|---|---|
| Портативные тесты патчеров | `python3 tools/test_patchers.py -v` | **exit 1**: 11 тестов, 10 `ok`, 1 `ERROR` — `ProvenanceTests.test_v7_runtime_snapshot_and_preserved_v4_backend`: `FileNotFoundError: …/NativeDownloads/provenance.json` → [SAP-001](defects.md#sap-001) |
| Тесты режима ключа | `python3 tests/key_mode_runtime.py -v` | exit 0: `OK (skipped=3)`, «exact Spotify binary fixture absent». Фактически ничего не проверено |
| Политика режима (чистый C) | `gcc -std=gnu11 -Wall -Wextra -Isrc tests/key_mode_policy.c` + запуск | exit 0, без предупреждений; `PASS: only a verified mode-1 selector …` |
| Целостность манифеста | скрипт сверки SHA-256 по `source-manifest.json` | 118 записей; расхождение **только у `README.md`** (после `7342933`); неучтённых файлов нет → [SAP-005](defects.md#sap-005) |
| Компиляция Python-исходников | входит в `test_patchers.py` (`test_python_sources_compile`) | ok |
| ObjC/C с AudioToolbox/Foundation | — | **не проверено**: нет Apple SDK и GNUstep |
| `tools/test.py`, сборка IPA/DEB | — | **не проверено**: нужны `xcrun`, `codesign`, `dyld_info` |

## 9. Логирование и диагностика (обзор)

| Механизм | Файл | Хранилище | Включение |
|---|---|---|---|
| Флаг диагностики | `src/Diagnostics.m` | `NSUserDefaults` ключ `SAPDiagnosticsEnabled` (`:5`) | переключатель в UI; атомарный флаг читается и на аудиопотоке |
| Структурная трасса (JSONL) | `src/DiagnosticTrace.m` | `Documents/PixieTrace{,.1,.2}.jsonl`, до 8 МБ на файл (`:33`, `:54-56`) | при включённой диагностике |
| Журнал загрузок | `src/TrackJournal.m` | `Documents/SpotifyAudioProbeJournal{,.1}.log` (8 МБ, `:153`), `PixieActions{,.1}.log` (2 МБ, `:156`) | при включённой диагностике |
| Отчёты | `TrackJournal.m:1355-1366`, `TrackDownload.m:368-372`, `AudioProbeUI.m:281-321` | `SpotifyAudioProbeTracks.{json,txt}`, `SpotifyAudioProbeDownload.json`, `SpotifyAudioProbe.json`, `PixieSupportLog.json` | по действию пользователя / по завершении загрузки |
| Наблюдение за нативным логом Spotify | `src/NativeErrorLog.m` | в памяти и в журнале | при диагностике; через `SPTLog` |
| Трасса плеера | `src/PlayerTrace.m` | кольцо из 120 событий в памяти (`:24`) | при диагностике |
| Системный лог | `NSLog` в `TrackJournal.m`, `NativeOffline.m` | unified log iOS | при диагностике → [SAP-016](defects.md#sap-016) |

## 10. Инвентарь тестов

Подробный разбор покрытия — в [00-coverage.md](00-coverage.md). Кратко:

- `tools/test.py` собирает и запускает 21 хост-тест (ObjC/C с `-fsanitize=address,undefined`) и два Python-скрипта. Требует macOS.
- **Не подключены ни к одному раннеру**: `tests/album_metadata.m`, `tests/bench_audio.c`, `tests/bench_report.m`, `tests/sideload_dependencies.py`, `tests/localization-catalog.tsv`. `tests/key_mode_runtime.py` нигде не вызывается, а при ручном запуске пропускает все тесты. → [SAP-006](defects.md#sap-006)
- UI-фикстуры (`SettingsFixture.swift`, `SettingsFixtureActions.m`, `VitrineSettingsFixture.m`) запускаются отдельными скриптами `tools/test_settings_ui.py` и `tools/test_vitrine_settings_ui.py`, а не из `test.py`.
- `tests/jailbreak_patches.py` вызывается только из `build_debs.py`.
- Модули без собственных тестов: `AudioProbeUI.m`, `NativeOffline.m`, `EncodedAudio.m`, `SourceGraph.c`, `TracePlatform.m`, `JailbreakSupport.m`. `AlbumMetadata.m` — тест есть, но не подключён.

## 11. TODO, устаревшее, неиспользуемое

В исходниках нет ни одной метки `TODO/FIXME/XXX/HACK` (`grep -rn` по `NativeDownloads`). Устаревшие и неиспользуемые части найдены по ссылкам:

| Элемент | Доказательство | Статус |
|---|---|---|
| `tools/build.py` (9.1.88, EeveeSpotify 7.0.0) | `tools/build.py:25-26`, `:49-50` | устаревший → [SAP-002](defects.md#sap-002) |
| `tools/patch_offline_key_observer.py` (9.1.88) | docstring `:1`; ни один сборщик 9.1.90 его не импортирует | устаревший → [SAP-002](defects.md#sap-002), [SAP-008](defects.md#sap-008) |
| Ветка `SPOTIFY_TEST_IPA` в `tools/test.py:69-80` (читает `EeveeSpotify.dylib`) | — | устаревшая → [SAP-002](defects.md#sap-002) |
| `--fixed-key-mode-zero` | `tools/build_vitrine_9190.py:37`; `args.fixed_key_mode_zero` нигде не читается | флаг игнорируется → [SAP-003](defects.md#sap-003) |
| `SAPFetchAlbumMetadata` | определена `AlbumMetadata.m:53`, вызовов нет | мёртвый код (компилируется в IPA) |
| `SAPJournalReset`, `SAPJournalWriteReport`, `SAPJournalWriteTable` | `TrackJournal.m:1105`, `:1355`, `:1361`; вызовов нет | мёртвый публичный API |
| `+[SAPSettingsActions snapshotForDetails:]` | `AudioProbeUI.m:164`; вызовов нет | мёртвый код |
| `SAPWithObservedAudioKey` | `EncodedAudio.m:142`; вызовов нет | мёртвый код (⛔ модуль ключей) |
| Константа 9.1.88 в `SAPReadSourceGraph` | `SourceGraph.c:47-50` | устаревшая для 9.1.90 → [SAP-007](defects.md#sap-007) |
| `SAPL(russian, english)` всегда возвращает английский | `Localization.m:3-6` | осознанное решение (английский интерфейс); русские строки остались мёртвым грузом в вызовах |

## 12. Неизвестные области

1. Поведение внутренних API Spotify, которые используются через `objc_msgSend` и чтение памяти: потоки вызова колбэков, допустимость вызова транспорта Esperanto из чужих очередей, исключения. Без бинарника и устройства проверить нельзя.
2. Реальная частота вызовов `AudioUnitSetProperty(kAudioUnitProperty_SetRenderCallback)` и `StreamFormat` у Spotify — от неё зависит, исчерпываются ли пулы `AudioProbe.c` ([SAP-017](defects.md#sap-017)).
3. Набор символов `SG*` в Vitrine b14 и их сигнатуры.
4. Поведение под LiveContainer/SideStore (по `CHANGELOG.md` проверялось только там) — в частности, `_dyld_get_image_header(0)` в `JailbreakSupport.m:25` для DEB.
5. Будут ли iOS-версии ниже 16.1 вести себя иначе для DEB (цель `ios16.1`) и для IPA (цель `ios15.0`).

## Проверь сам (необязательно)

```sh
# 1. Манифест совпадает со всеми файлами, кроме README.md?
python3 -I -c "import json,hashlib,pathlib;m=json.load(open('source-manifest.json'));print([k for k,v in m.items() if hashlib.sha256(pathlib.Path(k).read_bytes()).hexdigest()!=v])"
# 2. Тест провенанса падает из-за отсутствующего файла?
python3 NativeDownloads/tools/test_patchers.py ProvenanceTests -v
# 3. Флаг --fixed-key-mode-zero где-нибудь читается?
grep -n "fixed_key_mode_zero" NativeDownloads/tools/*.py
# 4. Кто ссылается на тесты, не подключённые к раннеру?
grep -rn "album_metadata\|sideload_dependencies.py\|bench_audio" NativeDownloads/tools NativeDownloads/tests
```

Вопросы для самопроверки: почему `NativeKeyPatch.h` отсутствует в репозитории, но это не ошибка? Что произойдёт со звуком Spotify, если `dlopen` AudioToolbox в `resolveBackend` вернёт `NULL`?
