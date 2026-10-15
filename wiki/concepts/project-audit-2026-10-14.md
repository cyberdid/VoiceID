---
type: concept
tags: [аудит, архітектура, методологія, масштабування, пріоритети]
sources: [master-context, wespeaker-toolkit, voxwatch-osi-benchmark, nb-voxceleb-real-audio-benchmark, lit-ssl-clustering, lit-2026-10-14-audit-upstream]
updated: 2026-10-14
---

# VoiceID — детальний аудит проєкту станом на 2026-10-14

## Висновок і межі аудиту

**У цьому checkout VoiceID є дослідницькою базою знань та специфікацією майбутньої системи. Реалізованого production pipeline і VoiceEvolve тут немає.** Архітектурний напрям обґрунтований, але готовність до роботи на всьому корпусі, точність і швидкодія не підтверджені власними відтворюваними експериментами.

Досліджено файлову структуру, Git, master specification, основні wiki-сторінки, пілот і протокол, матеріали експорту NotebookLM, первинний огляд моделей, код та збережені outputs Kaggle-ноутбука. Зіставлено критичні таблиці локальних PDF WeSpeaker/VoxWatch з wiki; сторінку WeSpeaker з таблицями 1–3 додатково перевірено в рендері. Перевірено офіційний upstream ReDimNet2/OpenEvolve та документацію FAISS/Ward.

Не перевірено зовнішні historical репозиторії, сервер 4×4090, DGX Spark, фактичний корпус, checkpoint binaries, хмарний NotebookLM. Відсутність реалізації **в цьому репозиторії** не доводить, що її немає в інших місцях. Збережені outputs чужого ноутбука не є власним повторним прогоном.

Далі факти про файли та першоджерела відокремлено від проєктних рекомендацій. **Ризики, оцінки архітектури, наведені контрприклади й запропоновані зміни — висновки агента, якщо не вказано інше.** Це аудит, а не результати ML-бенчмарку.

## 1. Фактичний склад і зрілість

До додавання матеріалів аудиту, без `.git/` та ігнорованого `temp/`, виявлено:

| Об'єкт | Кількість / стан |
| :-- | :-- |
| Файли | 78 |
| Markdown | 68 |
| Wiki | 50 сторінок |
| Моделі / concepts / datasets / sources / experiments | 14 / 15 / 6 / 8 / 2 |
| PDF / notebooks / JSON | 2 / 1 / 4 |
| Реалізація `src/`, `evaluator/`, `voiceevolve/` | відсутня |
| `tests/`, `configs/`, dependency manifest, Dockerfile | відсутні |
| Root README | відсутній |
| Описані власні експерименти | пілот `proposed`, протокол `draft`; датованих результатів немає |

Є один commit у видимій локальній історії: `684ff79`, 2026-08-27. На початку аудиту вже були зміни `CLAUDE.md`, `wiki/index.md`, `wiki/log.md`, `wiki/overview.md` і 24 untracked-файли. До них належали новий master-документ, його wiki-копія, ReDimNet2/OpenEvolve та пакет NotebookLM. Це не дефект алгоритму, але актуальний контекст ще не зафіксований окремим commit і не має відтворюваного release snapshot.

**Оцінка зрілості:** постановка задачі сформована; архітектура описана; протокол потребує уточнень; реалізація й емпіричні докази в цьому checkout відсутні. Відсотки «готовності» тут були б довільними.

## 2. Що саме будується

Система має вирішувати послідовність різних задач:

1. Знайти мовлення й коректні часові інтервали.
2. Виділити сегменти, які справді містять одного мовця.
3. Отримати вектори голосу в одному версіонованому просторі.
4. Знайти кандидатів серед великої кількості векторів.
5. Побудувати глобальні групи без наперед відомого числа людей.
6. Консервативно об'єднувати, розділяти й доповнювати ці групи.
7. Лише за окремими доказами пов'язувати групу з реальною особою.

[[master-context]] правильно розділяє `speaker_cluster_id` та `person_id`, local diarization і global discovery. `UNKNOWN/UNRESOLVED` — корисна частина моделі рішення. Вимога Unknown-K відповідає нерозміченому відкритому світу; oracle Known-K потрібен тільки як діагностичний контроль.

Це сильні принципи. Найважливіша робоча гіпотеза: reciprocal graph, калібровані ребра та перевірка злиття можуть поліпшити компроміс між чистотою й охопленням. Вона ще потребує перевірки на незалежних мовцях та великому пулі кандидатів.

## 3. Підтверджені проблеми найвищого пріоритету

| ID | Проблема | Наслідок | Доказ |
| :-- | :-- | :-- | :-- |
| A01 | Немає executable pipeline/evaluator | не можна перевірити заявлену систему | фактичний склад checkout; layout у master §74 є рекомендованим |
| A02 | Старий і новий контекст дають різні інструкції | агент може реалізувати неактуальний план | 2 ТБ/ResNet/SimAM/P5 проти 3 ТБ/ReDimNet2/Level A |
| A03 | «×9 від самої зміни домену» не випливає з WeSpeaker | хибне пояснення головного ризику | PDF §3.2.1 і §3.2.2 використовують різні training sets |
| A04 | 9,141% помилково приписано SpeechBrain | необґрунтоване зниження пріоритету моделі | PDF Table 2 посилається на [16], ASV-subtools |
| A05 | 512 вимірів і вбудована L2-нормалізація ReDimNet2 не підтверджені | помилки schema, ANN та capacity planning | офіційний рецепт B6 має 192; `forward` не нормалізує вихід |
| A06 | Split-half оголошено гарантованим позитивом | забруднення калібрації багатомовцевими записами | master допускає кількох людей у файлі; стара wiki цього не враховує |
| A07 | 200 пар трактуються як абсолютний якір наднизької помилки | статистично непідтверджений safety gate | замалий і селективний набір; розрахунок у §7 |
| A08 | RAM-бюджет називає вектори «індексом» і пропускає граф | ризик перевищити RAM/VRAM | таблиці пілоту; окремий розрахунок у §9 |

Пріоритет означає порядок усунення перед повним запуском; це не список знайдених production crashes.

## 4. Узгодженість бази знань

### 4.1 Три master-копії зараз однакові, але синхронізація не автоматизована

`docs/VOICE_ID_OPENEVOLVE_MASTER.md`, `wiki/master-context.md` і `notebooklm/09_VoiceID_OpenEvolve_Master_Technical_Context.md` мають по 3263 рядки й однаковий SHA256:

`4fc91df522c6699bd81fed96f266f88713f4454e1905f0a5b50dfad83ed9df39`.

**Поточне розходження — не між цими трьома копіями.** Воно між новим master та старими тематичними сторінками/досьє. Автоматичного механізму експорту й перевірки copy hashes у репозиторії немає. Рекомендація: зберегти raw-документ незмінним, позначити канонічну редакцію й генерувати похідні копії з manifest/hash.

### 4.2 Суттєві конфлікти

| Питання | Новий master / overview | Старі сторінки / NotebookLM |
| :-- | :-- | :-- |
| Корпус | ~3 ТБ heterogeneous RU/UA/Surzhyk | ~2 ТБ Telegram, здебільшого RU |
| Основний encoder | ReDimNet2 B6/B3 | SimAM + ERes2NetV2; місцями ResNet + ECAPA |
| Мінімум сегмента | 1,5–2 с, бажано 4–8 с | фільтр ≥3 с, вікна 3–5 с |
| Перший R&D цикл | backend на frozen embeddings | ранній pseudo-label training |
| Кількість мовців | Unknown-K обов'язково | k-means 50k → AHC до заданого числа |
| Raw evidence | immutable | `open-questions` дозволяє не зберігати після embedding extraction |
| Аудіофронтенд | stream decode, без persistent WAV на весь корпус | попередній PCM → tar |
| Контроль моделей | усі кандидати перевіряються downstream | стара таблиця жорстко понижує/відкладає кілька сімейств |

Частину відмінностей можна пояснити історичними етапами. Проблема в тому, що старі сторінки не позначені як historical/superseded й досі дають чинні рекомендації. Після узгодження треба оновити тематичні exports; саме додавання ще одного master-досьє не усуває протилежних інструкцій у RAG.

### 4.3 Структурний lint

Автоматична перевірка wiki, без code fences і з урахуванням Obsidian basename resolution, не виявила нерозв'язаних wikilinks, сторінок без вхідних посилань або сторінок поза каталогом. YAML існуючих frontmatter парситься. Це позитивна властивість структури, **а не перевірка правдивості тексту**.

Водночас:

- `wiki/master-context.md` не має обов'язкового frontmatter.
- 46 із 50 сторінок мають `updated: 2026-08-27`, три — 2026-10-06, одна — без frontmatter.
- Каталог указує 47 сторінок, хоча на момент початку аудиту їх 50; заголовок моделей указує 12 при 14 model-сторінках.
- `open-questions` та журнал містять застарілі описи статусів/лічильників; останній запис синхронізації журналу не відповідає його стандартному формату.
- Не всі сторінки мають `sources`; для source-сторінок роль походження частково виконує `raw`, тому це слід оцінювати за типом сторінки.

### 4.4 Посилення впевненості під час синтезу

[[sources/lit-ssl-clustering]] чесно зазначає: числа ~0,99–1,01% отримані з search snippets, конкретна атрибуція не встановлена. [[concepts/pseudo-label-training]] використовує їх як «робочий стандарт» і з'єднує з DINO 2,627% з іншого джерела в єдину траєкторію. Таку послідовність експериментів не можна вважати підтвердженою без перевірки однакових encoder/data/scoring.

У `notebooklm/08_Research_Log_Literature_and_Open_Questions.md` невідоме питання про speaker IDs Golos перетворене на встановлену відсутність цих IDs; source-сторінка цього не встановлює. У `00_Project_Charter...` додається цільовий орієнтир EER 3–7%, хоча вимірів власного корпусу немає.

**Рекомендація:** експорти мають переносити статус доказу разом із твердженням: published, reproduced, hypothesis, unverified. Повторення однієї тези у wiki, NotebookLM і AI report не є трьома незалежними джерелами.

## 5. Помилки інтерпретації наукових джерел

### 5.1 CNCeleb: складність іншого benchmark ≠ чиста оцінка domain transfer

[[sources/wespeaker-toolkit]], [[datasets/cnceleb]], [[concepts/domain-mismatch]] та [[models/comparison]] трактують 0,723% → 6,492% як погіршення тієї самої системи від зміни домену. Локальний PDF говорить:

- VoxCeleb recipe навчається на VoxCeleb2-dev, 5994 speakers.
- CNCeleb recipe навчається на CNCeleb2 + CNCeleb1-dev, 1996 + 797 speakers.
- У CNCeleb інший enrollment protocol, зокрема multi-session averaging.

Числа таблиць правильні; **каузальне пояснення різниці неправильне**. Для вимірювання domain transfer потрібно зафіксувати checkpoint і scoring та змінювати test domain за описаним протоколом. CNCeleb доводить, що benchmark може бути значно важчим, але не встановлює множник деградації нашого корпусу.

### 5.2 Неправильна атрибуція baseline

У Table 2 PDF `ResNet34 [16]` має 9,141%. Reference [16] — *Asv-subtools: Open source toolkit for automatic speaker verification*, не SpeechBrain. Порівняння «SpeechBrain 9,141 vs WeSpeaker 6,492» не підтверджене цією статтею. Уточнювати треба і source-сторінку, і похідні модельні сторінки та досьє.

### 5.3 Рекомендації, що перевищують силу доказу

- Лідерство на CNCeleb не доводить лідерство на RU/UA Telegram. Це stress-test feature, а не заміна цільового eval.
- Менша кількість параметрів не дає автоматично пропорційної швидкодії.
- Поліпшення fusion/QMF у VoxWatch не гарантує той самий відсоток для іншої пари encoder і graph clustering.
- Заява, що більший corpus усуває overfitting, не враховує дублікатів, нерівномірності мовців і псевдоміток.
- Український експеримент у первинному огляді має лише опис (~1,5 год), без manifest, trial list, конфігурації та числової таблиці; це qualitative evidence, не відтворений ranking.
- `baseline` у NotebookLM названо `Ground Truth`. Вихід моделі не є істинною міткою.

## 6. Архітектура production: сильні рішення й відсутні контракти

### 6.1 Дані та виконання

Розподіл на immutable raw, versioned derived evidence і decision state з lineage — правильна основа. Stream decode економить місце. Окремий worker на кожну GPU природно розподіляє незалежний inference.

До реалізації потрібні чіткі контракти:

- `recording_id`, `segment_id` і idempotency key повинні однозначно залежати від source/version, а не тільки path.
- Один worker може втратити lease після запису vectors, але до commit у metadata. Повтор не має дублювати сегмент чи залишати недоступний shard.
- Потрібні atomic shard publication, checksums, offset validation, retries, quarantine та recovery після crash.
- Декодування, GPU prefetch і writer мають bounded queues та backpressure.
- Хеш файлу й хеш вмісту — різні: exact PCM hash не знаходить усі lossy re-encodes. Near-duplicate detection має бути окремим шаром.
- Дедуплікація обчислень не повинна знищувати provenance кожного forwarded copy.

Файли корпусу й inventory відсутні в checkout. Припущення «CPU/I/O — головне вузьке місце» треба вимірювати, а не вважати доведеним.

### 6.2 VAD і single-speaker gate

Silero VAD знаходить мовлення, але не гарантує одного мовця, відсутності overlap чи синтезованого голосу. Cheap homogeneity cascade економить ресурси тільки тоді, коли виміряно його recall для зміни мовця. False-negative цього gate може зіпсувати embedding і всі подальші decisions.

Особливі випадки: turn довший за вікно, дуже короткий другий голос, перекриття, голос із телевізора, stereo з різними співрозмовниками, музика з вокалом. Сліпе усереднення каналів може змішати людей або погасити сигнал за протифази.

VAD-розриви не слід видаляти без збереження відображення до original timestamps. Окремо визначити wall-clock duration, voiced duration, padding і мінімальну кількість speech samples. Політика 4–8 с має бути стартовим конфігом, а не універсальним законом.

### 6.3 Не кожна local speaker label є надійним cannot-link

Діаризація може помилково розділити одного мовця. Тому «різні local speaker IDs» — не завжди hard cannot-link. Визначити окремо trusted constraints, soft constraints і conflicts; записувати походження й confidence. Чат 1:1 також може містити переслані записи третіх людей.

### 6.4 State, стабільні ID та identity

Master передбачає merge/split lineage, але не описує семантику stable IDs при split і merge. Необхідно визначити aliases, survivor policy, retire events, membership versions, atomic prototype/index switch, rollback і аудит ручних overrides.

Потрібні сценарії приходу late data, одночасних attachments, order sensitivity, повторного ingestion й зміни encoder. «Щотижневий merge/split» — cadence, не алгоритм узгодження стану.

Окремий identity layer є перевагою. Trusted reference має зберігати джерело, дату, перевірку speaker homogeneity та можливість відкликання. Для practical use потрібен мінімальний human-review інтерфейс: A/B listening, fragment provenance, причина злиття, competitors, confirm/reject/split. Його реалізації тут немає.

## 7. Калібрація й safety: найважливіші методологічні виправлення

### 7.1 Split-half не є гарантованою speaker label

У багатомовцевому файлі перша і друга половина можуть належати різним людям. Навіть для перевіреного single-speaker запису половини мають спільний канал і сесію та не оцінюють cross-session robustness. Використовувати їх як consistency check і допоміжні positives, з явним status; не підміняти ними незалежну ground truth.

### 7.2 Випадкові negatives мають неправильний prior для ANN

При нерівномірній частоті мовців імовірність same-speaker у двох незалежно вибраних файлах:

\[
P(same)=\sum_s p_s^2.
\]

Вона дорівнює `1/K` тільки за рівномірного розподілу. Часті диктори й дублікатні ланцюги можуть істотно підвищувати забруднення.

Ще важливіше: ANN відбирає схожих кандидатів. Розподіл cosine серед випадкових пар набагато легший за impostors серед top-k. Калібратор потрібно навчати/оцінювати на candidates тієї retrieval policy, де він застосовується, з nearest impostors, quality strata і cross-session positives.

`P(same speaker | x)` після навчання на штучно збалансованих парах не є автоматично ймовірністю в production. Потрібні перевірка calibration/prior shift або чесна назва «score».

### 7.3 Що доводять 200 пар

Якщо 200 **незалежних і репрезентативних випробувань відповідного рішення** не дали жодної помилки, одностороння 95% верхня binomial bound дорівнює:

\[
1-0.05^{1/200}\approx0.014867=1.49\%.
\]

Для верхньої межі `10^-4` за нуля помилок потрібно щонайменше **29 956** таких незалежних випробувань. Це ілюстрація sample complexity, не вимога вручну розмітити стільки будь-яких пар. Для FMR випробування мають відображати прийняті same-speaker рішення; для FPR — негативні trials. Це різні знаменники.

Пари біля порога корисні для active learning, але селективні. Повторні сегменти тих самих людей також не незалежні. Тому 200 boundary pairs не дають популяційного safety certificate, навіть якщо помилок нуль.

### 7.4 Eval-набір не можна одночасно використовувати для порога й доказу

Пілот описує 200 пар як «недоторканий» якір і водночас використовує їх у калібрації. Потрібні окремі calibration-dev та frozen evaluation. Із фрази «ці ж пари — для QMF» слід однозначно зрозуміти, чи це weak pairs, чи manual anchors.

Повторний вибір candidate за hidden-test outcomes робить hidden test частиною selection. Потрібні обмежена кількість queries, окремий promotion holdout і фінальний blind evaluation. Необхідні speaker/session/content/duplicate checks, а не лише неперетин raw file paths.

### 7.5 Згода encoder не є незалежною істинною міткою

Архітектурно різні моделі можуть вивчити той самий channel bias й помилитися на тих самих impostors. Згода — feature/triage signal; її цінність потрібно перевірити conditional precision на розмічених парах. Вона не робить помилку «квадратично меншою» без незалежності.

## 8. Метрики: правильний напрям, недостатньо визначень

Master визначає `FMR = FP/(TP+FP)`. Це **1 − pairwise precision**, а не класичний `FPR = FP/(FP+TN)`. Формула коректна як false-discovery fraction для co-clustered pairs, але назву й denominator потрібно зафіксувати. `FSR = FN/(TP+FN)` відповідає 1 − pairwise recall.

Потрібні окремі показники: edge error, merge-event error, final pair contamination, частка забруднених clusters, sample/duration-weighted impurity, число постраждалих speakers. Merge двох чистих груп двох різних мовців, по 500 записів у кожній, створює **250 000** хибних co-clustered пар — одна помилка рішення має нелінійні наслідки.

### SafeCoverage поки не є executable metric

Визначення «coverage серед candidates, які проходять safety» допускає зміну denominator. Необхідно зафіксувати eligible population незалежно від candidate і окремо рахувати:

- частку прийнятих сегментів та voiced duration;
- частку outlier/unresolved/new singleton;
- multi-recording coverage та cross-session coverage;
- overall і worst-slice safety bounds.

Інакше «кожен segment — singleton» може дати бездоганну purity та високу nominal coverage без корисного discovery. Часті clean speakers можуть також приховати contamination рідкісних людей.

### Noise handling і статистика

Якщо HDBSCAN повертає `-1`, не можна мовчки вважати всі noise points одним speaker cluster. Водночас виключення всього noise з метрик робить denominator залежним від алгоритму. Зафіксувати singleton/unresolved semantics і звітувати abstention окремо.

Pairwise trials залежні через спільних speakers/sessions; confidence intervals мають враховувати ці групи. Для zero errors потрібні upper bounds, а не claim «0% назавжди». Для повторних evolution campaigns потрібен облік selection bias. Exact pairwise counts можна отримувати з contingency counts, не матеріалізуючи `N²` pairs.

У §89 master «падіння SafeCoverage не більше 3%» слід уточнити як relative percent чи percentage points. «Без статистично значущого погіршення» не доводить non-inferiority: потрібні заздалегідь визначені margin і CI. За baseline FMR=0 relative reduction взагалі не визначене.

## 9. Масштаб і обчислювальний бюджет

### 9.1 Планувати треба за числом сегментів

За умовного decimal 3 TB без container overhead отримуємо близько 52 083 год при 128 kbit/s або 208 333 год при 32 kbit/s. Якщо все є мовленням і робиться один неперекривний segment кожні 6 с, це приблизно 31,25 млн або 125 млн vectors. Реальний VAD зменшує обсяг; overlap збільшує; короткі turns змінюють distribution.

Ці числа — **сценарний розрахунок**, не inventory. Старі таблиці рахують переважно один embedding на ~15-секундне повідомлення, тому їх не можна перенести на 4–8-секундну сегментацію.

### 9.2 Vector store, ANN index і speaker graph — три різні витрати

Для FP16-векторів: `N × d × 2 bytes`. Для directed candidate graph з int32 neighbor IDs, float32 scores та int64 row offsets: `N × k × 8 + (N+1) × 8 bytes`.

| N | Vectors d=192, GB | Vectors d=512, GB | Graph k=100, GB | Graph k=200, GB |
| --: | --: | --: | --: | --: |
| 1 млн | 0,384 | 1,024 | 0,808 | 1,608 |
| 10 млн | 3,84 | 10,24 | 8,08 | 16,08 |
| 30 млн | 11,52 | 30,72 | 24,24 | 48,24 |
| 100 млн | 38,40 | 102,40 | 80,80 | 160,80 |

Decimal GB, без ANN index, edge features, temporary copies, graph library overhead та metadata. d=512 — сценарій, а не підтверджена розмірність ReDimNet2. Вже 30 млн ×192 plus k=200 graph — **59,76 GB** до додаткових структур.

FAISS `IndexHNSWFlat` зберігає FP32 vectors та власні graph structures; розмір FP16 shard не є його RAM. Див. [[sources/lit-2026-10-14-audit-upstream]]. Потрібні окремі measurements peak RAM/VRAM, build time і serialization size. Не всі варіанти індексів доступні в однаковому вигляді на CPU/GPU.

### 9.3 Чотири GPU не означають єдину 96-GB область

У специфікації вказано 4×4090. Для незалежних workers пам'ять і batch обмежені конкретною картою. DDP реплікує модель; шардинг індексу чи навчального стану потребує окремої реалізації. Практичний multi-GPU бюджет залежить від PCIe topology, transfer overhead і CPU/RAM.

### 9.4 Поточні оцінки throughput не виміряні

Заяви «хвилини на пілот», «кілька годин на все», «індекс уміщується за будь-якого сценарію» та «навчання доба на ітерацію» в старих сторінках не мають власного benchmark artifact. Із розміру corpus та моделі неможливо встановити їхню істинність.

Заміряти окремо files/s, decode audio-hours/hour, voiced seconds/s, segments/s, GPU utilization, queue wait, writer throughput, ANN build/query, graph build і clustering. Загальний час визначає pipeline bottleneck. Якщо effective speed `R` — audio-hours за wall-hour, backfill time `H/R`, після врахування diarization, retries та writing.

Перед повним backfill: 1k/10k/100k файлів → 1M vectors → більші scale pools. Точки 1M/10M/30M у master — майбутні тести, не пройдені етапи.

## 10. Graph clustering: що перевіряти окремо

### ANN recall є частиною якості

Якщо retrieval пропустив cross-language neighbor, кращий edge scorer не відновить відсутній edge. Потрібні exact-search control на вибірці, recall@k по hard slices, same-speaker graph connectivity, відсутність self-neighbors/duplicates і exact re-ranking відібраних кандидатів.

Потрібно розрізняти k кандидата, k graph, index build/search параметри й coverage evidence. Supergraph kmax=100–200 задає стелю пошуку. Він економить compute, але не доводить глобальної оптимальності й не дозволяє adaptive-k вийти за цю стелю.

### Reciprocal edges допомагають, але не гарантують чистоту

Два impostors можуть бути взаємними сусідами. Connected components транзитивно з'єднає `A–B–C`, навіть якщо `A` і `C` різні. Leiden/CPM знаходить communities, а не істинні people. Потрібні hard bridge cases, intra-cluster purity audit і independent merge witnesses.

Centroid, medoid, sharedNN і top cross edges не є незалежними доказами самі по собі. Десять вікон одного recording — не десять незалежних witnesses. Підтримку слід групувати за original recording/session/duplicate lineage.

### Ward і cosine AHC треба розвести

Master перелічує Sparse Ward/AHC, а стара wiki рекомендує average-linkage cosine. Це різні objectives. Ward використовує variance/euclidean; cosine average-linkage — інше правило. Для unit vectors `||u−v||² = 2(1−cos(u,v))`, але така еквівалентність pair ranking не робить кластерні linkage objectives тотожними.

Sparse connectivity не усуває всі можливі overhead чи небезпечні merges. Фіксувати implementation, stopping criterion, memory та поведінку disconnected components. Внутрішній exact AHC для «великих communities» теж може стати `N²` bottleneck — потрібен size cap.

### Multi-prototype та language features потребують контролю

Мова й codec корисні як nuisance/quality features, але можуть створити два clusters для однієї bilingual людини. Потрібні RU↔UA same-person/same-session та cross-session cases, різні recording conditions і missing/uncertain language metadata. Кількість prototypes впливає на search multiplicity, latency та ймовірність false attachment.

## 11. ReDimNet2: підтримати вибір, уточнити конкретну систему

Master чесно називає ReDimNet2 baseline, не абсолютним winner. Це слід зберегти. При цьому локальні overview/model pages сильніше стверджують «оптимальність для 4×4090», ніж дозволяють власні дані.

Перевірений upstream B6 LM recipe має `embed_dim=192`; `ReDimNet2Wrap.forward` не робить явного L2-normalize. Називати B6 «максимальною роздільною здатністю embeddings» некоректно: розмір архітектури і розмірність вихідного вектора — різні речі. Конкретні checkpoint binaries не перевірені; для їхньої фактичної dimension authoritative є `model_config` і output tensor.

Зафіксувати: upstream commit, checkpoint URI/hash, dataset/train_type, waveform units, required sample rate, mel/frontend config, voiced-window policy, padding, dtype, device, external normalization, output dimension. Зберігати raw embedding norm **до** L2 для QMF; після L2 вона майже завжди 1 і втрачає цей сигнал.

B3 і B6 — різні vector spaces. Рейтинг graph candidates на B3 не гарантує того самого рейтингу на B6. Кожен candidate/champion прив'язувати до embedding snapshot і повторно проходити gates при зміні encoder.

Стартовий bake-off можна обмежити ReDimNet2 B3/B6, ERes2NetV2 та SimAM/CAM++ за compute budget. Оцінювати downstream safety/coverage і measured cost; verification EER залишити sanity check. Підтримку tensor format і checkpoint loading перевірити до масштабного extraction.

## 12. OpenEvolve/VoiceEvolve: корисний другий етап

Перевага master: спочатку frozen embeddings і trusted evaluator, потім bounded mutation; навчання encoder відкладається. Так можна відокремити внесок graph logic від зміни neural representation.

### 12.1 Search engine не реалізує safety policy автоматично

Перевірений OpenEvolve відрізняє scalar fitness від MAP-Elites feature dimensions. Тому описаний у master constraint-first Pareto selection потребує окремої реалізації adapter/archive. Якщо `combined_score` не задано, агрегування метрик може змішати показники різного напряму та одиниць. Unsafe candidate має відхилятися незалежно від search score.

Optional `use_llm_feedback` змінює score в upstream evaluator; для вимірювання біометричної якості залишити його вимкненим. Critic/reviewer feedback зберігати окремо від фактичних метрик. Деталі коду й конфіга — [[sources/lit-2026-10-14-audit-upstream]].

### 12.2 Ізоляція має бути реальною

AST checks, restricted API, read-only file чи Python timeout не утворюють sandbox. До запуску generated code визначити process/container boundary, allowlisted inputs, відсутність network/secrets, RAM/CPU/wall-time limits, output quotas й termination усіх дочірніх процесів. Trusted evaluator рахує metrics з candidate output; candidate не має доступу до labels, evaluator internals чи deployment pointer.

Upstream config не реалізує зазначені memory/CPU limits. Це інженерна відповідальність VoiceEvolve. Потенційна проблема описує майбутню інтеграцію; у цьому checkout немає runnable candidate executor, який був би перевірений на exploitation.

### 12.3 Cached graph і mutable transforms

Master дозволяє `embedding transform` у Level A і водночас фіксує ANN supergraph для першої кампанії. Transform може змінити nearest neighbors. Треба або заморозити transforms у цій кампанії, або розглядати cached graph тільки як candidate pool з виміряним recall та окремим retrieval rebuild experiment.

### 12.4 Контракт першої кампанії

Достатньо одного target `build_edges`: inputs із визначеними shape/dtype, output із межами IDs, finite scores, no self-loops/duplicates, max degree, symmetry policy та deterministic behavior. Fixed Leiden і metric implementations. Спочатку static baseline + numeric tuning з рівним search budget, потім code evolution.

Корисні baseline comparisons: untuned baseline; budget-matched Optuna/random search; OpenEvolve candidate. Ablations прибирають adaptive-k, mutuality, quality feature, merge/split по одному. Наявність складнішого коду сама по собі не є науковою новизною.

Ray, multi-agent governance, local LLM і багатовимірні archives додають infrastructure cost. Їх вводити за виміряною потребою, після мінімального evaluator. 90-денний план не можна оцінити як календарну обіцянку без команди, корпусу, annotation budget та benchmark readiness.

## 13. Kaggle-ноутбук: реальні дефекти й правильна роль

`docs/benchmarking-using-voxceleb-for-real-audio.ipynb` містить 11 cells: 10 code, 1 markdown. AST parse усіх code cells успішний; це не доказ run-all. Перша порожня cell містить stale output із `NameError: DEVICE is not defined`.

Модель `WaveFakeModel`/`UltimateWaveFakeModel` — Wav2Vec2 + binary classifier human/spoof. Це **інша задача**, ніж speaker discovery. Її 98,33% accuracy / 1,67% validation EER не оцінюють speaker identity.

### Підтверджені кодові й методологічні проблеми

1. **Invalid format:** у cell 2 `f"{prob_v5:.4f:<10}"` дає `ValueError`. Відтворено чистим Python через `format(0.337, '.4f:<10')`. Наступні cells уже використовують format-then-pad, тобто є локальний правильний pattern.
2. **`except: pass`:** cell 2 приховує навіть помилку друку. Predict counters збільшуються до print, тому 400 processed у output не означає відсутність exceptions.
3. **Подвійний процент:** `(x/n)*100` і `:.2%` породжують 8425%, 9600%, 8000%, 9283,33%. Правильні fractions відповідно 337/400, 384/400, 28/35, 557/600.
4. **Silent zero substitution:** dataset `__getitem__` повертає zeros із первинною label при будь-якому load error. Пошкоджене аудіо стає навчальним прикладом без окремого обліку.
5. **Source-confounded split:** `human_paths = wild[:450] + vox[:450]`; train бере перші 720, val — решту 180. Отже real train містить 450 wild + 270 Vox, а real val — тільки 180 Vox. Це не рівномірний multi-domain split.
6. **Selection/reporting reuse:** epoch, threshold і фінальні metrics обираються/обчислюються на тих самих 360 validation samples. Незалежного test для цього claim немає.
7. **Potential content leakage:** ElevenLabs названо cross-format corpus; split за окремими paths не перевіряє overlap content, speaker/voice чи encoding variants. Фактичний overlap без manifest не встановлений.
8. **Selective seeds:** задано Python `random.seed`, але немає повного torch/NumPy/CUDA seed і dataset manifest; `os.listdir` порядок також не зафіксований.
9. **Narrow channel simulation:** 16→8→16 kHz — band limitation, не повна GSM/AMR/Opus simulation.
10. **Crop/padding bias:** перші 4 с та zero padding без speech-aware chunking можуть акцентувати silence/channel artifacts.
11. **Поріг 0,85 обрано після перегляду того самого fake pool:** наступний report уже не незалежний blind результат цього threshold selection.
12. **Checkpoint provenance:** pretrained paths прив'язані до Kaggle, hashes/dependency versions відсутні; scope prior training невідомий.

Збережені outputs: V7 при 0,2611 виявляє 28/35 spoof; при 0,85 — 30/35; окремий real pool при 0,85 має 43/600 false alarms. Це не confusion matrix одного репрезентативного corpus і не доказ production reliability. Для real pool при 0,2611 у wiki стоїть ~0%, але відповідного зовнішнього оцінювання в ноутбуці не знайдено; це число слід позначити невідомим.

Ноутбук корисний як приклад debugging, source shifts і train/val mistakes. Переносити його architecture або EER claim у SpeakerID без зміни задачі не можна.

## 14. Pseudo-label training: пізніше, після незалежної перевірки

Цикл може адаптувати encoder, але clean fragmentation теж шкодить: fragments однієї людини стають різними класами й AAM-loss відштовхує їх. False merges зближують різних людей; channel/language shortcuts можуть закріпитися. Великий обсяг без labels не усуває цих ризиків.

Перед запуском потрібні verified clean cores, session-aware sampling, обмеження frequent-speaker dominance, duplicate suppression, confidence weighting і незалежний cross-session/cross-language контроль. Weak/unresolved segments не повинні автоматично ставати hard labels.

Split-half consistency може рости навіть тоді, коли міжмовна recall погіршується. Кількість clusters також може стабілізуватись на неправильному рішенні. Зупинятися за зовнішнім frozen benchmark і safety bounds; тримати rollback на старий encoder/index.

Не переносити 80-bin AAM-рецепт WeSpeaker автоматично у ReDimNet2 checkpoint, у якого перевірений frontend має інший конфіг. Тренувальний recipe, frontend, loss і pretrained weights треба узгоджувати як систему.

## 15. Мінімальний практичний шлях

| Етап | Конкретний результат | Критерій переходу |
| :-- | :-- | :-- |
| 0. Узгодити контекст | чинні/historical сторінки, source corrections, snapshot у Git | немає двозначного baseline/корпусу/retention policy |
| 1. Інвентаризація | manifest + duration/codec/quality/duplicate distribution | відомі H, N-files, N-segments і availability data |
| 2. Eval foundation | trusted metrics, labeled protocol, dev/test separation | вручну перевірені metric edge cases; bounds і denominators frozen |
| 3. Encoder smoke | один pinned encoder + frontend, versioned embeddings | shape/norm/timestamp/error checks; measured inference cost |
| 4. Повний малий pipeline | exact control + ANN → conservative graph → clusters | quality і coverage звіт; provenance та review samples |
| 5. Backend comparison | HDBSCAN, AHC, graph families на тих самих inputs | curves + worst slices + ресурсні витрати |
| 6. Scale/incremental | memory budget, retries, rebuild/shadow, merge/split events | recovery/rollback, tail quality під distractors |
| 7. OpenEvolve v1 | один bounded target, sandbox, adapter, budget-matched search | independent improvement або чесний negative result |
| 8. Encoder adaptation | pseudo-label training/new snapshot | повторно пройдені downstream gates |

Перший milestone: **відтворюваний запуск, що бере реальні audio inputs, повертає anonymous clusters і показує quality/coverage/cost із перевірюваними доказами**. OpenEvolve не є передумовою цього milestone.

## 16. Що необхідно з'ясувати перед масштабним запуском

- Де фактичний корпус, Gold-Small і historical code? Що означає 2→3 ТБ: нова редакція чи розширення даних?
- Скільки speech hours, speaker-homogeneous segments, multi-speaker recordings і duplicates?
- Які CPU/RAM/storage/PCIe resources реально доступні?
- Яка числова межа FMR, які worst-slice requirements і допустима частка unresolved?
- Які annotation budget, trusted sessions та bilingual reference speakers?
- Який exact checkpoint і allowed metadata contract?
- Як semantic speaker ID змінюється при merge/split, хто перевіряє identity evidence?
- Як часто дозволено query promotion holdout і хто контролює release?

Ці відповіді потрібні для реалізації та capacity planning; їхня відсутність не заважала виконати документальний аудит.

## 17. Проведені перевірки та зміни

Перевірено file inventory/Git, counts і copy SHA256, YAML frontmatter, Obsidian wikilinks/catalog coverage, AST 10 code cells, minimal format reproduction, output counts, binomial bound, vector/graph memory arithmetic, primary PDF tables/refs і upstream source/config. GPU inference, training, ANN build і production tests не запускались: даних/реалізації в checkout немає.

Raw `docs/` залишено незмінним. Додано цей аудит і upstream source note; доповнено каталог, open questions та append-only log. Старі твердження масово не переписувались: вони залишаються простежуваними й позначені як проблеми цього аудиту. Зовнішній NotebookLM не змінювався.

## Джерела

[[master-context]] · [[overview]] · [[sources/wespeaker-toolkit]] · [[sources/voxwatch-osi-benchmark]] · [[sources/nb-voxceleb-real-audio-benchmark]] · [[sources/pretrained-models-review]] · [[sources/lit-ssl-clustering]] · [[sources/lit-model-checkpoints]] · [[sources/lit-2026-10-14-audit-upstream]] · [[models/comparison]] · [[models/redimnet2]] · [[concepts/speaker-clustering]] · [[concepts/pseudo-label-training]] · [[concepts/score-normalization]] · [[datasets/telegram-corpus]] · [[datasets/cnceleb]] · [[experiments/protocol]] · [[experiments/clustering-pilot]] · [[open-questions]]
