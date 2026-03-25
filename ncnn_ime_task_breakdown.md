# NCNN + IME INT8 Backend: Понедельная разбивка задач

**Дата:** 2026-03-25
**Проект:** Вариант 3 из presentation_variants.md — NCNN + IME-ядра для YOLO11n INT8
**Исполнители:**
- **СТ** — Сергей Тюрин (опыт ~2 года, автор Конституции V4, NCNN fork, IME-эксперт)
- **АН** — Алексей Новиков (опыт ~2 недели, быстро учится)

**Целевой KPI:** YOLO11n INT8 ≥35 FPS на 320×320 на BPI-F3 (K1, cluster0, 4 потока)
**Базовые документы:** Constitution V4 (DOCX), Flowmap V4 (XLSX), plan.md
**Оценка:** ~30 рабочих дней на двоих ≈ 8 календарных недель

---

## Обозначения

- **[0.5д]** — полдня (~4 часа)
- **[1д]** — полный день (~8 часов)
- Каждая задача привязана к kernel family (K-xxx) или правилу (R-xxx) из Конституции V4
- Задачи внутри недели можно делать параллельно (СТ и АН работают одновременно)

---

## Неделя 1: Среда и baseline (25–28 марта)

### Цель недели
Оба разработчика имеют рабочее окружение, собирают NCNN fork, запускают baseline бенчмарк YOLO11n FP16 на K1.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 1.1 | Поднять Docker `bf3-ncnn/`: `cp .env.example .env`, `bash build-docker.sh`, `bash run-docker.sh`. Проверить `riscv64-unknown-linux-gnu-gcc --version` | [0.5д] | Docker работает, SpacemiT toolchain доступен | — |
| 1.2 | Получить SSH-доступ к BPI-F3 от СТ, проверить подключение, `cat /proc/cpuinfo` | [0.5д] | SSH работает, видим X60 cores | — |
| 1.3 | Клонировать NCNN fork (`github.com/RISCY-SVT/ncnn`), переключиться на ветку `k1x-int8-triage`, собрать кросс-компиляцией в Docker | [1д] | `ncnn` собирается для rv64, `benchncnn` бинарник готов | — |
| 1.4 | Прочитать IME Spec (pdf, 22 стр.): разделы 1-3, понять vmadot/vmadot1/vmadot2/vmadot3/vmadotn, нарисовать на бумаге схему MAC-блока 4×4×8 | [1д] | Понимание класса A (GEMM) и класса B (sliding-window) инструкций | K-110, K-130 |
| 1.5 | Запустить `benchncnn` на K1: FP16 YOLO11n 640×640 и 320×320, записать результаты | [0.5д] | Baseline FPS зафиксирован | R-MEA-002 |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 1.6 | Code review собственного NCNN fork: что из `k1x-int8-triage` живо и переиспользуемо, что нужно переписать по V4 | [1д] | Список: keep / rewrite / delete | — |
| 1.7 | Настроить B-RVA22-FAST build lane: `-march=rva22u64_v_zfh_zvfh_zicond_zbc -mabi=lp64d -mtune=spacemit-x60`. Проверить что компилятор принимает флаги | [0.5д] | CMake preset / toolchain file с B-RVA22-FAST | R-BLD-002 |
| 1.8 | Создать ветку `v4-ime-backend` от `k1x-int8-triage`, зафиксировать начальную точку | [0.5д] | Чистая ветка для работы по V4 | — |
| 1.9 | Запустить INT8 baseline на K1 (generic fallback), зафиксировать ~20 сек/кадр для сравнения | [0.5д] | INT8 baseline записан | R-MEA-001 |
| 1.10 | Подготовить для АН описание архитектуры NCNN: dispatch path, Layer class, packing system, как добавлять новый kernel | [1д] | Markdown-документ или устная сессия 1 час | — |

### Отчёт за неделю 1
- Docker и SSH работают
- NCNN fork собирается обоими build lanes (B-CMP, B-RVA22-FAST)
- Baseline: FP16 ~1.77 FPS (640), INT8 generic ~0.05 FPS (640)
- Code review завершён, план переиспользования ясен

---

## Неделя 2: Инфраструктура и подготовительные ядра (31 марта — 4 апреля)

### Цель недели
Runtime-инфраструктура (K-000, K-005, K-010), weight prepack (K-020/K-030), benchmark harness.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 2.1 | Изучить архитектуру NCNN по документу от СТ: Layer dispatch, Mat packing, Allocator | [1д] | Понимание как добавить новый kernel в NCNN | — |
| 2.2 | Написать benchmark harness: скрипт для K1, который запускает `benchncnn` с каноническими параметрами `--pin cluster0 --threads 4 --warmup 10 --runs 100`, логирует lane (B-CMP/B-RVA22-FAST/B-IME-FAST) и prefetch policy | [1д] | `bench.sh` + `parse_results.py` | R-MEA-002, R-MEA-003 |
| 2.3 | Изучить INT8 квантование в NCNN: `ncnn2table` (calibration), `ncnn2int8`, scale-only symmetric scheme. Прогнать calibration на 10 COCO изображениях | [1д] | Квантованная INT8 модель YOLO11n готова | R-ACC-001 |
| 2.4 | Написать accuracy-checker: запустить INT8 модель на 100 COCO val2017 изображениях, сравнить mAP с FP32 reference | [1д] | Скрипт `check_accuracy.py`, baseline mAP записан | R-QA-002 |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 2.5 | K-000: cpuset-aware feature discovery — `hwprobe` на целевом cpuset, определение B-CMP / B-RVA22-FAST / B-IME-FAST lane | [1д] | `runtime_probe.cpp`: функция `discover_features(cpuset)` -> lane + feature mask | K-000, R-BLD-004 |
| 2.6 | K-005: 64B-aligned arena allocator — slab classes (M-A ephemeral, M-B near-term, M-C skip buffers), Zic64b alignment law | [1д] | `arena_alloc.cpp`: aligned_alloc wrapper с 64B guarantee | K-005, R-ISA-002 |
| 2.7 | K-010: Input quantization + narrow pack3/pack4 staging — квантование входного RGB в INT8 без frame-wide pack32 | [1д] | `input_quant.cpp`: u8 image -> a_pack3or4_s8 staging | K-010, R-DF-001, R-DF-002 |
| 2.8 | K-020/K-030: Weight prepack — offline transform для 1x1 (K-020) и 3x3 (K-030) весов в packed layout, 64B slab geometry | [1д] | `weight_prepack.cpp`: raw weights -> w1x1_pack / w3x3_pack | K-020, K-030, R-CCH-001 |

### Отчёт за неделю 2
- Runtime инфраструктура: feature probe, allocator, input quant, weight prepack
- Benchmark harness готов, accuracy checker готов
- INT8 модель сквантована и проверена на accuracy

---

## Неделя 3: Главные вычислительные ядра — RVV (7–11 апреля)

### Цель недели
K-100 (1x1 pointwise GEMM) и K-150 (fused epilogue) — два самых важных ядра. Параллельно: layout ops.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 3.1 | K-160: Concat (direct-write) — vector copy из N веток в единый packed destination, 64B-aligned | [1д] | `concat_packn.cpp`: a_packN_s8[] -> a_packN_s8 | K-160, R-DF-003 |
| 3.2 | K-170: Slice/Split — view semantics (zero-copy через offset), narrow copy только если unavoidable | [0.5д] | `split_view.cpp`: a_packN_s8 -> views | K-170, R-DF-003 |
| 3.3 | K-180: Upsample x2 — nearest-neighbor INT8, pack-preserving, vector replicate + store | [0.5д] | `upsample_packn.cpp`: a_packN_s8 -> a_packN_s8 (2x spatial) | K-180, R-KRN-007 |
| 3.4 | K-145: SPPF / MaxPool — sliding-window max, pack-preserving INT8 | [1д] | `sppf_packn.cpp`: a_packN_s8 -> a_packN_s8 | K-145, R-KRN-007 |
| 3.5 | Написать unit-тесты для K-160, K-170, K-180, K-145: random input -> проверка vs наивная reference impl | [1д] | `test_layout_ops.cpp`: все тесты проходят | R-QA-003 |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 3.6 | K-100: 1x1 pointwise INT8 GEMM (RVV) — direct packed GEMM, `vle8` + `vwmul` + `vwadd` accumulate INT32, Zba address math | [1.5д] | `conv1x1_s8_rvv.cpp`: a_packN_s8 × w1x1_pack -> INT32 accum | K-100, R-KRN-001 |
| 3.7 | K-150: Fused epilogue — INT32 accumulator -> bias -> fixed-point requant -> PWL SiLU -> packed INT8 store. Zicond/Zbb scalar tail | [1.5д] | `fused_epilogue.h`: inline, вызывается из всех producer ядер | K-150, R-KRN-005, R-ACC-003 |
| 3.8 | Интеграция K-100 + K-150 в NCNN dispatch: `Convolution_riscv.cpp`, INT8 1x1 path -> наш kernel | [0.5д] | 1x1 conv слои YOLO11 используют K-100 | — |
| 3.9 | Microbench K-100 на K1: замер cycles/op, сравнение с FP16 baseline | [0.5д] | Microbench результат записан | R-MEA-001 |

### Отчёт за неделю 3
- K-100 (1x1 GEMM, RVV) работает и интегрирован
- K-150 (fused epilogue) готов
- Layout ops (K-160, K-170, K-180, K-145) реализованы и протестированы
- Первые microbench результаты на K1

---

## Неделя 4: 3x3 Conv и Depthwise (14–18 апреля)

### Цель недели
K-120 (3x3 IGEMM) и K-140 (depthwise 3x3) — второй и третий по важности compute kernels.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 4.1 | K-050: RVA22 memory assists — обёртки `prefetch_r()`, `prefetch_w()`, guarded `cbo_zero()` с runtime safety gate | [0.5д] | `mem_assists.h`: inline helpers | K-050, R-ISA-006 |
| 4.2 | Интеграция layout ops (K-160, K-170, K-180, K-145) в NCNN dispatch: подключить к соответствующим Layer classes | [1д] | Concat/Split/Upsample/Pooling используют наши ядра | — |
| 4.3 | Layer-by-layer тестирование: запустить YOLO11n с логированием, проверить что каждый слой использует правильный kernel (не fallback) | [1д] | Лог: все слои используют V4 ядра, fallback counter = 0 | R-QA-001 |
| 4.4 | Добавить в benchmark harness layerbench: замер каждого слоя отдельно, выявление hotspots | [0.5д] | `layerbench.sh` + результаты | R-MEA-001 |
| 4.5 | Изучить IGEMM алгоритм (arxiv:1907.02129) — indirect convolution, row-pointer indirection | [1д] | Понимание алгоритма для code review работы СТ | K-120, R-DF-004 |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 4.6 | K-120: Standard 3x3 INT8 IGEMM (RVV) — indirect GEMM с scalar row-pointer indirection, Zba address generation (sh1add/sh2add), unit-stride vector loads. Без full im2col! | [2д] | `conv3x3_igemm_s8_rvv.cpp`: a_packN_s8 × w3x3_pack -> INT32 -> K-150 epilogue | K-120, R-KRN-002, R-DF-004 |
| 4.7 | K-140: Depthwise 3x3 direct spatial (RVV) — для YOLO11 Detect DWConv в classification head | [1д] | `convdw3x3_s8_rvv.cpp`: a_packN_s8 × wdw_pack -> INT32 -> K-150 epilogue | K-140, R-KRN-003 |
| 4.8 | Интеграция K-120, K-140 в NCNN dispatch + K-150 epilogue | [0.5д] | 3x3 и depthwise слои используют V4 ядра | — |
| 4.9 | Microbench K-120, K-140 на K1 | [0.5д] | Microbench результаты | R-MEA-001 |

### Отчёт за неделю 4
- Все основные compute ядра RVV готовы: K-100 (1x1), K-120 (3x3), K-140 (depthwise)
- Layout ops интегрированы в NCNN
- Layer-by-layer тест: 0 fallbacks
- Layerbench показывает hotspots

---

## Неделя 5: Stem, Detect tail, End-to-end RVV (21–25 апреля)

### Цель недели
Замкнуть RVV-path: stem (вход сети) и detect tail (выход), первый end-to-end прогон YOLO11n INT8 полностью на V4 ядрах.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 5.1 | K-146: C2PSA stub — если sensitivity-aware, использовать local FP16 carve-out; иначе INT8 passthrough | [1д] | `c2psa_stub.cpp`: минимальная реализация с local mixed precision | K-146, R-ACC-002, R-KRN-008 |
| 5.2 | End-to-end YOLO11n INT8 прогон (RVV path): full-frame forward, замер FPS | [0.5д] | FPS на 320×320 и 640×640, RVV-only lane | R-MEA-001 |
| 5.3 | Accuracy validation: сравнить mAP INT8 (V4 ядра) vs FP32 reference на 100 COCO images | [0.5д] | mAP delta ≤1% | R-QA-002 |
| 5.4 | Отладка: если есть расхождения accuracy — найти и исправить проблемный слой | [1д] | Все слои корректны | — |
| 5.5 | Обновить benchmark harness: полный отчёт (microbench + layerbench + full-frame), все три уровня | [0.5д] | Трёхуровневый benchmark report | R-MEA-001 |
| 5.6 | Подготовить еженедельный отчёт для начальства: RVV-only результаты, прогресс, план IME | [0.5д] | Отчёт 1 стр. | — |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 5.7 | K-090: Stem-special 3x3 s2 low-C entry (RVV) — pack3/pack4 staging -> первый packN. Специализированное ядро для C=3 (RGB) | [1.5д] | `stem_conv3x3s2_s8_rvv.cpp`: a_pack3or4_s8 -> a_packN_s8 | K-090, R-KRN-006 |
| 5.8 | K-190: Detect tail + late dequant — keep branch convs INT8, dequant только для tiny финальных тензоров перед decode/NMS. Zicond/Zbb scalar helpers | [1д] | `detect_tail.cpp`: a_packN_s8 -> small fp16/fp32 decode tensor | K-190, R-DF-005 |
| 5.9 | Интеграция K-090, K-190 в NCNN, end-to-end RVV path полностью замкнут | [0.5д] | Вся сеть на V4 ядрах, 0 fallbacks | — |
| 5.10 | Профилирование end-to-end: perf stat, кэш-промахи, branch mispredictions | [1д] | Профиль с аннотациями: где основные потери | — |

### Отчёт за неделю 5
- **Milestone: первый end-to-end YOLO11n INT8 на V4 RVV ядрах**
- FPS RVV-only: ожидание ~7 FPS на 320×320 (аналог baseline NCNN RVV)
- Accuracy: mAP delta vs FP32
- Профиль: топ-5 hotspots для IME-ускорения

---

## Неделя 6: IME-ядра — основной спринт (28 апреля — 2 мая)

### Цель недели
Добавить IME-варианты главных compute ядер (K-110, K-130, K-095). Это ключевое ускорение ~5-10x.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 6.1 | K-040: IME-specific weight prepack — перепаковка весов для `vmadot*` microkernels | [1д] | `weight_prepack_ime.cpp`: w_raw_s8 -> wime_pack | K-040, R-BLD-005 |
| 6.2 | Написать unit-тесты для IME-ядер: K-110 vs K-100 reference, K-130 vs K-120 reference — bit-exact comparison | [1д] | `test_ime_kernels.cpp`: IME output == RVV output ± 0 | — |
| 6.3 | Обновить K-000 feature discovery: добавить IME gate (cluster0-only, XSMTVDot detection) | [0.5д] | Feature probe различает RVV / IME lanes | K-000, R-BLD-005 |
| 6.4 | Обновить benchmark harness для B-IME-FAST lane: cpuset pinning cluster0-only | [0.5д] | bench.sh поддерживает `--lane ime` | R-MEA-003 |
| 6.5 | Изучить inline asm syntax для vmadot/vmadot1/vmadot2/vmadot3 — подготовка к помощи с K-095 | [1д] | Написан и запущен hello-world vmadot на K1 | K-095 |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 6.6 | K-110: 1x1 pointwise INT8 GEMM (IME) — `vmadot` replaces inner dot-product primitive, same outer contract as K-100 | [1.5д] | `conv1x1_s8_ime.cpp/.S`: a_packN_s8 × wime_pack -> INT32 -> K-150 | K-110, R-KRN-001 |
| 6.7 | K-130: Standard 3x3 INT8 IGEMM (IME) — `vmadot` + `vmadot1`/`vmadot2` sliding-window, same scalar/Zba outer path as K-120 | [2д] | `conv3x3_igemm_s8_ime.cpp/.S`: direct conv without im2col | K-130, R-KRN-002 |
| 6.8 | Интеграция IME dispatch: если lane == B-IME-FAST и cluster0, использовать K-110/K-130 вместо K-100/K-120 | [0.5д] | Runtime auto-select RVV vs IME | R-BLD-005 |

### Отчёт за неделю 6
- IME ядра K-110 (1x1) и K-130 (3x3) работают
- Корректность подтверждена: IME output == RVV output
- Первые IME microbench результаты

---

## Неделя 7: IME stem + оптимизация + end-to-end IME (5–9 мая)

### Цель недели
Завершить IME-path (K-095), end-to-end YOLO11n INT8 на IME, оптимизация производительности.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 7.1 | K-095: Stem-special 3x3 s2 (IME) — под руководством СТ, vmadot* asm для C=3 entry | [1д] | `stem_conv3x3s2_s8_ime.cpp/.S`: IME stem | K-095 |
| 7.2 | End-to-end YOLO11n INT8 (IME path): full-frame forward, замер FPS | [0.5д] | FPS IME на 320×320 и 640×640 | R-MEA-001 |
| 7.3 | Accuracy validation: INT8 IME vs FP32 на 100 COCO images | [0.5д] | mAP delta ≤1% | R-QA-002 |
| 7.4 | CI setup: GitHub Actions с qemu-user для smoke test (RVV path, без IME — нет QEMU эмуляции) | [1д] | CI pipeline: build + RVV functional test | — |
| 7.5 | Начать писать документацию: README (рус + англ), архитектура, как добавить ядро | [1д] | README.md draft | — |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 7.6 | Performance profiling IME path: perf stat, сравнение с RVV, поиск оставшихся bottlenecks | [1д] | Профиль: где ещё теряем cycles | — |
| 7.7 | Tile-size autotune: перебор Ktile {64,96,128} × OCtile {32,64,96} × Ntile {8,16,32}, выбор оптимальных для K1 | [1д] | Оптимальные tile sizes записаны в конфиг | R-CCH-002 |
| 7.8 | Prefetch tuning: измерить эффект prefetch.r/prefetch.w в K-100/K-110/K-120/K-130, включить/выключить | [0.5д] | Решение: prefetch on/off для каждого ядра | R-CCH-005 |
| 7.9 | Оптимизация K-150 fused epilogue: profiling PWL SiLU, проверка bit-exactness | [0.5д] | Epilogue оптимизирован | R-ACC-003 |
| 7.10 | Code review работы АН (layout ops, tests, benchmark harness) | [0.5д] | CR comments, merge | — |
| 7.11 | Подготовить сравнительную таблицу: RVV vs IME vs SpacemiT closed vs Тюрин C rewrite | [0.5д] | Таблица 4 колонки × 3 метрики | — |

### Отчёт за неделю 7
- **Milestone: end-to-end YOLO11n INT8 на IME ядрах**
- FPS IME: ожидание ~20-35 FPS на 320×320
- Сравнение с SpacemiT (35 FPS) и Тюрин C (41 FPS)
- CI pipeline работает (RVV smoke test)

---

## Неделя 8: Финальная оптимизация, документация, delivery (12–16 мая)

### Цель недели
Достичь ≥35 FPS, документация, однокнопочная демка, финальный отчёт.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 8.1 | Однокнопочный скрипт: `run_yolo_demo.sh` — скачивает модель, квантует (или берёт готовую), запускает инференс на K1 | [1д] | Скрипт работает из чистого клона | K3 (usability) |
| 8.2 | Документация: финализация README, CONTRIBUTING, примеры использования | [1д] | Полная документация | — |
| 8.3 | QA audit: packN support audit — все ncnn операторы в YOLO11n используют packed path, нет тихих fallbacks | [0.5д] | Audit report: 100% packN coverage | R-QA-003 |
| 8.4 | Runtime log: при первом запуске выводить discovered features, cpuset, active lane | [0.5д] | Логирование по R-QA-004 | R-QA-004 |
| 8.5 | Финальный benchmark report: microbench + layerbench + full-frame, все три lanes, сравнительная таблица | [0.5д] | Полный benchmark document | R-MEA-001 |
| 8.6 | Подготовка финального отчёта для начальства | [0.5д] | 2-3 стр. отчёт с графиками | — |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 8.7 | Финальная оптимизация: если <35 FPS — профилировать, тюнить hot loop, пробовать TCM для selected slabs (K1 cluster0 512KB TCM) | [1.5д] | ≥35 FPS на 320×320 | KPI |
| 8.8 | 2-thread auxiliary check: бенчмарк на 320×320 с 2 потоками (cluster0) | [0.5д] | FPS @ 2 threads записан | R-THR-002 |
| 8.9 | Code review всего кода АН, финальный merge | [0.5д] | Всё слито в v4-ime-backend | — |
| 8.10 | Подготовка PR в upstream NCNN (или решение о собственном репозитории) | [0.5д] | PR draft или repo setup | K6 (открытость) |
| 8.11 | Архитектурные заметки для следующей фазы (ORT EP): какие ядра переиспользуемы, что нужно абстрагировать | [0.5д] | `notes_for_ort.md` | — |

### Отчёт за неделю 8 (ФИНАЛЬНЫЙ)
- **KPI: ≥35 FPS YOLO11n INT8 320×320 на K1** (достигнут / не достигнут + план)
- Accuracy: mAP delta vs FP32
- Сравнение с SpacemiT closed binary
- Однокнопочная демка работает
- Код в открытом репозитории
- Готовность к следующей фазе (ORT EP)

---

## Сводная таблица трудозатрат

### По неделям

| Неделя | АН (дни) | СТ (дни) | Итого | Ключевой milestone |
|--------|----------|----------|-------|-------------------|
| 1 | 3.5 | 3.5 | 7 | Среда работает, baseline зафиксирован |
| 2 | 4 | 4 | 8 | Инфраструктура (K-000, K-005, K-010, K-020/030) |
| 3 | 4 | 4 | 8 | K-100 + K-150 + layout ops |
| 4 | 4 | 4 | 8 | K-120 + K-140 + интеграция |
| 5 | 3.5 | 4 | 7.5 | **End-to-end RVV path** |
| 6 | 4 | 4 | 8 | IME ядра K-110, K-130 |
| 7 | 4 | 4 | 8 | **End-to-end IME path** |
| 8 | 4 | 3.5 | 7.5 | **≥35 FPS, delivery** |
| **Итого** | **31** | **31** | **62** | |

### По kernel families

| Kernel | Описание | Автор | Неделя | Дни |
|--------|----------|-------|--------|-----|
| K-000 | Feature discovery | СТ→АН updates | 2, 6 | 1.5 |
| K-005 | Arena allocator | СТ | 2 | 1 |
| K-010 | Input quantization | СТ | 2 | 1 |
| K-020/030 | Weight prepack (RVV) | СТ | 2 | 1 |
| K-040 | Weight prepack (IME) | АН | 6 | 1 |
| K-050 | Memory assists | АН | 4 | 0.5 |
| K-090 | Stem 3x3 s2 (RVV) | СТ | 5 | 1.5 |
| K-095 | Stem 3x3 s2 (IME) | АН (под рук. СТ) | 7 | 1 |
| K-100 | 1x1 GEMM (RVV) | СТ | 3 | 2 |
| K-110 | 1x1 GEMM (IME) | СТ | 6 | 1.5 |
| K-120 | 3x3 IGEMM (RVV) | СТ | 4 | 2 |
| K-130 | 3x3 IGEMM (IME) | СТ | 6 | 2 |
| K-140 | Depthwise 3x3 | СТ | 4 | 1 |
| K-145 | SPPF/MaxPool | АН | 3 | 1 |
| K-146 | C2PSA stub | АН | 5 | 1 |
| K-150 | Fused epilogue | СТ | 3 | 2 |
| K-160 | Concat | АН | 3 | 1 |
| K-170 | Slice/Split | АН | 3 | 0.5 |
| K-180 | Upsample | АН | 3 | 0.5 |
| K-190 | Detect tail | СТ | 5 | 1 |

### Распределение по ролям

**Сергей Тюрин (СТ) — 31 дней:**
- Compute ядра (K-100, K-110, K-120, K-130, K-140, K-090, K-095 review): ~12 дней
- Инфраструктура (K-000, K-005, K-010, K-020/030): ~4 дня
- Fused epilogue (K-150) + Detect tail (K-190): ~3 дня
- Профилирование и оптимизация: ~5 дней
- Code review, архитектура, интеграция: ~4 дня
- Документация, отчёты: ~3 дня

**Алексей Новиков (АН) — 31 дней:**
- Layout ops (K-160, K-170, K-180, K-145, K-146): ~4 дня
- IME helpers (K-040, K-095, feature discovery update): ~3 дня
- Тестирование (unit tests, accuracy, layer-by-layer): ~5 дней
- Benchmark harness + отчёты: ~4 дня
- Изучение (IME spec, NCNN arch, IGEMM paper, INT8 quant): ~5 дней
- Инфраструктура (Docker, SSH, CI, calibration): ~4 дня
- Документация + демка: ~4 дня
- Еженедельные отчёты: ~2 дня

---

## Риски и митигация

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| IME не даёт ожидаемого ускорения из-за memory bottleneck | Средняя | Высокое | Prefetch tuning, tile-size autotune, TCM для hot slabs |
| Compiler не поддерживает rva22u64 profile syntax | Низкая | Низкое | Fallback на явный `-march=rv64gc...` с расширениями |
| SSH к BPI-F3 нестабилен | Средняя | Среднее | QEMU для функциональных тестов, batch-режим бенчмарков |
| NCNN upstream отклоняет PR | Низкая | Низкое | Собственный fork + репозиторий |
| АН не успевает освоить inline asm для IME | Средняя | Низкое | СТ помогает / берёт K-095 на себя |
| PWL SiLU не проходит accuracy gate (±1 quant level) | Низкая | Среднее | Увеличить число сегментов PWL, сравнить с точным SiLU |
| C2PSA требует полноценной mixed-precision реализации | Средняя | Среднее | Начать с FP16 carve-out, оптимизировать позже |

---

## Зависимости между задачами

```
Неделя 1: Среда
  ├── 1.1-1.3 (АН: Docker, SSH, build) ──┐
  ├── 1.4 (АН: IME spec)                 │
  └── 1.6-1.8 (СТ: review, lanes)        │
                                          ▼
Неделя 2: Инфраструктура
  ├── 2.5 K-000 (СТ) ─────────────────────┐
  ├── 2.6 K-005 (СТ) ─────────────────────┤
  ├── 2.7 K-010 (СТ) ─────────────────────┤
  ├── 2.8 K-020/030 (СТ) ─────────────────┤
  └── 2.1-2.4 (АН: learn, bench, quant)   │
                                           ▼
Неделя 3: Core kernels
  ├── 3.6 K-100 (СТ) ◄── K-020 ──────────┐
  ├── 3.7 K-150 (СТ) ◄── K-010            │
  └── 3.1-3.4 (АН: layout ops) ◄── K-005  │
                                           ▼
Неделя 4: 3x3 + DW
  ├── 4.6 K-120 (СТ) ◄── K-030, K-150 ───┐
  ├── 4.7 K-140 (СТ)                      │
  └── 4.1-4.4 (АН: assists, integration)  │
                                           ▼
Неделя 5: End-to-end RVV
  ├── 5.7 K-090 (СТ) ◄── K-010, K-150 ───┐
  ├── 5.8 K-190 (СТ)                      │
  └── 5.1-5.6 (АН: C2PSA, E2E test)      │
                                           ▼
Неделя 6: IME kernels
  ├── 6.6 K-110 (СТ) ◄── K-100, K-040 ───┐
  ├── 6.7 K-130 (СТ) ◄── K-120, K-040    │
  └── 6.1-6.5 (АН: K-040, tests, IME)    │
                                           ▼
Неделя 7: IME E2E + optimization
  ├── 7.1 K-095 (АН) ◄── K-090, IME exp  │
  ├── 7.6-7.9 (СТ: profiling, tuning)    │
  └── 7.2-7.5 (АН: E2E, CI, docs)        │
                                           ▼
Неделя 8: Delivery
  ├── 8.7 Final optimization (СТ)
  └── 8.1-8.6 (АН: demo, docs, report)
```

---

## Источники

- `K1X_NCNN_INT8_FASTEST_BACKEND_CONSTITUTION_REVISED_V4_RVA22.docx` — Constitution V4
- `K1X_NCNN_INT8_BACKEND_FLOWMAP_FINAL_REVISED_V4_RVA22.xlsx` — Flowmap V4 (Nodes, Edges, Kernels, Rules)
- `plan.md` — предыдущий план (v1, одна персона)
- `presentation_variants.md` — анализ 13 вариантов инференса
- IME Spec v20240422 (`docs/spacemit-ime-asciidoc.pdf`)
- IGEMM paper: https://arxiv.org/abs/1907.02129
