# VOICE_ID + OpenEvolve — Master Technical Context

**Project:** Large-Scale Open-World Speaker Discovery and Identification  
**Research layer:** VoiceEvolve  
**Evolution engine:** OpenEvolve  
**Primary encoder baseline:** ReDimNet2  
**Compute:** 4× NVIDIA RTX 4090 + NVIDIA DGX Spark  
**Target corpus:** ~3 TB heterogeneous speech audio  
**Primary languages:** Russian, Ukrainian, Surzhyk / code-switching  
**Document purpose:** єдиний детальний контекст для R&D, LLM/agent workflows, implementation, benchmarking, algorithm discovery, scientific review та подальшої підготовки paper/patent materials.

---

## 0. Коротко: що ми будуємо

Мета — не просто перевіряти «чи це голос відомої людини».

Мета — взяти приблизно **3 TB аудіо**, де:

- кількість реальних людей наперед невідома;
- більшість файлів не мають надійного `person_id`;
- одна людина може зустрічатися в тисячах файлів;
- один файл може містити кілька людей;
- записи можуть бути російською, українською, суржиком або з code-switching;
- присутні різні телефони, кодеки, мікрофони, шуми, тривалість, 8/16 kHz, Opus/AMR/GSM/MP3/AAC тощо;
- одна й та сама людина може звучати по-різному в різних умовах.

Потрібно автоматично отримати:

```text
3 TB audio
→ speech
→ speaker-homogeneous segments
→ speaker embeddings
→ unknown global speaker structure
→ Speaker_000001 ... Speaker_N
→ later: verified real-world identity
```

Ключова вимога:

> **False merge набагато гірший за false split.**

Тобто краще тимчасово мати:

```text
Speaker_001
Speaker_847
```

які насправді є однією людиною,

ніж випадково об'єднати двох різних людей в один cluster.

---

# 1. Правильна постановка задачі

Назва задачі:

> **Large-Scale Open-World Speaker Discovery and Identification**

Це не звичайний closed-set VoiceID.

## 1.1 Open-world

Система повинна нормально працювати, коли з'являється людина, якої вона ніколи раніше не бачила.

Можливі рішення для нового embedding:

```text
ATTACH_TO_EXISTING_CLUSTER
NEW_CLUSTER
OUTLIER
UNRESOLVED
```

`UNKNOWN` / `UNRESOLVED` — це нормальний результат, а не помилка.

---

## 1.2 Unknown-K

Ми не знаємо:

\[
K = \text{кількість реальних дикторів}
\]

Тому production algorithm не має права отримати `n_clusters=118`, `n_clusters=5000` тощо.

Known-K experiments допускаються лише як **diagnostic/oracle control**, але не як production winner.

---

## 1.3 Speaker cluster ≠ person

Це фундаментальна межа:

```text
speaker_cluster_id != person_id
```

`Speaker_001234` означає:

> «за акустичними доказами ці записи з високою ймовірністю належать одному фізичному голосу».

Це ще не означає:

> «ця людина — Іван Іванов».

Identity додається окремо, коли з'являється надійний reference voice або інші підтверджені дані.

---

# 2. Повний production pipeline

```text
IMMUTABLE RAW AUDIO
        ↓
inventory / metadata / SHA256 / provenance
        ↓
stream decode + resample
        ↓
Silero VAD
        ↓
speaker homogeneity / change detection
        ↓
conditional local diarization
        ↓
speaker-homogeneous speech segments
        ↓
ReDimNet2
        ↓
L2-normalized speaker embeddings
        ↓
versioned embedding store
        ↓
FAISS / cuVS ANN candidate retrieval
        ↓
adaptive speaker similarity / edge scoring
        ↓
mutual sparse speaker graph
        ↓
unknown-K clustering
        ↓
risk-aware merge / split / outlier handling
        ↓
stable anonymous speaker clusters
        ↓
human audit / evidence
        ↓
separate identity resolution
        ↓
person database
```

---

# 3. Data planes

Система має три різні типи даних.

| Plane | Що зберігаємо | Політика |
|---|---|---|
| Raw evidence | original audio, hash, URI, provenance | immutable |
| Derived evidence | VAD, diarization, segments, embeddings, quality | append-only / versioned |
| Decision state | cluster membership, merge, split, identity link | mutable, але повністю versioned |

## Чому це важливо

Якщо завтра ReDimNet2 B6 буде замінений іншим encoder:

- старі embeddings не видаляються;
- нові embeddings отримують новий `encoder_version`;
- рішення можна порівняти;
- можна відкотити clustering;
- можна відтворити будь-який experiment.

---

# 4. Stage 0 — Inventory

До ML потрібно пройти весь corpus через metadata inventory.

Мінімальні поля:

```text
audio_file_id
path / URI
sha256
file_size
codec
bitrate
sample_rate
channels
duration
container
source
provenance
corruption_status
ingest_timestamp
```

3 TB не дорівнює фіксованій кількості годин.

Приблизно:

| Середній bitrate | Години для 3 TB |
|---:|---:|
| 256 kbit/s | ~26k h |
| 128 kbit/s | ~52k h |
| 64 kbit/s | ~104k h |
| 32 kbit/s | ~208k h |

Тому capacity planning починається не з «у нас 3 TB», а з реального inventory.

---

# 5. Stage 1 — Audio frontend

Не потрібно попередньо перетворювати весь corpus у persistent WAV 16 kHz.

Правильніше:

```text
original
→ stream decode
→ temporary PCM
→ resample
→ VAD / segmentation
```

Інструменти:

- FFmpeg / ffprobe;
- якісний resampler;
- CPU workers;
- prefetch queues.

---

# 6. Stage 2 — VAD

Початковий baseline:

> **Silero VAD + ONNX Runtime**

Завдання:

- видалити тишу;
- залишити voiced regions;
- отримати timestamps;
- оцінити voiced duration.

Результат:

```text
recording_id
speech_start_ms
speech_end_ms
speech_duration
```

---

# 7. Stage 3 — Local diarization

Дуже важливо не змішувати дві різні задачі.

## Local diarization

Питання:

> «Хто говорить коли всередині одного конкретного recording?»

Наприклад:

```text
00:00–00:08 → LOCAL_SPK_0
00:08–00:21 → LOCAL_SPK_1
00:22–00:35 → LOCAL_SPK_0
```

## Global discovery

Питання:

> «Чи LOCAL_SPK_0 з recording A є тією самою фізичною людиною, що LOCAL_SPK_3 з recording B?»

Це зовсім інша задача.

---

## 7.1 Diarization не запускаємо на всьому

Рекомендований cascade:

```text
VAD
↓
cheap homogeneity / speaker-change test
↓
clearly single-speaker?
    YES → ordinary segmentation
    NO  → expensive diarization
```

Candidate diarization engines:

- pyannote;
- NVIDIA NeMo Sortformer;
- Streaming Sortformer;
- VBx.

---

# 8. Stage 4 — Speaker-homogeneous segments

Після VAD/diarization створюємо units, які повинні містити один голос.

Приблизна стартова політика:

- hard minimum voiced speech: ~1.5–2 s;
- preferred window: ~4–8 s;
- довгі turns розбиваються;
- overlap / severe contamination quarantine;
- timestamps зберігаються.

Metadata:

```text
segment_id
recording_id
local_speaker_id
start_ms
end_ms
speech_duration
SNR
quality
overlap_probability
language_prob_RU
language_prob_UA
language_prob_other
codec
channel
source_domain
encoder_version
frontend_version
```

---

# 9. Stage 5 — ReDimNet2

Поточний production baseline:

> **ReDimNet2-B6**

Efficiency tier:

> **ReDimNet2-B3**

ReDimNet2 не оголошується абсолютним winner.

Причина:

paper EER з різних робіт не можна напряму порівнювати через різні:

- training datasets;
- augmentation;
- calibration;
- trial lists;
- checkpoints;
- scoring protocol.

Вибір encoder робиться на нашому benchmark.

---

## 9.1 Чому ReDimNet2

Причини:

- сильний accuracy/compute Pareto;
- відносно компактний;
- придатний для великого batch inference;
- сильна історична спадковість із ReDimNet;
- добре масштабується на RTX 4090;
- достатньо дешевий для iterative research.

---

# 10. Embedding extraction на 4× RTX 4090

Для corpus inference:

```text
GPU0 → worker 0
GPU1 → worker 1
GPU2 → worker 2
GPU3 → worker 3
```

Це **4 independent workers**, а не DDP.

DDP потрібен для synchronized training.

Worker:

```text
queue lease
→ CPU decode/resample prefetch
→ dynamic length batching
→ GPU inference
→ L2 normalize
→ append vector
→ transactional completion
```

---

# 11. Embedding storage

Vectors не слід зберігати як десятки мільйонів звичайних PostgreSQL blobs.

Рекомендовано:

```text
PostgreSQL
→ IDs
→ metadata
→ state
→ lineage
→ shard offsets

Binary/Arrow vector shards
→ actual FP16 vectors

Parquet
→ manifests / analytics
```

Обов'язково:

```text
encoder_name
checkpoint_hash
encoder_version
frontend_version
dimension
dtype
normalization
```

Ніколи не змішуємо embeddings різних vector spaces без явної transform/migration procedure.

---

# 12. ANN — candidate generation

Для десятків мільйонів embeddings dense \(N^2\) similarity неможлива.

Тому:

```text
embedding
→ ANN
→ top-k candidate neighbours
```

Основні candidate engines:

- FAISS;
- cuVS;
- CAGRA;
- IVF-Flat / IVF-PQ для більшого масштабу;
- за потреби DiskANN/SPANN-подібні SSD-рішення.

Research supergraph:

\[
k_{max}\approx100-200
\]

Будуємо один high-recall graph, після чого багато experiments можуть повторно використовувати його без нового ANN search.

---

# 13. Не просто cosine threshold

Найпростіший baseline:

\[
s_{ij} = cos(e_i,e_j)
\]

Але production edge decision має бути сильнішим.

Feature vector:

\[
x_{ij} =
[
cos_{ij},
rank_i(j),
rank_j(i),
sharedNN,
Jaccard(N_i,N_j),
density_i,
density_j,
duration_i,
duration_j,
quality_i,
quality_j,
language_i,
language_j,
channel,
codec,
constraints
]
\]

Потім:

\[
p_{ij}=P(same\ speaker|x_{ij})
\]

На старті calibrator може бути:

- logistic regression;
- GBDT;
- small MLP.

GNN — пізніше, якщо прості methods не дають достатнього результату.

---

# 14. Speaker graph

Base safe edge:

```text
high same-speaker score
AND
mutual/reciprocal neighbour
AND
no cannot-link
AND
sufficient quality
```

Можна мати два threshold:

```text
T_core > T_fringe
```

### Core edge

Має право формувати cluster.

### Fringe edge

Може бути evidence для:

- attachment;
- outlier recovery;
- merge verification;

але не має самостійно склеювати великі clusters.

---

# 15. Seed algorithms, які ми проганяємо

Це початкові families, від яких починається research.

---

## 15.1 HDBSCAN

Дуже просто:

> шукає щільні групи точок і може залишати сумнівні точки як noise.

Плюси:

- unknown-K;
- outliers;
- історично сильний baseline у VoiceID.

Мінуси:

- глобальна щільність може змінюватися між domains;
- менш explicit control кожного небезпечного edge;
- складніше incremental production logic.

Роль:

> mandatory Tier-1 baseline.

---

## 15.2 Sparse Ward / AHC

Дуже просто:

> спочатку кожна точка/група окрема, потім найближчі clusters поступово об'єднуються.

Sparse version:

> дозволяємо merge лише між candidate neighbours із sparse connectivity graph.

Плюси:

- зрозуміла merge logic;
- сильний historical baseline;
- зручний після microcluster contraction.

Мінуси:

- threshold / stopping criterion критичний;
- dense AHC не масштабується;
- known-K version не підходить production.

---

## 15.3 Threshold Graph

Дуже просто:

```text
if similarity > threshold:
    create edge
```

Після цього connected components / community detection.

Плюс:

- дуже простий і пояснюваний baseline.

Мінус:

- один глобальний threshold слабко працює на heterogeneous domains.

---

## 15.4 Mutual-kNN + Leiden

Дуже просто:

> A і B з'єднуються, якщо вони обидва вважають один одного близькими сусідами.

Потім Leiden розділяє sparse graph на communities.

Плюси:

- unknown-K;
- explicit graph;
- scalable;
- reciprocal edges відсіюють багато випадкових similarities.

Це один із головних production candidates.

---

## 15.5 Leiden + CPM

Leiden — community detection.

CPM — objective, який дозволяє контролювати бажану internal density community.

Потенційно корисно, бо один fixed global threshold не завжди відображає різну density голосових clusters.

---

## 15.6 Reciprocal FINCH

FINCH використовує nearest-neighbour relations, щоб автоматично будувати clusters без заданого K.

Для нашої задачі розглядаємо **conservative/reciprocal variant**.

Роль:

- fast unknown-K challenger;
- seed для evolution.

---

## 15.7 Adaptive mutual-kNN

Не задаємо всім:

```text
k = 30
```

Замість цього:

\[
k_i=f(local\ density, quality, ambiguity)
\]

Щільному cluster може вистачити малого `k`.

Складному sparse region може знадобитися більше neighbours.

Це одна з потенційно нових algorithmic areas.

---

## 15.8 Calibrated edge graph

Edge визначається не лише cosine similarity.

```text
cosine
+ reciprocal rank
+ shared neighbours
+ density
+ duration
+ quality
+ language
+ codec
+ channel
+ local constraints
```

Це головна різниця від примітивного threshold graph.

---

## 15.9 Core-first multi-prototype clustering

Спочатку будуємо лише дуже надійне core cluster.

Потім fringe points додаються окремо.

Кластер представляється не одним centroid, а:

- centroid;
- medoid;
- best exemplars;
- boundary exemplars;
- RU prototype;
- UA prototype;
- telephone prototype;
- wideband prototype;
- radius statistics.

---

## 15.10 Hybrid: Leiden → split → conservative merge

Один із найсильніших design candidates:

```text
ANN
→ calibrated graph
→ Leiden initial communities
→ suspicious-cluster split
→ conservative inter-cluster merge
→ outlier reassignment
```

Partition — це лише proposal.

Final identity cluster проходить окрему verification logic.

---

# 16. Merge logic

Ніколи:

```text
centroid(A) similar centroid(B)
→ merge
```

Merge має вимагати **кілька незалежних evidence signals**.

Наприклад:

- centroid similarity;
- medoid similarity;
- top cross-cluster similarities;
- lower quantiles;
- multiple independent recordings;
- reciprocal support;
- cluster radius;
- language/channel composition;
- competitor margin;
- cannot-link contradictions;
- number of independent witnesses.

Thresholds:

```text
T_candidate < T_attach < T_merge
```

`T_merge` — не просто число, а policy.

Чим більші clusters — тим суворішим має бути merge.

---

# 17. Split detector

Великий cluster може бути contaminated.

Signals:

- large radius;
- internal multimodality;
- low edge density;
- bimodal similarity distribution;
- unstable Leiden partitions;
- cannot-link conflicts;
- competing identity evidence;
- різкий radius growth після merge.

Підозрілий cluster:

```text
local exact similarity
→ high-resolution Leiden
→ local HDBSCAN
→ spectral refinement
→ optional learned graph refinement
```

---

# 18. Outlier / unresolved logic

Система не повинна максимізувати coverage будь-якою ціною.

Можливі final states:

```text
ATTACHED
NEW_CLUSTER
OUTLIER
UNRESOLVED
```

Для attachment потрібні:

- sufficient same-speaker probability;
- top1/top2 margin;
- cluster support;
- quality;
- no contradictory evidence.

---

# 19. Stable anonymous speaker state

Приклад:

```text
Speaker_001238
```

Зберігаємо:

```text
cluster_id
cluster_version
centroid
medoid
best exemplars
hard exemplars
language prototypes
channel prototypes
telephone prototype
wideband prototype
p10 similarity-to-centroid
p50
p90
radius
confidence
creation_event
merge/split lineage
```

---

# 20. Identity resolution

Identity з'являється пізніше.

```text
trusted reference voice
→ embedding
→ ANN against speaker clusters
→ candidate cluster
→ verification
→ identity evidence
→ person_id
```

Schema:

```text
speaker_cluster
person
identity_evidence
```

Одній людині може тимчасово відповідати кілька speaker clusters.

Це допустимий false split.

---

# 21. Incremental production

Після initial backfill ми не recluster 3 TB при кожному новому файлі.

```text
new audio
→ VAD / diarization
→ ReDimNet2
→ prototype ANN
```

Далі:

```text
strong match
→ attach

ambiguous
→ unresolved queue

no match
→ novel microcluster pool
```

Cadence:

```text
hourly
→ strict attachments

daily
→ novel pool clustering

weekly
→ merge/split/outlier consolidation

monthly/model release
→ broad shadow rebuild
```

---

# 22. Evaluation philosophy

Головний принцип:

\[
Cost(FalseMerge) \gg Cost(FalseSplit)
\]

Тому objective не:

> «отримати максимум ARI».

А:

> «максимізувати coverage тільки всередині frozen false-merge safety envelope».

---

# 23. Основні metrics

## Pairwise

\[
FMR = \frac{FP}{TP+FP}
\]

\[
FSR = \frac{FN}{TP+FN}
\]

Також:

- Pair Precision;
- Pair Recall;
- Pair F1.

---

## Clustering

- B-cubed Precision/Recall/F1;
- V-measure;
- ARI;
- NMI;
- homogeneity;
- completeness;
- fragmentation.

---

## Cluster contamination

Для cluster \(c\):

\[
Impurity(c)=1-\frac{\max_s n_{c,s}}{|c|}
\]

Вимірюємо:

```text
P50
P95
P99
MAX
```

---

## Coverage

Не можна дозволити algorithm сказати:

```text
все OUTLIER
```

і отримати ідеальну precision.

Тому coverage вимірюється окремо.

Центральна ідея:

> **SafeCoverage**

Coverage рахується лише серед candidates, які проходять false-merge safety constraints.

---

# 24. Benchmark partitions

```text
TRAIN
→ supervised model training

SEARCH-DEV
→ frequent algorithm evolution

SHADOW-DEV
→ rare promotion gate

HIDDEN-TEST
→ release candidates

FINAL-BLIND
→ final evaluation
```

Усі speakers — disjoint.

Candidate code не бачить hidden labels.

---

# 25. VoiceEvolve

VoiceEvolve — це **не одна library**.

Це наша повна research platform.

Формула:

```text
LLM / algorithm evolution
→ candidate code
→ isolated execution
→ deterministic evaluator
→ scientific safety gates
→ population / lineage
→ reviewer
→ human promotion
```

Коротко:

> **OpenEvolve — evolution engine.  
> VoiceEvolve — вся research governance система навколо нього.**

---

# 26. Що таке OpenEvolve

OpenEvolve — open-source evolutionary coding agent.

Його основна ідея:

```text
початковий program
→ LLM пропонує mutation
→ candidate запускається
→ evaluator дає metrics
→ кращі candidates залишаються
→ вони стають parents
→ LLM створює наступні variants
→ repeat
```

OpenEvolve — open-source implementation у стилі AlphaEvolve.

Поточний upstream:

> `algorithmicsuperintelligence/openevolve`

Ліцензія:

> Apache-2.0

Python:

> >= 3.10

---

# 27. Чим OpenEvolve відрізняється від Optuna

## Optuna

Шукає переважно числа:

```text
k = 30
threshold = 0.72
margin = 0.05
resolution = 1.2
```

## OpenEvolve

Може змінити саму логіку:

```python
score = cosine
```

на:

```python
score = (
    cosine
    + alpha * reciprocal_support
    + beta * shared_neighbors
    - gamma * local_ambiguity
)
```

Або навіть повністю інший sequence:

```text
graph
→ prune
→ core
→ attach
→ split
```

Тобто OpenEvolve потрібен, коли ми хочемо знайти **нову algorithmic logic**, а не лише optimal hyperparameters.

---

# 28. Чому OpenEvolve вибраний для VoiceID

Причини:

1. **Може еволюціонувати код**, а не лише числа.
2. Підходить для functions, які можна автоматично evaluate.
3. Підтримує population-based search.
4. Підтримує island model.
5. Підтримує MAP-Elites / quality-diversity style search.
6. Можна використовувати multi-model LLM ensemble.
7. Є cascade evaluation.
8. Є experiment artifacts / feedback.
9. Є checkpoints / resume.
10. Open-source і Apache-2.0.
11. Підходить для OpenAI-compatible endpoints і локальних LLM servers.
12. Добре вкладається у наш deterministic evaluator.

---

# 29. Population — чому не один champion

Якщо завжди тримати лише «найкращий» algorithm, search може рано застрягти.

Наприклад:

```text
Algorithm A
→ excellent precision
→ poor coverage

Algorithm B
→ slightly worse precision
→ great scale

Algorithm C
→ best cross-language
→ average overall

Algorithm D
→ very simple / interpretable
```

Вони всі можуть бути цінними.

Тому OpenEvolve тримає population.

---

# 30. Island model

Приклад:

```text
Island 1 → graph/density ideas
Island 2 → robust statistics
Island 3 → merge/split
Island 4 → outlier/prototype
Island 5 → hybrids
```

Islands певний час еволюціонують незалежно.

Періодично найкращі ideas мігрують між islands.

Це знижує ризик premature convergence.

---

# 31. MAP-Elites

Замість одного leaderboard:

```text
best score
```

зберігаємо різні high-quality solutions у різних niches.

Наприклад dimensions:

```text
false_merge_safety
coverage
runtime
complexity
cross_language_score
memory_efficiency
```

Це особливо корисно для VoiceID, тому що у нас немає одного універсального scalar objective.

---

# 32. Cascade evaluation

Не треба запускати 10M benchmark для кожної випадкової mutation.

OpenEvolve підтримує multi-stage evaluation.

Для VoiceEvolve:

```text
Stage 0
syntax/static checks

Stage 1
contract tests

Stage 2
tiny smoke dataset

Stage 3
cheap SEARCH-DEV subset

Stage 4
false-merge gate

Stage 5
full SEARCH-DEV

Stage 6
multi-seed

Stage 7
SHADOW-DEV
```

95% слабких candidates повинні помирати дешево.

---

# 33. OpenEvolve architecture у нашому випадку

```text
                    LLM
                     │
             mutation proposal
                     │
                     ▼
              OpenEvolve
        ┌────────────┼────────────┐
        │            │            │
    Controller    Population    Database
        │          / islands       │
        └────────────┼────────────┘
                     ▼
               candidate code
                     │
                     ▼
                  sandbox
                     │
                     ▼
               VoiceID evaluator
                     │
             metrics + artifacts
                     │
                     ▼
                 OpenEvolve
                     │
          selection / archive
                     │
                     ▼
              next generation
```

---

# 34. Що OpenEvolve має право змінювати у VoiceID

Перший research level — **cached embeddings**.

Mutable:

```text
embedding transform
similarity function
adaptive k
neighbour selection
graph construction
edge calibration
edge pruning
community detection wrapper
split detector
merge policy
outlier reassignment
prototype builder
score normalization
```

Це називаємо:

> **Level A — algorithm discovery over frozen embeddings**

---

# 35. Що OpenEvolve НЕ має права змінювати у v1

Frozen:

```text
hidden labels
benchmark manifests
metric implementations
safety thresholds
promotion policy
resource limits
test split
production deployment pointer
ReDimNet2 internal architecture
local diarization boundaries
```

LLM не може:

- змінити test labels;
- прибрати hard speakers;
- змінити metric;
- сховати outliers;
- змінити denominator;
- вибрати seed після результату;
- сам себе promote.

---

# 36. Приклад mutation target №1 — similarity

Seed:

```python
def similarity(a, b, ctx):
    return cosine(a, b)
```

OpenEvolve може знайти:

```python
def similarity(a, b, ctx):
    s = cosine(a, b)
    r = reciprocal_rank_score(ctx)
    n = shared_neighbor_score(ctx)
    q = min(ctx.quality_a, ctx.quality_b)
    return s + 0.08*r + 0.05*n + 0.03*q
```

Але evaluator визначає, чи це реально краще.

---

# 37. Mutation target №2 — adaptive k

Seed:

```python
k = 30
```

Candidate:

\[
k_i=f(local\ density, quality, ambiguity)
\]

Наприклад:

```text
dense clear area → k=10
ambiguous area → k=40
sparse area → k=70
```

Мета:

- не додавати небезпечні edges у щільних regions;
- не втрачати same-speaker connectivity у sparse regions.

---

# 38. Mutation target №3 — edge acceptance

Seed:

```python
edge = cosine > 0.72
```

Evolution target:

```python
def accept_edge(i, j, ctx):
    ...
```

Signals:

- cosine;
- reciprocal rank;
- sharedNN;
- local density;
- quality;
- duration;
- language;
- codec;
- margin;
- local diarization constraints.

---

# 39. Mutation target №4 — cluster merge

Seed:

```python
if centroid_similarity > T:
    merge()
```

Evolution target:

```python
def should_merge(A, B, stats):
    ...
```

Вхід:

```text
centroid sim
medoid sim
top cross edges
quantiles
independent witnesses
cluster radius
language distribution
channel distribution
competitor margin
cannot-links
```

Це один із найцінніших targets через asymmetric false-merge cost.

---

# 40. Mutation target №5 — split

```python
def should_split(cluster, stats):
    ...
```

AI може комбінувати:

- internal density;
- radius;
- multimodality;
- instability;
- sub-community strength;
- contradiction counts.

---

# 41. Mutation target №6 — outlier reassignment

Історичний simple rule:

```text
nearest centroid >= threshold
AND
nearest - second_nearest >= margin
```

OpenEvolve може знайти richer rule:

```text
top1
top2
top3
margin
cluster radius
shared neighbours
prototype support
quality
domain
```

---

# 42. Mutation target №7 — speaker prototype

Baseline:

\[
prototype=\frac{1}{N}\sum_i e_i
\]

Alternatives:

- weighted mean;
- trimmed mean;
- medoid;
- centroid + medoid;
- multiple condition prototypes;
- exemplar mixture;
- quality-weighted prototypes;
- robust estimator.

---

# 43. OpenEvolve evaluator

Найважливіший принцип:

> OpenEvolve не визначає істину.

Evaluator — окремий trusted component.

Candidate отримує тільки input data і дозволені metadata.

Evaluator:

- запускає candidate;
- порівнює output із hidden ground truth;
- рахує metrics;
- повертає aggregate score/artifacts;
- не показує labels агенту.

---

# 44. Promotion hierarchy

Не один weighted score.

## Gate 1 — validity

```text
runs?
schema correct?
no leakage?
resource constraints?
```

## Gate 2 — safety

```text
false merge
catastrophic contamination
precision lower bound
```

## Gate 3 — quality

```text
SafeCoverage
B3
V-measure
ARI
fragmentation
```

## Gate 4 — robustness

```text
RU
UA
Surzhyk
telephone
short speech
codec
device
```

## Gate 5 — cost

```text
runtime
RAM
VRAM
graph size
```

---

# 45. Karpathy autoresearch

`autoresearch` — простіша research philosophy.

Ідея:

```text
agent changes training code
→ fixed-time experiment
→ objective metric
→ keep or reject
→ repeat
```

Сильна сторона — дуже короткий і зрозумілий loop.

В оригінальному design:

- fixed training environment;
- agent-modifiable training program;
- fixed evaluator/metric;
- багато коротких experiments;
- keep/reject discipline.

---

# 46. OpenEvolve vs autoresearch

| | OpenEvolve | autoresearch |
|---|---|---|
| Основна мета | еволюція program logic | швидкий autonomous experiment loop |
| Population | так | переважно current/best trajectory |
| Islands | так | ні як core concept |
| MAP-Elites | так | ні |
| Code mutations | core | core |
| Training recipe optimization | може | дуже природно |
| Algorithm family search | дуже природно | менш структуровано |
| VoiceID role | Level A algorithm discovery | Level B training loop |

---

# 47. Як вони доповнюють одне одного

Правильна модель:

```text
VoiceEvolve
│
├── OpenEvolve
│   └── evolve clustering / graph / merge / split code
│
├── autoresearch-style loop
│   └── evolve ReDimNet2 training recipe
│
├── Optuna / BOHB / ASHA
│   └── tune numerical hyperparameters
│
├── PBT
│   └── dynamic schedules
│
├── Ray
│   └── execute / schedule experiments
│
└── trusted evaluator
    └── scientific truth
```

---

# 48. Level A — OpenEvolve

Input:

> frozen ReDimNet2 embeddings

Goal:

> знайти кращий global discovery algorithm.

Перевага:

- не потрібно кожного разу rerun neural inference;
- experiments дешеві;
- attribution чиста;
- можна провести тисячі candidates.

---

# 49. Level B — autoresearch / PBT

Після стабілізації clustering backend.

Ми дозволяємо змінювати ReDimNet2 training recipe:

```text
optimizer
LR
margin
loss
hard negatives
sampling
augmentation
codec balance
RU/UA balance
language adversary
freeze/unfreeze
```

Це значно дорожчий loop, тому його не слід починати першим.

---

# 50. Optuna / ASHA / BOHB

Вони не конкурують з OpenEvolve.

Вони відповідають на інше питання.

OpenEvolve:

> «яка логіка алгоритму краща?»

Optuna:

> «які числа для цієї логіки кращі?»

ASHA/Hyperband:

> «які candidates варто зупинити раніше, щоб не витрачати GPU?»

BOHB:

> Bayesian optimization + Hyperband-style resource allocation.

---

# 51. PBT

Population Based Training потрібен переважно для model-training stage.

Замість static:

```text
LR = fixed schedule
margin = fixed
augmentation = fixed
```

PBT може змінювати їх під час training.

---

# 52. Ray

Ray — execution layer.

Він не вигадує algorithms.

Його роль:

```text
OpenEvolve requests experiment
→ Ray checks resources
→ schedules CPU/GPU task
→ captures result
→ returns artifacts
```

---

# 53. Повна research architecture

```text
                  YOU / PI
                     │
             research direction
                     │
            ┌────────┴────────┐
            │                 │
         Astra          Local LLMs
    principal scientist       │
            │                 │
            └───────┬─────────┘
                    ▼
               VoiceEvolve
                    │
       ┌────────────┼─────────────┐
       │            │             │
   OpenEvolve   autoresearch   Optuna/PBT
       │            │             │
       └────────────┼─────────────┘
                    ▼
                   Ray
                    │
                    ▼
                4× RTX4090
                    │
                    ▼
          deterministic evaluator
                    │
                    ▼
         Academic Methodology AI
                    │
                    ▼
              human promotion
```

---

# 54. Роль Astra

Astra не використовується для кожної mutation.

Astra — **Principal Scientist / Algorithm Architect**.

Завдання:

- formulate major hypotheses;
- analyze plateaus;
- review top candidates;
- explain why mechanism works;
- propose new algorithm families;
- red-team evaluator;
- review critical architecture changes;
- analyze 50–500 experiment histories.

Routine mutation generation можна віддавати дешевшим/local models.

---

# 55. DGX Spark

DGX Spark — research/control plane.

Роль:

```text
VoiceEvolve controller
OpenEvolve population manager
local LLM inference
researcher / critic / reviewer
experiment history analysis
methodology scientist
scheduler head
```

Не ставимо Spark у основний bulk ReDimNet2 extraction path за замовчуванням.

---

# 56. 4× RTX 4090

4090 server — experimental compute plane.

Основне:

- embedding extraction;
- model fine-tuning;
- diarization backlog;
- GPU ANN;
- parallel training experiments.

Під час initial backfill:

```text
GPU0–3 → embeddings
```

Пізніше:

```text
GPU0–1 → incremental embedding
GPU2   → diarization
GPU3   → research / cuVS
```

Це стартова policy, не immutable rule.

---

# 57. Academic Methodology Scientist

Це окрема роль.

Не вигадує рішення першим.

Питає:

```text
What exactly is the hypothesis?
What would falsify it?
What is the baseline?
What is the primary endpoint?
Is there leakage?
Are splits speaker-disjoint?
Do confidence intervals support the claim?
Was an ablation performed?
Does hidden test confirm it?
Does prior art already contain the method?
```

---

# 58. Три verdict levels

Не всі engineering improvements є scientific discoveries.

Рекомендується:

```text
ENGINEERING VALID
SCIENTIFICALLY SUPPORTED
PUBLICATION-GRADE
```

Приклад:

```text
Experiment: PASSED
Engineering result: useful
Scientific claim: NOT YET SUPPORTED
Reason:
- no independent dataset
- CI overlaps
- missing ablation
```

---

# 59. Chain of Evidence

Для кожного claim потрібен evidence chain.

```text
CLAIM
"Algorithm X reduces false merges by 35%"
        ↓
metrics table
        ↓
experiment_id
        ↓
git commit
        ↓
config
        ↓
dataset manifest/hash
        ↓
raw predictions
        ↓
evaluator version
```

Без цього claim не вважається publication-grade.

---

# 60. Experiment lifecycle

```text
select parent
→ explicit hypothesis
→ expected mechanism
→ minimal diff
→ critic review
→ static checks
→ immutable experiment manifest
→ isolated worktree/container
→ contract tests
→ smoke benchmark
→ cheap SEARCH-DEV
→ false-merge safety gate
→ full SEARCH-DEV
→ multi-seed
→ SHADOW-DEV
→ reviewer
→ archive / candidate / reject
```

---

# 61. Experiment manifest

Кожний experiment має зберігати:

```text
experiment_id
parent_id
hypothesis
expected_effect
mechanism
changed_files
exact_git_diff
code_hash
config_hash
data_manifest_hash
container_hash
random_seeds
resource_limits
runtime
metrics
confidence_intervals
artifacts
ablation_requirement
novelty_claim
review_status
```

---

# 62. Anti-Goodhart

VoiceEvolve може навчитися «обманювати» metric, якщо evaluator слабкий.

Тому забороняємо candidate:

- змінювати evaluator;
- читати labels;
- змінювати dataset split;
- видаляти hard samples;
- оголошувати все outlier;
- зменшувати denominator;
- використовувати `person_id` як feature;
- вибирати seed після результатів;
- змінювати safety threshold;
- promote себе самостійно.

---

# 63. Найважливіший перший benchmark

З frozen ReDimNet2 embeddings:

```text
HDBSCAN
vs
Sparse Ward/AHC
vs
Mutual-kNN + Leiden
vs
Adaptive Graph + Leiden
vs
Adaptive Graph + Leiden + merge/split
vs
Reciprocal FINCH
```

Primary plots:

```text
FMR vs Coverage
FSR vs FMR
B3 Precision vs Recall
runtime vs N
RAM/VRAM vs N
largest-cluster contamination
cross-language false merge
cross-language false split
```

---

# 64. Gold-Small

Historical regression benchmark:

- ~36,237 files;
- 118 known speakers;
- ~8.7 GB 16 kHz WAV.

Потрібно:

- reconcile 36,237 vs 36,232 usable;
- immutable manifest;
- reproduce historical ReDimNet/HDBSCAN;
- reproduce sparse AHC controls;
- add modern metrics;
- speaker-disjoint methodology.

---

# 65. Gold-Hard

Потрібен новий harder benchmark.

Over-sample:

- nearest impostors;
- same gender/age/accent;
- RU↔UA same speaker;
- Surzhyk;
- telephone;
- codec mismatch;
- device mismatch;
- short speech;
- emotional speech;
- long time gaps;
- noise;
- overlap leakage.

---

# 66. Scale benchmark

Gold labels потрібно вставляти в великі pools distractors.

```text
Gold anchors
+
1M distractors

then:
10M

then:
30M+
```

Перевіряємо, чи не руйнується purity зі зростанням search universe.

---

# 67. RU / UA / Surzhyk

Ключові tracks:

```text
RU → RU
UA → UA
RU → UA
UA → RU
RU → Surzhyk
UA → Surzhyk
cross-codec
cross-device
wideband → telephone
long → short
cross-language + device
cross-language + telephone
```

Критично:

> cross-language benchmark має використовувати тих самих bilingual speakers.

---

# 68. Cross-language encoder improvement

Після baseline:

```text
ReDimNet2
→ codec/channel multi-condition fine-tuning
→ language nuisance projection
→ AS-Norm / calibration
→ GRL only when data supports it
→ w2v-BERT 2.0 teacher/reference
→ distillation / fusion
```

Важливо не видалити speaker information разом із language information.

---

# 69. Encoder bake-off

Required candidates:

```text
ReDimNet2-B6
ReDimNet2-B3
ERes2NetV2
CAM++
SimAM-ResNet100
w2v-BERT 2.0 speaker system
ECAPA2 as R&D control
optional WavLM Large control
```

Winner визначається не paper EER.

Winner визначається downstream:

```text
false merge
false split
SafeCoverage
cross-language
short speech
production compute
```

---

# 70. Чому новий algorithm може виникнути саме тут

Open-source components самі по собі не є нашим invention.

Наприклад:

```text
ReDimNet2
FAISS
Leiden
OpenEvolve
PyTorch
```

— це building blocks.

Новизна може виникнути у:

- adaptive edge formula;
- local-density-aware `k`;
- risk-aware merge criterion;
- multi-witness cluster verification;
- split detector;
- outlier recovery;
- multi-prototype representation;
- false-merge-constrained unknown-K clustering;
- novel interaction цих components.

---

# 71. Коли можна говорити «новий алгоритм»

Не після того, як OpenEvolve видав красивий code.

Потрібно:

1. candidate не є trivial parameter tuning;
2. є чітко описаний mechanism;
3. перевершує strong baselines;
4. проходить hidden test;
5. повторюється на independent conditions;
6. є ablation;
7. improvement statistically supported;
8. prior-art search не знаходить той самий method;
9. algorithm generalizes beyond one benchmark.

---

# 72. Potential research contribution

Можлива formulation:

> **Precision-first adaptive graph clustering for large-scale open-world speaker discovery under asymmetric identity risk.**

Або:

> **Risk-aware unknown-K speaker discovery using calibrated reciprocal graphs and multi-evidence merge verification.**

Це робочі formulations, а не заявлена новизна до benchmark/prior-art review.

---

# 73. Potential patent boundary

Патентувати не:

```text
ReDimNet2 + FAISS + Leiden
```

А потенційно:

```text
specific computer-implemented method
for open-world speaker discovery
using adaptive sparse speaker graph,
risk-aware cluster verification,
incremental unknown-speaker handling,
and explicit abstention
```

Patent discussion має починатися **до public disclosure**, якщо candidate справді сильний.

---

# 74. Recommended repository layout

```text
voice-id/
├── README.md
├── MASTER_CONTEXT.md
├── configs/
│   ├── production/
│   ├── benchmark/
│   └── evolve/
│
├── data/
│   ├── manifests/
│   └── schemas/
│
├── src/
│   ├── ingest/
│   ├── audio/
│   ├── vad/
│   ├── diarization/
│   ├── embeddings/
│   ├── ann/
│   ├── graph/
│   ├── clustering/
│   ├── merge/
│   ├── split/
│   ├── outliers/
│   ├── identity/
│   └── metrics/
│
├── evaluator/
│   ├── trusted/
│   ├── hidden_labels/
│   ├── metrics/
│   └── promotion/
│
├── voiceevolve/
│   ├── openevolve/
│   ├── researcher/
│   ├── critic/
│   ├── reviewer/
│   ├── population/
│   ├── sandbox/
│   └── manifests/
│
├── autoresearch/
│   └── redimnet2_training/
│
├── experiments/
├── notebooks/
├── tests/
└── docs/
```

---

# 75. OpenEvolve integration layout

```text
voiceevolve/openevolve/
├── initial_program.py
├── evaluator_adapter.py
├── evolve_config.yaml
├── allowed_api.py
├── candidate_contract.py
└── artifacts/
```

`initial_program.py` не повинен містити весь VoiceID.

Він має містити **bounded mutable logic**.

Наприклад:

```python
def edge_score(...):
    ...

def choose_k(...):
    ...

def should_merge(...):
    ...

def should_split(...):
    ...

def assign_outlier(...):
    ...
```

---

# 76. Чому bounded functions кращі

Якщо сказати агенту:

> «перепиши весь speaker system»

ми втратимо:

- attribution;
- safety;
- reproducibility;
- meaningful ablations.

Якщо mutation area маленька:

```text
candidate function
+ fixed scaffold
+ fixed evaluator
```

можна зрозуміти, що саме дало improvement.

Це ближче до FunSearch-style algorithm discovery.

---

# 77. Search islands для першої кампанії

Я б почав із чотирьох:

## Island A — Graph geometry

```text
adaptive k
mutuality
shared neighbours
density
edge scoring
```

## Island B — Robust statistics

```text
normalization
local margins
robust prototypes
calibration
```

## Island C — Merge/split

```text
multi-witness merge
cluster radius
split detector
stability
```

## Island D — Outlier/attachment

```text
abstention
novel cluster
outlier recovery
prototype attach
```

Пізніше окремий hybrid island.

---

# 78. Fitness design для OpenEvolve

Не рекомендовано:

\[
fitness = ARI
\]

Краще constraint-first.

Pseudo:

```text
if invalid:
    reject

if FMR > safety_limit:
    reject

if catastrophic_cluster_contamination:
    reject

otherwise:
    rank by Pareto dimensions:
        SafeCoverage
        B3 F1
        FSR
        cross-language robustness
        runtime
        memory
        simplicity
```

---

# 79. Чому Pareto важливий

Може бути:

```text
Candidate A
FMR = best
Coverage = lower

Candidate B
FMR = slightly worse but safe
Coverage = much higher

Candidate C
same quality
2× faster
```

Немає одного «правильного» scalar ordering.

Тому archive має зберігати diverse Pareto-optimal candidates.

---

# 80. LLM strategy

Routine evolution:

- local model на DGX Spark;
- cheaper API models;
- model ensemble.

Astra:

- top candidate review;
- plateau analysis;
- new seed family;
- meta-analysis;
- critical algorithm invention session.

Не витрачаємо Astra на 1000 дрібних mutations.

---

# 81. Local model on DGX Spark

Spark може тримати local reasoning/coding model для:

- mutation proposals;
- critique;
- experiment summarization;
- candidate explanation;
- novelty hypothesis;
- methodology checks.

Фактичний model choice треба benchmark-нути на наших реальних tasks, а не обирати за загальним leaderboard.

---

# 82. Production ≠ research

Production:

> стабільний, conservative, versioned pipeline.

Research:

> агресивно генерує candidates.

Ніколи не дозволяємо:

```text
OpenEvolve candidate
→ automatically production
```

Правильно:

```text
OpenEvolve
→ evaluator
→ shadow
→ methodology audit
→ human review
→ production release
```

---

# 83. 90-day plan

## Days 1–15 — reproducibility foundation

- freeze Gold-Small;
- reconcile file count;
- reproduce historical results;
- implement modern metrics;
- create speaker-disjoint splits;
- physically isolate hidden labels.

Exit:

> evaluator можна довіряти.

---

## Days 16–30 — ReDimNet2 baseline

- exact B3/B6 checkpoints;
- fixed frontend;
- FP32/FP16/BF16 checks;
- RU/UA/codec/duration slices;
- baseline HDBSCAN;
- baseline sparse AHC.

Exit:

> один frozen embedding snapshot.

---

## Days 31–45 — Graph benchmark

- FAISS/cuVS;
- kmax supergraph;
- mutual-kNN;
- Leiden/CPM;
- adaptive edge baseline;
- FINCH.

Exit:

> FMR–Coverage curves для основних families.

---

## Days 46–60 — VoiceEvolve/OpenEvolve v1

- bounded mutable functions;
- OpenEvolve integration;
- 4 islands;
- cascade evaluator;
- experiment manifests;
- checkpoints;
- lineage.

Exit:

> OpenEvolve autonomously evolves clustering candidates safely.

---

## Days 61–75 — merge/split/outlier evolution

- risk-aware merge;
- split detector;
- outlier recovery;
- multi-prototype state;
- shadow benchmark.

Exit:

> є candidates, які перевершують static baselines або чітко показано, що baseline сильніший.

---

## Days 76–90 — scientific audit

- multi-seed;
- ablations;
- hidden-test promotion;
- scale distractors;
- methodology review;
- novelty/prior-art review;
- select next campaign.

Exit:

> перший defensible research conclusion.

---

# 84. Phase після 90 днів

1. Cross-language ReDimNet2 fine-tuning.
2. NAP / nuisance suppression.
3. AS-Norm / calibration.
4. w2v-BERT2 teacher.
5. distillation.
6. autoresearch training loop.
7. PBT schedules.
8. 1M/10M/30M scale.
9. incremental production prototype.
10. publication/patent decision.

---

# 85. Frozen principles

1. Open-world first.
2. Unknown-K mandatory.
3. cluster_id != person_id.
4. Local diarization != global discovery.
5. ReDimNet2-B6 is baseline, not dogma.
6. HDBSCAN remains mandatory baseline.
7. Sparse graph is primary new direction.
8. False merge >> false split.
9. Coverage only after safety.
10. Abstention is valid.
11. Raw data immutable.
12. Derived data versioned.
13. Decisions have lineage.
14. Hidden labels inaccessible.
15. LLM never acts as final evaluator.
16. OpenEvolve begins on cached embeddings.
17. Autoresearch-style loop begins later for training.
18. 4×4090 = compute plane.
19. DGX Spark = research/control plane.
20. Every scientific claim must have reproducible evidence.

---

# 86. Найкоротше пояснення всієї системи

## Production

```text
Audio
→ VAD / diarization
→ ReDimNet2
→ embeddings
→ ANN
→ speaker graph
→ unknown-K clustering
→ conservative merge/split
→ stable anonymous speakers
→ identity mapping
```

## Research

```text
Astra / local LLM
→ OpenEvolve
→ candidate algorithm
→ Ray
→ experiments
→ deterministic evaluator
→ methodology audit
→ human promotion
```

## Training improvement

```text
ReDimNet2
→ autoresearch-style experiment loop
→ PBT / Optuna / ASHA
→ new encoder snapshot
→ repeat global clustering benchmark
```

---

# 87. Ключова різниця інструментів одним реченням

**OpenEvolve**  
→ шукає нову логіку коду/алгоритму.

**Karpathy autoresearch**  
→ дає мінімалістичний autonomous loop «змінив → запустив → виміряв → залишив/відкинув», дуже зручний для training research.

**Optuna**  
→ шукає хороші числові параметри.

**ASHA / Hyperband / BOHB**  
→ економлять compute, рано зупиняючи слабкі experiments.

**PBT**  
→ змінює hyperparameters/schedules прямо під час training.

**Ray**  
→ розподіляє execution по CPU/GPU.

**Astra**  
→ principal scientist / high-value reasoning.

**Local LLM**  
→ дешевий постійний mutation/critic/reviewer worker.

**Evaluator**  
→ єдиний суддя фактичної якості.

**Academic Methodology Scientist**  
→ перевіряє, чи маємо право робити науковий висновок.

---

# 88. Основна наукова гіпотеза

Наш головний research bet:

> Для великого heterogeneous open-world speaker corpus precision-first sparse graph із adaptive/calibrated edges, reciprocal support, explicit abstention та multi-evidence merge/split verification дасть кращу false-merge / coverage frontier, ніж standalone HDBSCAN та традиційні clustering baselines.

Це має бути **спростовна гіпотеза**, а не припущення.

---

# 89. Приклад formal hypothesis

Не:

> «Adaptive graph буде кращим».

А:

> На speaker-disjoint hidden test adaptive reciprocal graph з calibrated edge scoring знизить pairwise false-merge rate щонайменше на 30% відносно HDBSCAN при падінні SafeCoverage не більше 3%, без статистично значущого погіршення cross-language RU↔UA slice.

Це вже scientific statement.

---

# 90. Що буде успіхом

### Engineering success

- система обробляє 3 TB;
- працює incrementally;
- lineage / rollback;
- manageable compute.

### Research success

- новий algorithm стабільно перевершує baselines;
- hidden test;
- multi-seed;
- ablation;
- scale test;
- robust slices.

### Scientific success

- mechanism зрозумілий;
- result reproducible;
- prior-art check;
- claim supported;
- paper-ready.

### Innovation success

- algorithmic novelty справді відрізняється від відомих methods;
- можна розглядати patent / publication.

---

# 91. OpenEvolve upstream — актуальні технічні факти

Станом на перевірку у 2026 році upstream OpenEvolve має:

- repository: `algorithmicsuperintelligence/openevolve`;
- Apache-2.0 license;
- Python >=3.10;
- evolutionary program database;
- MAP-Elites quality-diversity mechanisms;
- island-based evolution;
- periodic migration;
- population archive;
- LLM ensemble configuration;
- OpenAI-compatible endpoints;
- cascade evaluation;
- configurable parallel evaluation;
- experiment artifacts;
- checkpoints/resume;
- Docker / Python install paths;
- custom population strategy hooks у сучасній codebase.

Для нашої архітектури це означає, що OpenEvolve можна використовувати як **search engine**, не віддаючи йому ownership над evaluator/governance.

---

# 92. Що ми НЕ копіюємо з OpenEvolve буквально

VoiceID має свої safety requirements.

Тому OpenEvolve upstream evaluator не є автоматично нашим trusted scientific evaluator.

Ми будуємо adapter:

```text
OpenEvolve candidate
→ restricted candidate API
→ sandbox
→ VoiceID trusted evaluator
→ aggregate metrics
→ OpenEvolve selection
```

Hidden-test service може бути взагалі фізично окремим process/container/host.

---

# 93. Майбутній multi-agent layout

```text
Researcher
→ генерує hypothesis / mutation

Critic
→ шукає leakage / trivial tuning / confounders

Executor
→ запускає candidate

Evaluator
→ deterministic, hidden labels

Reviewer
→ аналізує mechanism / repeated results

Methodologist
→ scientific validity

Population Manager
→ islands / archive / lineage

Human PI
→ final promotion / patent / paper decision
```

---

# 94. Practical first OpenEvolve campaign

Не починаємо з усієї системи.

Перший mutable target:

```python
def build_edges(embeddings, ann_neighbors, metadata):
    ...
```

Fixed:

```text
ReDimNet2 snapshot
ANN supergraph
Leiden implementation
benchmark
metrics
```

Еволюціонуємо тільки:

- adaptive k;
- mutuality;
- edge score;
- pruning;
- local margin.

Після цього окрема campaign:

```python
def should_merge(cluster_a, cluster_b, stats):
    ...
```

Потім:

```python
def assign_outlier(point, candidates, stats):
    ...
```

Це дає чисті ablations.

---

# 95. Чому не NAS спочатку

Neural architecture search дуже дорогий.

Якщо проблема насправді в:

- clustering;
- thresholds;
- language nuisance;
- graph construction;

NAS лише спалить compute.

Тому порядок:

```text
Level A
post-embedding algorithm

→ Level B
training recipe

→ Level C
bounded pooling/head changes

→ only later
architecture/NAS
```

---

# 96. Найважливіша practical advantage OpenEvolve

У нас є ideal property:

> **candidate algorithm можна дешево і об'єктивно перевірити на cached embeddings.**

Це означає:

- тисячі experiments;
- без повторного ReDimNet2 inference;
- deterministic comparison;
- швидкий evolutionary loop;
- хороша reproducibility.

Саме тому VoiceID clustering — дуже хороший домен для algorithm evolution.

---

# 97. Ризики

## Risk 1 — benchmark overfit

Mitigation:

- speaker-disjoint hidden test;
- shadow-dev;
- final blind.

## Risk 2 — evaluator gaming

Mitigation:

- immutable evaluator;
- sandbox;
- outlier penalties;
- coverage.

## Risk 3 — giant false merge

Mitigation:

- asymmetric gates;
- cluster contamination metrics;
- merge witnesses.

## Risk 4 — fragmentation

Mitigation:

- FSR;
- B-cubed recall;
- fragmentation metrics.

## Risk 5 — language splitting

Mitigation:

- same-person RU/UA benchmark;
- prototypes;
- nuisance suppression.

## Risk 6 — new encoder space

Mitigation:

- versioned embeddings;
- parallel shadow index.

## Risk 7 — AI generates opaque spaghetti code

Mitigation:

- bounded functions;
- complexity metric;
- reviewer;
- required explanation/ablation.

---

# 98. Final target architecture

```text
                          ┌────────────────────┐
                          │      USER / PI     │
                          └─────────┬──────────┘
                                    │
                         research direction
                                    │
             ┌──────────────────────┼─────────────────────┐
             │                                            │
        Astra / Scientist                         DGX Spark local AI
             │                                            │
             └──────────────────────┬─────────────────────┘
                                    ▼
                              VoiceEvolve
                                    │
          ┌─────────────────────────┼──────────────────────────┐
          │                         │                          │
     OpenEvolve               autoresearch                Optuna/PBT
 algorithm evolution        model-training loop           numeric search
          │                         │                          │
          └─────────────────────────┼──────────────────────────┘
                                    ▼
                                   Ray
                                    │
                                    ▼
                           4 × RTX 4090 server
                                    │
                                    ▼
                         deterministic evaluator
                                    │
                                    ▼
                       Academic Methodology Scientist
                                    │
                                    ▼
                             Human promotion
                                    │
                   ┌────────────────┴────────────────┐
                   ▼                                 ▼
              Production                         Paper/Patent
```

---

# 99. Джерела / reference projects

## Internal project context

- `VOICE_ID_MASTER_CONTEXT.md`
- historical `VOICE_ID`
- `voice-id-claude`
- `voice-id-copilot`
- `voice-id-eco`
- `voice-id-auto`

## OpenEvolve

- https://github.com/algorithmicsuperintelligence/openevolve
- https://github.com/algorithmicsuperintelligence/openevolve/blob/main/README.md
- https://github.com/algorithmicsuperintelligence/openevolve/blob/main/configs/default_config.yaml
- https://github.com/algorithmicsuperintelligence/openevolve/blob/main/openevolve/evaluator.py
- https://github.com/algorithmicsuperintelligence/openevolve/blob/main/pyproject.toml

## Conceptual inspirations

- AlphaEvolve
- FunSearch
- MAP-Elites
- island-model evolutionary search
- Population Based Training
- Karpathy `autoresearch`
- Ray Tune
- Optuna / ASHA / BOHB

---

# 100. Bottom line

Проєкт складається з двох великих систем.

## A. VoiceID Production Platform

```text
3 TB audio
→ speech segmentation
→ ReDimNet2
→ embeddings
→ ANN
→ sparse speaker graph
→ unknown-K discovery
→ merge/split/outlier
→ stable anonymous clusters
→ identity resolution
```

## B. VoiceEvolve Research Platform

```text
LLM scientist
→ OpenEvolve code evolution
→ candidate algorithms
→ Ray execution
→ deterministic evaluator
→ methodology review
→ champion selection
```

`autoresearch` додається не замість OpenEvolve, а поруч:

```text
OpenEvolve
→ шукає кращу algorithmic logic

autoresearch/PBT
→ шукає кращий training recipe

Optuna
→ шукає кращі числові parameters

Evaluator
→ визначає, що реально краще

Methodology Scientist
→ визначає, що науково доведено
```

Ціль — не «натюнити HDBSCAN».

Ціль:

> **створити контрольоване середовище, в якому AI може відкривати, перевіряти і відтворювати нові алгоритми large-scale open-world speaker discovery, не жертвуючи false-merge safety та науковою валідністю.**

