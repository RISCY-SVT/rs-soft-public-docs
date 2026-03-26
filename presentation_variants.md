# Варианты инференса YOLO11 на BananaPi BPI-F3 (SpacemiT K1)
(Все технически и теоретически возможные)

## Платформа

| Параметр | Значение |
|----------|----------|
| SoC | SpacemiT K1 (X60 cores) |
| ISA | RV64GCV + Zfh/Zvfh + SpacemiT IME |
| VLEN | 256 бит |
| Кластеры | Cluster0: 4x X60 с IME (2.0 TOPS INT8), Cluster1: 4x X60 без IME |
| Память | 32KB L1D, 512KB L2/cluster, 512KB TCM (только cluster0) |
| ОС | Bianbu Linux (на базе Ubuntu) |
| Модель | YOLO11n (640x640, INT8/FP16) |

---

## Сводная таблица вариантов

| #  | Путь инференса | V рынка, оценка | IME факт | IME план | Зрелость | FPS 320x320 | FPS 640x640 | Трудоемкость | Ссылки |
|----|----------------|-----------------|---------|---------|----------|------------|------------|--------------|--------|
| 1  | Bianbu + ORT + SpacemiT EP | 6%              | Да | Да | Закрытый бинарник | ~35 | ~8-9 * | Вендор (0) | [Bianbu AI/ORT](https://bianbu.spacemit.com/en/ai/onnxruntime/), [BRDK](https://bianbu.spacemit.com/en/brdk/) |
| 2  | NCNN (open source RVV) | 10%             | Нет | Нет | Рабочий | ~7 * | ~1.8 | Низкая | [GitHub](https://github.com/Tencent/ncnn), [RVV ядра](https://github.com/Tencent/ncnn/blob/master/src/layer/riscv/), [YOLO экспорт](https://docs.ultralytics.com/integrations/ncnn/) |
| 3  | **NCNN + IME-ядра** (наша работа) | (входит в #2)   | Нет | Да | Нужна разработка | ~20-60 * | ~5-15 | Высокая | [NCNN](https://github.com/Tencent/ncnn), [IME spec](https://github.com/spacemit-com/riscv-ime-extension-spec) |
| 4  | ONNX Runtime (vanilla CPU) | 20%             | Нет | Нет | Базовая | ~1.2-2 * | ~0.3-0.5 | Низкая | [GitHub](https://github.com/microsoft/onnxruntime), [build docs](https://onnxruntime.ai/docs/build/inferencing.html) |
| 5  | ONNX Runtime + SpacemiT EP | (входит в #4)   | Да | Да | Закрытый | ~35 | ~8-9 * | Вендор (0) | [Bianbu AI/ORT](https://bianbu.spacemit.com/en/ai/onnxruntime/) |
| 6  | **ONNX Runtime + свой EP/MLAS** | (входит в #4)   | Нет | Да | Нужна разработка | ~20-60 * | ~5-15 | Очень высокая | [ORT EP docs](https://onnxruntime.ai/docs/execution-providers/), [UCB RISC-V fork](https://github.com/ucb-bar/onnxruntime-riscv), [discussion](https://github.com/microsoft/onnxruntime/discussions/17466) |
| 7  | TFLite + XNNPACK | 28%             | Нет | Нет | FP32 RVV, INT8 gap | ~2-4 * | ~0.5-1 | Высокая | [XNNPACK](https://github.com/google/XNNPACK), [SiFive RVV blog](https://www.sifive.com/blog/sifive-accelerates-risc-v-vector-integration-in-xnnpack-for-optimized-ai-inference) |
| 8  | **TFLite + XNNPACK + IME** | (входит в #7)   | Нет | Да | Нужна разработка | ~12-40 * | ~3-10 | Очень высокая | [XNNPACK](https://github.com/google/XNNPACK), [TFLite delegate](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/delegates/xnnpack/README.md) |
| 9  | MNN (Alibaba) | 7%              | Нет | Нет | Частичная RVV | ~4-8 * | ~1-2 | Средняя | [GitHub](https://github.com/alibaba/MNN), [RVV issue](https://github.com/alibaba/MNN/issues/3820) |
| 10 | OpenCV DNN | 8%              | Нет | Нет | Частичная RVV | ~1.2-2 * | ~0.3-0.5 | Средняя | [GitHub](https://github.com/opencv/opencv), [RVV wiki](https://github.com/opencv/opencv/wiki/OpenCV-RISC-V), [RVV blog](https://opencv.org/blog/optimizing-opencv-for-the-risc-v-architecture/) |
| 11 | Apache TVM | 9%              | Нет | Нет | Теоретически | ~4-12 * | ~1-3 | Очень высокая | [GitHub](https://github.com/apache/tvm), [RISC-V RFC](https://github.com/apache/tvm-rfcs/blob/main/rfcs/0075_RISC-V_CSI-NN2_Intergration.md), [demo](https://github.com/accelr-net/tvm-riscv-demo) |
| 12 | ExecuTorch (Meta) | 7%              | Нет | Нет | Экспериментальная | — | — | Очень высокая | [GitHub](https://github.com/pytorch/executorch) |
| 13 | GGML + YOLO (нестандартно) | 0%              | Нет | Нет | Нет поддержки CV | N/A | N/A | Нецелесообразно | [GGML](https://github.com/ggml-org/ggml), [llama.cpp](https://github.com/ggml-org/llama.cpp), [RVV PR](https://github.com/ggml-org/llama.cpp/pull/12530) |

> \* Расчётное значение. Пересчёт выполнен линейной экстраполяцией по количеству пикселей (320x320 = 102K px, 640x640 = 410K px, соотношение ~1:4).
>
> \*\* Трудоемкость: Низкая — до недели. Средняя — 2-3 недели. Высокая — больше месяца. Очень высокая — несколько месяцев.

---

## Детальное описание вариантов

### 1. Bianbu + ONNX Runtime + SpacemiT Execution Provider (вендорный путь)

**Описание:** SpacemiT предоставляет закрытый проприетарный Execution Provider для ONNX Runtime в составе Bianbu Linux SDK. EP использует IME-инструкции для ускорения INT8-инференса.

**Обоснование:**
- Единственный на данный момент способ получить реальное IME-ускорение "из коробки"
- Заявленная производительность: ~35 FPS на 320x320 (SpacemiT demo)
- Закрытый код: невозможно модифицировать, отлаживать, портировать на другие платформы

**Плюсы:** Максимальная производительность без усилий
**Минусы:** Vendor lock-in, закрытый код, нет контроля, привязка к Bianbu SDK

---

### 2. NCNN (open source, только RVV)

**Описание:** Фреймворк от Tencent для мобильного/edge инференса. Уже имеет RVV-ядра для RISC-V (FP16, FP32). Есть готовые примеры запуска YOLO на K1.

**Обоснование:**
- Наиболее зрелый open-source путь для RISC-V
- Существующий baseline: ~566 мс/кадр (1.77 FPS) на 640x640 FP16
- Хорошо структурированный код, понятная архитектура ядер
- Активное сообщество, есть RISC-V мейнтейнеры

**Плюсы:** Работает сейчас, чистый код, легко профилировать
**Минусы:** Без IME — медленно; INT8 fallback ~20 сек/кадр

---

### 3. NCNN + IME-ускоренные INT8-ядра (приоритетный путь)

**Описание:** Добавление собственных INT8-ядер с IME-инструкциями (vmadot, vmadot1-3, vmadotn) в NCNN. Это основной путь, описанный в Constitution V4 С.Тюрина.

**Обоснование:**
- Кратчайший путь от baseline (RVV FP16) к целевой производительности (IME INT8)
- Sliding-window инструкции позволяют Conv без im2col (ключевое ускорение ~10x)
- Наработки С.Тюрина: 14 семейств ядер (K-000...K-190), три профиля сборки
- Гибридная архитектура: fused epilogue int32->bias->requant->SiLU->int8
- Тестируемость: можно начать с одного ядра и видеть прогресс

**Плюсы:** Open source, максимальный контроль, переиспользуемые ядра
**Минусы:** Большой объем работы (~30 дн), требует глубокого знания ISA

**Трудоемкость:** ~30 рабочих дней (с учетом наработок Сергея Тюрина)

---

### 4. MNN (Alibaba)

**Описание:** Mobile Neural Network — фреймворк от Alibaba, аналог NCNN. Используется в Taobao, DingTalk. Имеет частичную поддержку RISC-V через RVV.

**Обоснование:**
- Подтвержденные тесты на BPI-F3 (есть отчеты в сообществе)
- Поддержка dynamic shape, что удобно для YOLO
- Менее зрелая RISC-V поддержка по сравнению с NCNN
- Документация преимущественно на китайском

**Плюсы:** Работает на K1, хорошая оптимизация памяти
**Минусы:** Слабее RISC-V поддержка, сложнее добавить IME

---

### 5. TensorFlow Lite + XNNPACK (только RVV)

**Описание:** Google TFLite использует XNNPACK как backend для CPU-инференса. XNNPACK имеет начальную поддержку RISC-V (FP32), но INT8 micro-kernels для RVV отсутствуют.

**Обоснование:**
- Стандарт для мобильного ML на Android
- Модульная архитектура micro-kernel: четкое разделение логики графа и вычислительных ядер
- FP32 RVV работает, но INT8 — основной gap
- Требуется экспорт YOLO11 в TFLite формат

**Плюсы:** Огромная экосистема, стандартный формат моделей
**Минусы:** INT8 RVV gap, необходимость конвертации модели

---

### 6. TensorFlow Lite + XNNPACK + IME micro-kernels

**Описание:** Добавление собственных INT8 GEMM micro-kernels с IME в XNNPACK. Обсуждается в gemini_task_discussion.md с примерами кода.

**Обоснование:**
- Покрывает ~22% рынка бэкендов инференса (TFLite + PyTorch Mobile + MediaPipe)
- Строгая архитектура micro-kernel — высокий порог входа, но качественный результат
- Пример сигнатуры: `xnn_f32_gemm_minmax_ukernel_4x8__spacemit_ime`
- XNNPACK сам управляет тайлингом, многопоточностью, кэш-оптимизацией

**Плюсы:** Широкое покрытие экосистемы Google, переиспользуемость ядер
**Минусы:** Сложная интеграция (~38 дн), строгие требования к формату ядер

---

### 7. OpenCV DNN

**Описание:** Встроенный модуль инференса в OpenCV. Поддерживает загрузку моделей ONNX, Caffe, Darknet. Частичная RVV-оптимизация.

**Обоснование:**
- OpenCV де-факто стандарт для CV-приложений (~4-6% tier-1 пользователей)
- Простота использования: `cv::dnn::readNetFromONNX()` + `net.forward()`
- RISC-V RVV поддержка минимальна, в основном скалярный fallback
- Для серьезного инференса уступает специализированным фреймворкам

**Плюсы:** Простой API, большая экосистема CV
**Минусы:** Медленный инференс, нет IME, ограниченная оптимизация

---

### 8. Apache TVM (автоматическая кодогенерация)

**Описание:** ML-компилятор, который берет модель (ONNX/PyTorch) и компилирует в оптимизированный код под конкретное железо. Теоретически может генерировать RVV-код.

**Обоснование:**
- "Взрослый" компиляторный подход для производителей чипов
- Описание IME-инструкций через TVM TIR позволит автоматически генерировать ядра
- Auto-tuning может найти оптимальные тайлинги для K1
- Требует написания полноценного backend для SpacemiT

**Плюсы:** Автоматизация, масштабируемость на новые модели
**Минусы:** Высокий порог входа, долгая разработка backend, нет готовой RISC-V IME поддержки

---

### 9. ONNX Runtime (vanilla CPU EP)

**Описание:** Стандартный CPU Execution Provider в ORT. Использует внутреннюю библиотеку MLAS. Без специфических RVV/IME оптимизаций.

**Обоснование:**
- Работает "из коробки" — ORT компилируется на RISC-V
- Использует скалярный C++ код, без векторных оптимизаций
- Самый медленный из вариантов ORT
- Полезен только как baseline для сравнения

**Плюсы:** Нулевая трудоемкость, стандартный ONNX формат
**Минусы:** Крайне медленный без векторных ядер

---

### 10. ONNX Runtime + SpacemiT EP (вендорный, = вариант 1)

**Описание:** Идентичен варианту 1. SpacemiT поставляет закрытый EP в составе Bianbu SDK.

---

### 11. ONNX Runtime + собственный EP/MLAS с IME

**Описание:** Написание собственного Execution Provider или добавление RISC-V IME ядер в MLAS (Microsoft Linear Algebra Subprogram) внутри ORT.

**Обоснование:**
- ORT покрывает ~30% рынка инференса (enterprise, HuggingFace Optimum)
- Один EP покрывает все модели в формате ONNX
- Важно: OpenBLAS недостаточно — нужно работать с MLAS напрямую для INT8
- SpacemiT уже сделали форк — есть отправная точка

**Плюсы:** Универсальность, enterprise-покрытие
**Минусы:** Сложная кодовая база (~42 дн), C++ boilerplate

---

### 12. ExecuTorch (Meta)

**Описание:** Новейший runtime от Meta для edge/embedded PyTorch. Замена PyTorch Mobile. Теоретически может работать на RISC-V.

**Обоснование:**
- Стратегически важен: Meta позиционирует ExecuTorch как будущее edge ML
- Текущая RISC-V поддержка: экспериментальная/отсутствует
- Делегирует вычисления в XNNPACK — поддержка XNNPACK автоматически дает ExecuTorch
- Слишком рано для production использования на K1

**Плюсы:** Будущий стандарт для PyTorch Edge
**Минусы:** Незрелый, нет RISC-V поддержки, зависит от XNNPACK

---

### 13. GGML / llama.cpp (нецелесообразно для YOLO)

**Описание:** GGML — библиотека тензорных вычислений, созданная для LLM (llama.cpp, whisper.cpp). Имеет RVV-ядра для RISC-V.

**Обоснование:**
- GGML работает на K1 (llama.cpp с RVV подтвержден)
- Но: GGML заточен под transformer-архитектуры (LLM), не под CNN/YOLO
- Нет стандартного способа запустить YOLO через GGML
- Полезен для демонстрации LLM на RISC-V, но не для задачи YOLO

**Плюсы:** Работающий RVV код, хайп-потенциал для LLM
**Минусы:** Не поддерживает YOLO, нецелесообразно для CV

---

### Дополнительно рассмотренные и отвергнутые варианты

| Вариант | Причина отклонения | Ссылка |
|---------|-------------------|--------|
| CSI-NN2 / SHL (T-Head) | Создан для T-Head C906/C910, неопределенная совместимость с K1 X60 | [GitHub](https://github.com/XUANTIE-RV/csi-nn2), [SHL](https://github.com/openvinotoolkit/shl) |
| Tengine (OPEN AI LAB) | Практически не поддерживается с 2022, мертвое сообщество | [GitHub](https://github.com/OAID/Tengine) |
| PaddlePaddle / Paddle Lite | Нет реальной RISC-V поддержки, несмотря на заявления | [GitHub](https://github.com/PaddlePaddle/Paddle-Lite) |
| Darknet | Мертвый проект, нет RISC-V поддержки | — |
| MMDeploy + NCNN | Обертка над NCNN — сводится к вариантам 2/3, добавляет сложность | — |
| Caffe / Caffe2 | Legacy, поглощены PyTorch | — |

---

## Рекомендуемая стратегия

### Приоритет 1: NCNN + IME (вариант 3)
- **Почему:** Кратчайший путь к результату. Есть baseline (RVV FP16), есть Constitution V4 с архитектурой ядер, есть наработки С.Тюрина. Sliding-window инструкции IME дают 10x ускорение для Conv без im2col.
- **Результат:** Open-source YOLO11 инференс с IME, 5-15 FPS на 640x640 INT8
- **Покрытие:** ~8% рынка tier-2 бэкендов + стратегическое значение для embedded/edge

### Приоритет 2: ONNX Runtime + свой EP (вариант 6)
- **Почему:** Максимальное покрытие enterprise-рынка (~30%). Один EP обслуживает все ONNX-модели. Ядра из NCNN переиспользуются в MLAS.
- **Результат:** Универсальный инференс любых ONNX-моделей с IME-ускорением

### Приоритет 3: XNNPACK + IME (вариант 8)
- **Почему:** Покрывает Google-экосистему (TFLite, PyTorch Mobile, MediaPipe). ~22% tier-2. Micro-kernel архитектура обеспечивает качество.
- **Результат:** Поддержка мобильных и edge ML-приложений на RISC-V

### Итого оптимальный маршрут:
```
Prerequisites (8 дн) -> NCNN+IME (45 дн) -> ORT EP (60 дн) -> XNNPACK (65 дн)

Итого: ~178 рабочих дня (~6 месяцев)
```

---

## Источники

### Платформа
- [BananaPi BPI-F3 документация](https://docs.banana-pi.org/en/BPI-F3/BananaPi_BPI-F3)
- [SpacemiT K1 datasheet](https://docs.banana-pi.org/en/BPI-F3/SpacemiT_K1_datasheet)
- [SpacemiT K1 продуктовая страница](https://www.spacemit.com/en/key-stone-k1/)
- [SpacemiT IME Extension Spec (GitHub)](https://github.com/spacemit-com/riscv-ime-extension-spec)
- [IME анализ (Remlab)](https://www.remlab.net/op/riscv-xstime.shtml)
- [Bianbu Developer Portal](https://bianbu.spacemit.com/en/)
- [SpacemiT GitHub](https://github.com/spacemit-com)
- [CNX Software обзор BPI-F3](https://www.cnx-software.com/2024/05/10/banana-pi-bpi-f3-sbc-spacemit-k1-octa-core-risc-v-ai-soc/)

### Модель
- [Ultralytics YOLO11 документация](https://docs.ultralytics.com/models/yolo11/)
- [Ultralytics GitHub](https://github.com/ultralytics/ultralytics)
- [YOLO11 NCNN экспорт](https://docs.ultralytics.com/integrations/ncnn/)
- [Форматы экспорта моделей](https://docs.ultralytics.com/modes/export/)

### Фреймворки инференса
- [NCNN GitHub](https://github.com/Tencent/ncnn) | [RISC-V ядра](https://github.com/Tencent/ncnn/blob/master/src/layer/riscv/) | [Сборка](https://github.com/Tencent/ncnn/wiki/how-to-build)
- [MNN GitHub](https://github.com/alibaba/MNN) | [RVV roadmap](https://github.com/alibaba/MNN/issues/3820)
- [XNNPACK GitHub](https://github.com/google/XNNPACK) | [SiFive RVV интеграция](https://www.sifive.com/blog/sifive-accelerates-risc-v-vector-integration-in-xnnpack-for-optimized-ai-inference)
- [TFLite XNNPACK delegate](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/lite/delegates/xnnpack/README.md)
- [OpenCV GitHub](https://github.com/opencv/opencv) | [RISC-V wiki](https://github.com/opencv/opencv/wiki/OpenCV-RISC-V) | [RVV оптимизации](https://opencv.org/blog/optimizing-opencv-for-the-risc-v-architecture/)
- [Apache TVM GitHub](https://github.com/apache/tvm) | [RISC-V RFC](https://github.com/apache/tvm-rfcs/blob/main/rfcs/0075_RISC-V_CSI-NN2_Intergration.md) | [RISC-V demo](https://github.com/accelr-net/tvm-riscv-demo)
- [ONNX Runtime GitHub](https://github.com/microsoft/onnxruntime) | [Execution Providers](https://onnxruntime.ai/docs/execution-providers/) | [UCB RISC-V fork](https://github.com/ucb-bar/onnxruntime-riscv)
- [Bianbu ORT + SpacemiT EP](https://bianbu.spacemit.com/en/ai/onnxruntime/)
- [ExecuTorch GitHub](https://github.com/pytorch/executorch)
- [GGML GitHub](https://github.com/ggml-org/ggml) | [llama.cpp GitHub](https://github.com/ggml-org/llama.cpp) | [RVV PR](https://github.com/ggml-org/llama.cpp/pull/12530) | [RVV ускорение (блог)](https://cloud-v.co/blog/risc-v-1/accelerating-llama-cpp-with-risc-v-vector-extension-3)

### Отвергнутые варианты
- [CSI-NN2 GitHub](https://github.com/XUANTIE-RV/csi-nn2) | [SHL](https://github.com/openvinotoolkit/shl)
- [Tengine GitHub](https://github.com/OAID/Tengine)
- [PaddlePaddle Lite GitHub](https://github.com/PaddlePaddle/Paddle-Lite)

### Внутренние документы проекта
- `docs/K1X_NCNN_INT8_FASTEST_BACKEND_CONSTITUTION_REVISED_V4_RVA22.docx` — Constitution V4 (семейства ядер, профили сборки)
- `docs/spacemit-ime-asciidoc.pdf` — спецификация IME расширения
