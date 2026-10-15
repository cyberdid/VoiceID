---
type: overview
tags: [синтез, теза, головна, master-context, openevolve, redimnet2]
sources: [master-context, lit-ssl-clustering, wespeaker-toolkit, voxwatch-osi-benchmark, pretrained-models-review, voice-id-parameters-methods, nb-voxceleb-real-audio-benchmark]
updated: 2026-10-06
---

# VoiceID + OpenEvolve — Генеральний огляд і майстер-теза

Головна сторінка wiki та єдине джерело правди (**Single Source of Truth**) проєкту.  
Повний архітектурний контекст зафіксовано в **[[master-context]]** (`docs/VOICE_ID_OPENEVOLVE_MASTER.md`).  
Каталог усіх сторінок — [[index]]. Хронологія — [[log]].

---

## 0. Що ми будуємо (Постановка задачі)

**Large-Scale Open-World Speaker Discovery and Identification**

Мета — не просто закрита перевірка «чи це голос відомої особи», а автоматичне виявлення невідомої глобальної структури дикторів у неоднорідному масиві **~3 TB аудіо**:

```text
3 TB heterogeneous audio (RU, UA, Surzhyk, code-switching)
        ↓
Stream decode + Resample (CPU workers)
        ↓
Silero VAD (timestamps, voiced duration)
        ↓
Speaker homogeneity / Conditional local diarization
        ↓
Speaker-homogeneous segments (4–8 с)
        ↓
ReDimNet2 (Production Baseline Encoder)
        ↓
L2-normalized speaker embeddings (Binary/Arrow shards + Postgres)
        ↓
FAISS / cuVS ANN candidate retrieval
        ↓
Adaptive edge scoring (OpenEvolve evolved functions)
        ↓
Mutual sparse speaker graph
        ↓
Unknown-K clustering & conservative merge/split
        ↓
Stable anonymous speaker clusters (Speaker_000001 ... Speaker_N)
        ↓
Human audit / Verified real-world identity resolution
```

### Залізний закон проєкту
> **«False merge набагато гірший за false split.»**  
> Краще тимчасово мати два окремі кластери на одну фізичну людину, ніж випадково злити двох різних людей в один кластер.

---

## 1. Тріада системи: VoiceID + VoiceEvolve + OpenEvolve

1. **VoiceID:** цільовий виробничий конвеєр великомасштабної ідентифікації дикторів на 4× NVIDIA RTX 4090 та DGX Spark.
2. **OpenEvolve** (`algorithmicsuperintelligence/openevolve`, Apache-2.0): відкритий агентний рушій еволюції коду на базі LLM у стилі *AlphaEvolve* (Island Model, MAP-Elites, мутації коду Python).
3. **VoiceEvolve:** повна governance-платформа досліджень (ізольований запуск, детермінований Evaluator, 5 Promotion Gates, збереження лінійки алгоритмів). Детальніше — [[concepts/openevolve]].

### Чому OpenEvolve замість простої Optuna
Optuna шукає лише скалярні числа ($k$, threshold, margin).  
**OpenEvolve мутує саму логіку коду Python**, замінюючи наївне косинусне порівняння на топологічні формули близькості з урахуванням зворотного сусідства (mutual-kNN), спільного оточення та локальної неоднозначності.

---

## 2. Модельний ряд: ReDimNet2 як основний бейзлайн

- **Production Baseline:** **[[models/redimnet2]] (ReDimNet2-B6)** — найкраще співвідношення точності та обчислювальної складності на 4× RTX 4090.
- **Efficiency Tier:** **ReDimNet2-B3** — швидкий варіант для ітеративних циклів мутацій.
- **Дослідницький шортліст:**
  - **[[models/simam-resnet]] (SimAM_ResNet34):** 110k дикторів VoxBlink2, офіційно Multilingual.
  - **[[models/3d-speaker]] (ERes2NetV2):** лідер за стійкістю на чужому домені (CNCeleb 6,14%) та коротких 2-секундних чанках (1,48%).
  - **[[models/3d-speaker]] (CAM++):** ультралегка модель (7,2M параметрів) для масового скринінгу.
  - **[[models/wespeaker]] (ResNet34-LM):** опорна контрольна точка.

---

## 3. Сім цілей алгоритмічної еволюції (Mutation Targets)

OpenEvolve автоматично знаходить та покращує такі модулі конвеєра:
1. **Target №1 — Similarity Scoring:** зважування ребер графа (косинус + reciprocal support + shared neighbors − ambiguity).
2. **Target №2 — Adaptive k:** динамічний вибір кількості сусідів за локальною щільністю.
3. **Target №3 — Edge Acceptance:** критерії валідації ребер mutual-kNN графа.
4. **Target №4 — Cluster Merge:** консервативні правила злиття кластерів.
5. **Target №5 — Split Detector:** виявлення та розділення контамінованих (забруднених) кластерів.
6. **Target №6 — Outlier Reassignment:** приєднання ізольованих фрагментів.
7. **Target №7 — Speaker Prototype:** мульти-центроїдне представлення голосу мовця.

---

## 4. Ієрархія валідації (5 Promotion Gates)

Жоден згенерований алгоритм не потрапляє у продакшн без проходження гейтів:
- **Gate 1 (Validity):** детермінізм, відсутність помилок виконання та NaN.
- **Gate 2 (Safety):** нульовий витік (0 leakage), суворий пріоритет уникнення False Merges.
- **Gate 3 (Quality):** перемога над бейзлайнами (HDBSCAN, Sparse Ward AHC, Leiden+CPM).
- **Gate 4 (Robustness):** стійкість до телефонії 8 кГц, шумів, суржику та чанків 2–3 с.
- **Gate 5 (Cost):** відповідність бюджету пам'яті та лінійна масштабованість на 3 TB.

---

## 5. Пов'язані ресурси проєкту

- **Майстер-контекст (SSOT):** [[master-context]]
- **Хмарний хаб Google NotebookLM:** [VoiceID у NotebookLM](https://notebook.google.com/notebook/05bff619-45df-4903-b2ac-1bb8e7117a47)
- **Локальний пакет досьє:** `notebooklm/`
- **Протокол досліджень:** [[experiments/protocol]] та [[experiments/clustering-pilot]]
