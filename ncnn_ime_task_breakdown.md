# NCNN + IME INT8 Backend: Понедельная разбивка задач  
  
**Дата:** 2026-03-26 (v2)  
**Проект:** Вариант 3 из [presentation_variants.md](presentation_variants.md) — NCNN + IME-ядра для YOLO11n INT8  
**Исполнители:**  
- **СТ** — Сергей Тюрин (опыт ~2 года, автор Конституции V4, NCNN fork, IME-эксперт) 
- **АН** — Алексей Новиков (опыт ~2 недели, быстро учится) 
 
**Целевой KPI:** YOLO11n INT8 ≥ 5 FPS на 640×640 (соответствует ≥35 FPS на 320×320) на BPI-F3 (K1, cluster0, 4 потока)  
**Stretch goal:** ≥ 8 FPS на 640×640 (сопоставимо с SpacemiT ORT EP C++ harness: 11.8 FPS)  
**Базовые документы:** Constitution V4 (DOCX), Flowmap V4 (XLSX), Consensus Plan (specs/)  
**Оценка:** 10 календарных недель (30 марта — 6 июня), ~40 рабочих дней на человека, ~80 person-days итого  
 
---

## KPI: Еженедельный замер FPS
### Зачем

Главный показатель прогресса, замеряемый **каждую пятницу**. Руководство видит прогресс по числу **FPS на 640×640 INT8**, а так же по количеству реализованных ядер, выполнению других задач согласно плана.  

### Каноническая команда замера

```bash
# === CANONICAL FPS BENCHMARK (640x640) ===
# Запускается на BPI-F3 (K1), всегда одинаково.
# Изменяется только бинарник yolo11 (по мере обновления ядер).

ssh svt@banana 'bash -s' <<'REMOTE'
  # Чистая среда
  unset OMP_WAIT_POLICY OMP_NUM_THREADS OMP_PROC_BIND GOMP_CPU_AFFINITY
  export LD_LIBRARY_PATH=/home/svt/opencv-install-k1x-gtk3/lib:${LD_LIBRARY_PATH:-}

  # Параметры ФИКСИРОВАННЫЕ, менять НЕЛЬЗЯ:
  BIN=/home/svt/ncnn-k1x-int8-smoke/bin/yolo11
  PARAM=/home/svt/ncnn-k1x-int8-smoke/models/yolo11n-int8.ncnn.param
  MODEL=/home/svt/ncnn-k1x-int8-smoke/models/yolo11n-int8.ncnn.bin
  IMAGE=/home/svt/ncnn-k1x-int8-smoke/models/photo_2024-10-11_10-04-04.jpg

  $BIN \
    --param $PARAM \
    --bin $MODEL \
    --image $IMAGE \
    --bench-only --forward-only \
    --pin cluster0 --threads 4 \
    --warmup 10 --runs 100 --repeats 3 \
    --strict-omp-env 1 --quiet --no-gui
REMOTE
```

### Что фиксировано и почему

| Параметр | Значение | Почему фиксировано |
|---|---|---|
| **Разрешение** | **640×640** | Целевое рабочее разрешение для видео |
| Модель | YOLO11n INT8 | Целевая модель проекта |
| Изображение | photo_2024-10-11_10-04-04.jpg | Одно и то же изображение, одинаковый контент |
| Pinning | cluster0 (CPUs 0-3) | Cluster0 имеет IME, исключает кросс-кластерный jitter |
| Threads | 4 | Оптимум для одного кластера (доказано в логах) |
| Warmup | 10 | Прогрев JIT/кэша, убирает cold-start |
| Runs | 100 | Достаточно для стабильного среднего |
| Repeats | 3 | 3 серии по 100, берём mean of means |
| forward_only | да | Только forward pass, без пре/пост-обработки |

### Дополнительные замеры (для анализа, не для KPI)

```bash
# FP16 reference (тот же формат, другая модель):
$BIN --param yolo11n.ncnn.param --bin yolo11n.ncnn.bin ... (остальное идентично)

# 320×320 (для быстрой итерации во время разработки):
# Пересчёт: FPS_640 ≈ FPS_320 / 4  (линейная экстраполяция по пикселям)
# Использовать для отладки, НЕ для отчётов
```

### Таблица прогресса KPI

| Неделя | Дата замера | FPS 640×640 INT8 | Δ vs предыдущая | Что сделано | Примечание |
|--------|------------|-----------------|----------------|-------------|------------|
| 0 | 2026-04-03 | — | — | Setup, Конституция | Baseline ещё не замерен на V4 |
| 1 | 2026-04-10 | ~0.05 (baseline) | — | Baseline зафиксирован | Generic INT8 fallback |
| 2 | 2026-04-17 | ? | | Инфраструктура | K-000, K-005, K-010 |
| 3 | 2026-04-24 | ? | | 1x1 GEMM + epilogue | K-100, K-150 |
| 4 | 2026-05-01 | ? | | 3x3 IGEMM + DW | K-120, K-140 |
| 5 | 2026-05-08 | ? | | End-to-end RVV | Первый полный прогон |
| 6 | 2026-05-15 | ? | | IME ядра | K-110, K-130 |
| 7 | 2026-05-22 | ? | | End-to-end IME | Ключевой milestone |
| 8 | 2026-05-29 | ? | | Оптимизация | Tile tuning, prefetch |
| 9 | 2026-06-06 | **≥ 5 FPS** | | Delivery | Финальный результат |

**Ожидаемая траектория:** 0.05 → 0.05 → 0.1 → 0.3 → 0.5 → 1.5 → 3-5 → 4-8 → 5-8 → ≥5 FPS

TODO: Добавить диаграмму с двумя графиками: ожидаемая vs реальная траектории  
TODO: Добавить диаграмму исполнения задач проекта: план vs факт в процентах.

---

## Обозначения

- **[0.5д]** — полдня (~4 часа)
- **[1д]** — полный день (~8 часов)
- Каждая задача привязана к kernel family (K-xxx) или правилу (R-xxx) из Конституции V4
- Задачи внутри недели можно делать параллельно (СТ и АН работают одновременно)

---

## Неделя 0: Конституция + Setup (30 марта — 3 апреля)

### Цель недели
СТ доводит Конституцию до финала. АН настраивает окружение и создаёт систему KPI-замеров.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 0.1 | Поднять Docker `bf3-ncnn/`: `cp .env.example .env`, `bash build-docker.sh`, `bash run-docker.sh`. Проверить `riscv64-unknown-linux-gnu-gcc --version` | [0.5д] | Docker работает, SpacemiT toolchain доступен | — |
| 0.2 | Получить SSH-доступ к BPI-F3 от СТ, проверить подключение, `cat /proc/cpuinfo` | [0.5д] | SSH работает, видим X60 cores | — |
| 0.3 | Клонировать NCNN fork (`github.com/RISCY-SVT/ncnn`), переключиться на ветку `k1x-int8-triage`, собрать кросс-компиляцией в Docker | [1д] | `ncnn` собирается для rv64, `benchncnn` и `yolo11` бинарники готовы | — |
| 0.4 | Создать скрипт `kpi_measure.sh` — каноническая команда замера FPS (см. раздел KPI выше). Протестировать: запустить на K1, получить число | [1д] | Скрипт работает, FPS получен | KPI |
| 0.5 | Изучить consensus plan: `specs/analysis/discussion.md`, `design.md`, `open-questions.md`. Подготовить позицию по human tiebreaks (concat scope, integration boundary) | [1д] | Записка с позицией для обсуждения с СТ | P0 |
| 0.6 | Прочитать IME Spec (pdf, 22 стр.): разделы 1-3, vmadot/vmadot1/vmadot2/vmadot3/vmadotn, схема MAC-блока 4×4×8 | [1д] | Понимание класса A (GEMM) и класса B (sliding-window) инструкций | K-110, K-130 |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 0.7 | Финализация Конституции V5: загрузить RAG (тулчейн SpacemiT, спецификации RVV/IME), запустить мульти-LLM consensus round 2 | [3д] | Constitution V5 с закрытыми P0-вопросами | Consensus |
| 0.8 | P0 evidence: валидация B-RVA22-FAST compiler flags на реальном sysroot K1 | [0.5д] | Подтверждение: `-march=...` компилируется и запускается | G1 |
| 0.9 | P0 evidence: экспорт реального ncnn-графа YOLO11n, сверка с Flowmap V4 | [0.5д] | Граф экспортирован, расхождения задокументированы | G2 |
| 0.10 | Code review собственного NCNN fork: что из `k1x-int8-triage` переиспользуемо по V5 | [1д] | Список: keep / rewrite / delete | — |

### Отчёт за неделю 0
- Docker и SSH работают (АН)
- Скрипт KPI замера работает, первый FPS зафиксирован
- Конституция V5 в работе (СТ)
- P0 evidence: compiler flags проверены, граф экспортирован

---

## Неделя 1: Конституция (финал) + Baseline (7–11 апреля)

### Цель недели
СТ завершает Конституцию, фиксирует все P0 evidence. АН фиксирует baseline и строит benchmark harness.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 1.1 | Запустить `kpi_measure.sh` на K1: зафиксировать INT8 baseline (generic fallback, ~0.05 FPS на 640×640) | [0.5д] | **Baseline KPI зафиксирован** — это знаменатель для всех будущих сравнений | KPI |
| 1.2 | Запустить FP16 baseline: ~1.77 FPS на 640×640 | [0.5д] | FP16 reference записан | R-MEA-002 |
| 1.3 | Написать benchmark harness: `bench.sh` + `parse_results.py` — автоматический запуск канонического замера, парсинг результатов, вывод таблицы прогресса | [1д] | Harness работает, выводит FPS + delta vs baseline | R-MEA-002, R-MEA-003 |
| 1.4 | Изучить INT8 квантование в NCNN: `ncnn2table` (calibration), `ncnn2int8`, scale-only symmetric scheme. Прогнать calibration на 10 COCO изображениях | [1д] | Квантованная INT8 модель YOLO11n готова | R-ACC-001 |
| 1.5 | Изучить архитектуру NCNN по документу от СТ: Layer dispatch, Mat packing, Allocator. Прочитать `convolution_riscv.cpp` — понять dispatch path | [1д] | Понимание как добавить новый kernel в NCNN | — |
| 1.6 | Написать accuracy-checker: `check_accuracy.py` — запустить INT8 модель на 100 COCO val2017, сравнить mAP с FP32 reference | [1д] | Baseline mAP записан | R-QA-002 |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 1.7 | Финализация Конституции V5: закрыть оставшиеся P0 вопросы (pack32 audit, PWL SiLU reference, IGEMM latency-hiding proof) | [2д] | **Constitution V5 финализирована** | P0 |
| 1.8 | Настроить B-RVA22-FAST build lane: CMake preset / toolchain file | [0.5д] | Оба build lanes работают (B-CMP, B-RVA22-FAST) | R-BLD-002 |
| 1.9 | Создать ветку `v4-ime-backend` от `k1x-int8-triage`, зафиксировать начальную точку | [0.5д] | Чистая ветка для работы по V5 | — |
| 1.10 | Подготовить для АН walkthrough архитектуры NCNN: dispatch path, Layer class, packing system, как добавлять kernel | [1д] | Markdown-документ или устная сессия 1 час | — |
| 1.11 | P0 evidence: capture B-CMP baseline benchmark (каноническая команда) | [0.5д] | B-CMP benchmark записан | G5 |
| 1.12 | Обсуждение с АН: human tiebreaks (concat scope, integration boundary, pack32 go/no-go) | [0.5д] | Решения зафиксированы в consensus-log | — |

### Отчёт за неделю 1
- **KPI baseline зафиксирован: ~0.05 FPS на 640×640 INT8**
- FP16 reference: ~1.77 FPS
- Constitution V5 финализирована, все P0 evidence закрыты
- Benchmark harness и accuracy checker готовы
- Human tiebreaks решены

---

## Неделя 2: Инфраструктура и подготовительные ядра (14–18 апреля)

### Цель недели
Runtime-инфраструктура (K-000, K-005, K-010), weight prepack (K-020/K-030).

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 2.1 | Добавить в benchmark harness layerbench: замер каждого слоя отдельно, идентификация hotspots. Воспроизвести hotspots_B log | [1д] | `layerbench.sh` + hotspot report | R-MEA-001 |
| 2.2 | Написать unit-test framework: шаблон для тестирования ядер (random input → наш kernel vs reference → compare) | [1д] | `test_kernel_template.cpp` | R-QA-003 |
| 2.3 | Изучить IGEMM алгоритм (arxiv:1907.02129) — indirect convolution, row-pointer indirection | [1д] | Понимание алгоритма, заметки для code review K-120 | K-120 |
| 2.4 | P0: аудит pack32 через операторный путь ncnn для каждого оператора в YOLO11n | [1д] | Audit report: какие операторы поддерживают pack32, какие нет | G3 |
| 2.5 | **Замер KPI** (пятница) | [0.5д] | FPS записан в таблицу прогресса | KPI |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 2.6 | K-000: cpuset-aware feature discovery — `hwprobe` на целевом cpuset, определение B-CMP / B-RVA22-FAST / B-IME-FAST lane | [1д] | `runtime_probe.cpp`: `discover_features(cpuset)` → lane + feature mask | K-000, R-BLD-004 |
| 2.7 | K-005: 64B-aligned arena allocator — slab classes (M-A ephemeral, M-B near-term, M-C skip buffers), Zic64b alignment law | [1д] | `arena_alloc.cpp`: aligned_alloc wrapper с 64B guarantee | K-005, R-ISA-002 |
| 2.8 | K-010: Input quantization + narrow pack3/pack4 staging — квантование входного RGB в INT8 без frame-wide pack32 | [1д] | `input_quant.cpp`: u8 image → a_pack3or4_s8 staging | K-010, R-DF-001, R-DF-002 |
| 2.9 | K-020/K-030: Weight prepack — offline transform для 1x1 (K-020) и 3x3 (K-030) весов в packed layout, 64B slab geometry | [1д] | `weight_prepack.cpp`: raw weights → w1x1_pack / w3x3_pack | K-020, K-030, R-CCH-001 |

### Отчёт за неделю 2
- Runtime инфраструктура: feature probe, allocator, input quant, weight prepack
- Layerbench и unit-test framework готовы
- pack32 audit завершён
- **KPI: ожидаем ~0.05 FPS** (инфраструктура пока не влияет на FPS)

---

## Неделя 3: Главные вычислительные ядра — RVV (21–25 апреля)

### Цель недели
K-100 (1x1 pointwise GEMM) и K-150 (fused epilogue) — два самых важных ядра. Параллельно: layout ops.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 3.1 | K-160: Concat (direct-write) — vector copy из N веток в единый packed destination, 64B-aligned | [1д] | `concat_packn.cpp` | K-160, R-DF-003 |
| 3.2 | K-170: Slice/Split — view semantics (zero-copy через offset), narrow copy только если unavoidable | [0.5д] | `split_view.cpp` | K-170, R-DF-003 |
| 3.3 | K-180: Upsample x2 — nearest-neighbor INT8, pack-preserving, vector replicate + store | [0.5д] | `upsample_packn.cpp` | K-180, R-KRN-007 |
| 3.4 | K-145: SPPF / MaxPool — sliding-window max, pack-preserving INT8 | [1д] | `sppf_packn.cpp` | K-145, R-KRN-007 |
| 3.5 | Unit-тесты для K-160, K-170, K-180, K-145 | [0.5д] | `test_layout_ops.cpp`: все тесты проходят | R-QA-003 |
| 3.6 | **Замер KPI** (пятница) | [0.5д] | FPS записан (ожидаем рост за счёт 1x1 ядра) | KPI |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 3.7 | K-100: 1x1 pointwise INT8 GEMM (RVV) — direct packed GEMM, `vle8` + `vwmul` + `vwadd` accumulate INT32, Zba address math | [1.5д] | `conv1x1_s8_rvv.cpp` | K-100, R-KRN-001 |
| 3.8 | K-150: Fused epilogue — INT32 accumulator → bias → fixed-point requant → PWL SiLU → packed INT8 store | [1.5д] | `fused_epilogue.h`: inline, вызывается из всех producer ядер | K-150, R-KRN-005, R-ACC-003 |
| 3.9 | Интеграция K-100 + K-150 в NCNN dispatch: `Convolution_riscv.cpp`, INT8 1x1 path → наш kernel | [0.5д] | 1x1 conv слои YOLO11 используют K-100 | — |
| 3.10 | Microbench K-100 на K1: замер cycles/op, сравнение с FP16 baseline | [0.5д] | Microbench результат записан | R-MEA-001 |

### Отчёт за неделю 3
- K-100 (1x1 GEMM, RVV) работает и интегрирован
- K-150 (fused epilogue) готов — SiLU теперь fused, не отдельный слой
- Layout ops (K-160, K-170, K-180, K-145) реализованы и протестированы
- **KPI: ожидаем ~0.1–0.3 FPS** (1x1 convolution ускорен, остальное — fallback)

---

## Неделя 4: 3x3 Conv и Depthwise (28 апреля — 2 мая)

### Цель недели
K-120 (3x3 IGEMM) и K-140 (depthwise 3x3) — второй и третий по важности compute kernels.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 4.1 | K-050: RVA22 memory assists — обёртки `prefetch_r()`, `prefetch_w()`, guarded `cbo_zero()` с runtime safety gate | [0.5д] | `mem_assists.h`: inline helpers | K-050, R-ISA-006 |
| 4.2 | Интеграция layout ops (K-160, K-170, K-180, K-145) в NCNN dispatch | [1д] | Concat/Split/Upsample/Pooling используют наши ядра | — |
| 4.3 | Layer-by-layer тестирование: YOLO11n с логированием, проверка что каждый слой → правильный kernel | [1д] | Лог: fallback counter → 0 (для реализованных типов) | R-QA-001 |
| 4.4 | Добавить fallback counter в layerbench: сколько слоёв используют V4 ядра vs generic | [0.5д] | Отчёт: N из M слоёв на V4 ядрах | R-QA-001 |
| 4.5 | **Замер KPI** (пятница) | [0.5д] | FPS записан | KPI |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 4.6 | K-120: Standard 3x3 INT8 IGEMM (RVV) — indirect GEMM с scalar row-pointer indirection, Zba address generation, unit-stride vector loads. Без full im2col! | [2д] | `conv3x3_igemm_s8_rvv.cpp` | K-120, R-KRN-002, R-DF-004 |
| 4.7 | K-140: Depthwise 3x3 direct spatial (RVV) — для YOLO11 Detect DWConv | [1д] | `convdw3x3_s8_rvv.cpp` | K-140, R-KRN-003 |
| 4.8 | Интеграция K-120, K-140 в NCNN dispatch + K-150 epilogue | [0.5д] | 3x3 и depthwise слои используют V4 ядра | — |
| 4.9 | Microbench K-120, K-140 на K1 | [0.5д] | Microbench результаты | R-MEA-001 |

### Отчёт за неделю 4
- Все основные compute ядра RVV готовы: K-100 (1x1), K-120 (3x3), K-140 (depthwise)
- Layout ops интегрированы в NCNN
- **KPI: ожидаем ~0.3–0.5 FPS** (3x3 — самый тяжёлый тип, его ускорение заметно)

---

## Неделя 5: Stem, Detect tail, End-to-end RVV (5–9 мая)

### Цель недели
Замкнуть RVV-path: stem (вход сети) и detect tail (выход), первый end-to-end прогон YOLO11n INT8 полностью на V4 ядрах.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 5.1 | K-146: C2PSA stub — если sensitivity-aware, использовать local FP16 carve-out; иначе INT8 passthrough | [1д] | `c2psa_stub.cpp` | K-146, R-ACC-002 |
| 5.2 | End-to-end YOLO11n INT8 прогон (RVV path): full-frame forward, замер FPS | [0.5д] | FPS на 640×640, RVV-only lane | R-MEA-001 |
| 5.3 | Accuracy validation: mAP INT8 (V4 ядра) vs FP32 reference на 100 COCO images | [0.5д] | mAP delta ≤ 1% | R-QA-002 |
| 5.4 | Отладка: если расхождения accuracy — найти и исправить проблемный слой | [1д] | Все слои корректны | — |
| 5.5 | **Замер KPI** (пятница) — **MILESTONE: первый end-to-end RVV** | [0.5д] | FPS записан | KPI |
| 5.6 | Еженедельный отчёт: RVV-only результаты, прогресс, план IME | [0.5д] | Отчёт 1 стр. | — |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 5.7 | K-090: Stem-special 3x3 s2 low-C entry (RVV) — pack3/pack4 staging → первый packN | [1.5д] | `stem_conv3x3s2_s8_rvv.cpp` | K-090, R-KRN-006 |
| 5.8 | K-190: Detect tail + late dequant — keep branch convs INT8, dequant только для tiny финальных тензоров | [1д] | `detect_tail.cpp` | K-190, R-DF-005 |
| 5.9 | Интеграция K-090, K-190 в NCNN, end-to-end RVV path замкнут | [0.5д] | Вся сеть на V4 ядрах, 0 fallbacks | — |
| 5.10 | Профилирование end-to-end: perf stat, кэш-промахи, branch mispredictions | [1д] | Профиль: топ-5 hotspots для IME-ускорения | — |

### Отчёт за неделю 5
- **MILESTONE: первый end-to-end YOLO11n INT8 на V4 RVV ядрах**
- **KPI: ожидаем ~1–2 FPS на 640×640** (все ядра RVV, нет fallbacks)
- Accuracy: mAP delta vs FP32
- Профиль: определены hotspots для IME

---

## Неделя 6: IME-ядра — основной спринт (12–16 мая)

### Цель недели
Добавить IME-варианты главных compute ядер (K-110, K-130). Это ключевое ускорение ~5-10x.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 6.1 | K-040: IME-specific weight prepack — перепаковка весов для `vmadot*` microkernels | [1д] | `weight_prepack_ime.cpp` | K-040, R-BLD-005 |
| 6.2 | Unit-тесты для IME-ядер: K-110 vs K-100 reference, K-130 vs K-120 reference — bit-exact | [1д] | `test_ime_kernels.cpp`: IME output == RVV output ± 0 | — |
| 6.3 | Обновить K-000 feature discovery: добавить IME gate (cluster0-only, XSMTVDot detection) | [0.5д] | Feature probe различает RVV / IME lanes | K-000, R-BLD-005 |
| 6.4 | Обновить benchmark harness для B-IME-FAST lane: cpuset pinning cluster0-only | [0.5д] | bench.sh поддерживает `--lane ime` | R-MEA-003 |
| 6.5 | Изучить inline asm syntax для vmadot — подготовка к помощи с K-095 | [0.5д] | Hello-world vmadot запущен на K1 | K-095 |
| 6.6 | **Замер KPI** (пятница) | [0.5д] | FPS записан (ожидаем рост от IME 1x1) | KPI |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 6.7 | K-110: 1x1 pointwise INT8 GEMM (IME) — `vmadot` replaces inner dot-product, same outer contract as K-100 | [1.5д] | `conv1x1_s8_ime.cpp/.S` | K-110, R-KRN-001 |
| 6.8 | K-130: Standard 3x3 INT8 IGEMM (IME) — `vmadot` + `vmadot1`/`vmadot2` sliding-window | [2д] | `conv3x3_igemm_s8_ime.cpp/.S` | K-130, R-KRN-002 |
| 6.9 | Интеграция IME dispatch: если lane == B-IME-FAST и cluster0, использовать K-110/K-130 | [0.5д] | Runtime auto-select RVV vs IME | R-BLD-005 |

### Отчёт за неделю 6
- IME ядра K-110 (1x1) и K-130 (3x3) работают
- Корректность: IME output == RVV output
- **KPI: ожидаем ~3–5 FPS** (IME даёт ~5x на hot convolutions)

---

## Неделя 7: IME stem + end-to-end IME (19–23 мая)

### Цель недели
Завершить IME-path (K-095), end-to-end YOLO11n INT8 на IME.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 7.1 | K-095: Stem-special 3x3 s2 (IME) — под руководством СТ | [1д] | `stem_conv3x3s2_s8_ime.cpp/.S` | K-095 |
| 7.2 | End-to-end YOLO11n INT8 (IME path): full-frame forward, замер FPS | [0.5д] | FPS IME на 640×640 | R-MEA-001 |
| 7.3 | Accuracy validation: INT8 IME vs FP32 на 100 COCO images | [0.5д] | mAP delta ≤ 1% | R-QA-002 |
| 7.4 | CI setup: GitHub Actions с qemu-user для smoke test (RVV path) | [1д] | CI pipeline: build + RVV functional test | — |
| 7.5 | **Замер KPI** (пятница) — **MILESTONE: end-to-end IME** | [0.5д] | FPS записан | KPI |
| 7.6 | Начать документацию: README (рус + англ), архитектура | [0.5д] | README.md draft | — |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 7.7 | Performance profiling IME path: perf stat, сравнение с RVV | [1д] | Профиль: где ещё теряем cycles | — |
| 7.8 | Tile-size autotune: перебор Ktile × OCtile × Ntile | [1д] | Оптимальные tile sizes | R-CCH-002 |
| 7.9 | Prefetch tuning: измерить эффект prefetch в K-100/K-110/K-120/K-130 | [0.5д] | Решение: prefetch on/off для каждого ядра | R-CCH-005 |
| 7.10 | Code review работы АН + merge | [0.5д] | CR comments, merge | — |
| 7.11 | Сравнительная таблица: RVV vs IME vs SpacemiT closed | [0.5д] | Таблица | — |

### Отчёт за неделю 7
- **MILESTONE: end-to-end YOLO11n INT8 на IME ядрах**
- **KPI: ожидаем ~4–8 FPS на 640×640**
- Сравнение: RVV vs IME vs SpacemiT ORT EP

---

## Неделя 8: Оптимизация (26–30 мая)

### Цель недели
Профилирование, тюнинг, выжать максимум FPS.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 8.1 | QA audit: packN support audit — все операторы YOLO11n используют packed path, нет тихих fallbacks | [1д] | Audit report: 100% packN coverage | R-QA-003 |
| 8.2 | Оптимизация layout ops на основе layerbench: Concat, Split, Slice — если они в top-10 hotspots | [1д] | Optimized layout ops | — |
| 8.3 | K-150 epilogue: профилирование PWL SiLU, если это hotspot — увеличить число сегментов | [1д] | Epilogue оптимизирован | R-ACC-003 |
| 8.4 | **Замер KPI** (пятница) | [0.5д] | FPS записан | KPI |
| 8.5 | Обновить сравнительную таблицу и еженедельный отчёт | [0.5д] | Отчёт | — |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 8.6 | Оптимизация hot loops: profiling → identify bottleneck → rewrite inner loop | [2д] | Ускоренные hot loops | — |
| 8.7 | Пробовать TCM для selected slabs (K1 cluster0 512KB TCM) — если memory bottleneck | [1д] | TCM evaluation: полезно / бесполезно | — |
| 8.8 | Quantize/Dequantize/Requantize: написать RISC-V RVV версии (сейчас нет — gap!) | [1д] | Векторизованные Q/DQ/RQ | — |
| 8.9 | 2-thread auxiliary check: бенчмарк на 640×640 с 2 потоками | [0.5д] | FPS @ 2 threads записан | R-THR-002 |

### Отчёт за неделю 8
- Оптимизация по hotspots
- Quantize/Dequantize RVV ядра (если дают выигрыш)
- **KPI: ожидаем ~5–8 FPS**

---

## Неделя 9: Документация, демка, delivery (2–6 июня)

### Цель недели
Достичь ≥ 5 FPS, документация, демка, финальный отчёт.

### Алексей (АН)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 9.1 | Однокнопочный скрипт: `run_yolo_demo.sh` — скачивает модель, квантует, запускает инференс на K1 | [1д] | Скрипт работает из чистого клона | — |
| 9.2 | Документация: финализация README (рус + англ), CONTRIBUTING, примеры | [1д] | Полная документация | — |
| 9.3 | Runtime log: при первом запуске выводить discovered features, cpuset, active lane | [0.5д] | Логирование по R-QA-004 | R-QA-004 |
| 9.4 | Финальный benchmark report: microbench + layerbench + full-frame, все lanes, сравнительная таблица | [0.5д] | Полный benchmark document | R-MEA-001 |
| 9.5 | **Финальный замер KPI** (пятница) | [0.5д] | **Итоговый FPS** | KPI |
| 9.6 | Подготовка финального отчёта для Гачко/Никулина | [0.5д] | 2-3 стр. отчёт с графиками прогресса FPS | — |

### Сергей (СТ)

| # | Задача | Время | Результат | Ref |
|---|--------|-------|-----------|-----|
| 9.7 | Финальная оптимизация: если < 5 FPS — профилировать, тюнить | [1.5д] | ≥ 5 FPS на 640×640 | KPI |
| 9.8 | Code review всего кода АН, финальный merge | [0.5д] | Всё слито в v4-ime-backend | — |
| 9.9 | Подготовка PR в upstream NCNN (или решение о собственном репо) | [0.5д] | PR draft или repo setup | — |
| 9.10 | Архитектурные заметки для следующей фазы (ORT EP) | [0.5д] | `notes_for_ort.md` | — |

### Отчёт за неделю 9 (ФИНАЛЬНЫЙ)
- **KPI: ≥ 5 FPS YOLO11n INT8 640×640 на K1** (достигнут / не достигнут + план)
- Accuracy: mAP delta vs FP32
- Сравнение: наш open-source vs SpacemiT ORT EP closed (11.8 FPS)
- Однокнопочная демка работает
- Код в открытом репозитории
- Готовность к следующей фазе (ORT EP)

---

## Сводная таблица трудозатрат

### По неделям

| Неделя | Даты | АН (дни) | СТ (дни) | Итого | Ключевой milestone |
|--------|------|----------|----------|-------|-------------------|
| 0 | 30.03–03.04 | 5 | 5 | 10 | Setup + Конституция |
| 1 | 07.04–11.04 | 5 | 5 | 10 | **Baseline KPI** + Конституция финал |
| 2 | 14.04–18.04 | 4.5 | 4 | 8.5 | Инфраструктура (K-000..K-030) |
| 3 | 21.04–25.04 | 4 | 4 | 8 | K-100 + K-150 + layout ops |
| 4 | 28.04–02.05 | 3.5 | 4 | 7.5 | K-120 + K-140 + интеграция |
| 5 | 05.05–09.05 | 4 | 4 | 8 | **End-to-end RVV** |
| 6 | 12.05–16.05 | 4 | 4 | 8 | IME ядра K-110, K-130 |
| 7 | 19.05–23.05 | 4 | 4 | 8 | **End-to-end IME** |
| 8 | 26.05–30.05 | 4 | 4.5 | 8.5 | Оптимизация |
| 9 | 02.06–06.06 | 4 | 3 | 7 | **≥ 5 FPS, delivery** |
| **Итого** | | **42** | **41.5** | **83.5** | |

### По kernel families

| Kernel | Описание | Автор | Неделя | Дни |
|--------|----------|-------|--------|-----|
| K-000 | Feature discovery | СТ → АН updates | 2, 6 | 1.5 |
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

---

## Целевые показатели (reference points)

| Конфигурация | FPS 640×640 | Источник |
|---|---|---|
| NCNN INT8 generic fallback (текущий baseline) | ~0.05 | ncnn-logs Dec 2025 |
| NCNN FP16 RVV (текущий best) | ~1.77 | ncnn-logs Feb 2026 |
| NCNN INT8 RVV step07 (Сергей, Feb 2026) | ~0.64 | ncnn-logs Feb 2026 |
| **Наша цель (минимум)** | **≥ 5** | Гачко satisfied |
| ORT SpacemiT EP C++ harness (closed) | ~11.8 | ncnn-logs Feb 2026 |
| ORT SpacemiT EP tarball (closed) | ~39.6 | ncnn-logs Feb 2026 |
| Наша stretch goal | ≥ 8 | — |

---

## Риски и митигация

| Риск | Вероятность | Влияние | Митигация |
|------|-------------|---------|-----------|
| IME не даёт ожидаемого ускорения из-за memory bottleneck | Средняя | Высокое | Prefetch tuning, tile-size autotune, TCM для hot slabs |
| Compiler не поддерживает rva22u64 profile syntax | Низкая | Низкое | Fallback на явный `-march=rv64gc...` с расширениями |
| SSH к BPI-F3 нестабилен | Средняя | Среднее | QEMU для функциональных тестов, batch-режим бенчмарков |
| NCNN upstream отклоняет PR | Низкая | Низкое | Собственный fork + репозиторий |
| АН не успевает освоить inline asm для IME | Средняя | Низкое | СТ помогает / берёт K-095 на себя |
| PWL SiLU не проходит accuracy gate (±1 quant level) | Низкая | Среднее | Увеличить число сегментов PWL |
| C2PSA требует полноценной mixed-precision реализации | Средняя | Среднее | FP16 carve-out, оптимизировать позже |
| KPI не растёт: ядра ускоряют, но fallback слои доминируют | Средняя | Высокое | Layerbench каждую неделю → оптимизировать top hotspot |
| Конституция затягивается дольше 2 недель | Низкая | Среднее | АН параллельно занят setup + KPI, не блокирован |
