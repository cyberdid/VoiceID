---
type: overview
tags: [журнал, хронологія]
updated: 2026-08-27
---

# Log

Хронологічний append-only журнал. Формат запису — `CLAUDE.md`.
Останні 5: `grep "^## \[" wiki/log.md | tail -5`

## [2026-08-27] setup | Розгорнуто wiki за патерном LLM Wiki

Проєкт переведено на патерн Karpathy LLM Wiki
(<https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f>).
Створено `CLAUDE.md` (схема), `wiki/` з розділами `sources/ models/ concepts/ datasets/ experiments/`,
[[index]], [[log]], [[overview]], [[open-questions]].
`docs/` закріплено як шар незмінних raw sources.

## [2026-08-27] ingest | Пре-треновані моделі ідентифікації за голосом

Внутрішній огляд 12 відкритих моделей + таблиця параметрів + Colab-приклади.
Створено: [[sources/pretrained-models-review]], 9 сторінок моделей, [[models/comparison]],
[[concepts/speaker-embedding]], [[concepts/audio-preprocessing]].
Ключове: єдине джерело з результатом на **україномовних даних** (~1,5 год) —
ResNet > ECAPA-TDNN > Pyannote. Суперечить рейтингу за заявленим EER.

## [2026-08-27] ingest | Ідентифікація голосу: параметри та методології

Оглядовий звіт ~90 КБ, 90 посилань: акустика голосу, три традиційні криміналістичні методи,
AI-методи (ASR, ECAPA-TDNN, x-/i-вектори, ResNet), вплив параметрів аудіофайлу, спектральні
ознаки, Python-бібліотеки.
Створено: [[sources/voice-id-parameters-methods]], [[concepts/acoustic-features]],
[[concepts/task-types]], [[concepts/domain-mismatch]].
Оновлено: [[models/ecapa-tdnn]], [[models/resnet]], [[concepts/audio-preprocessing]].
Ключове: фізична основа ідентифікації — форманти F1–F3 як відбиток анатомії голосового
тракту. Прогалина джерела: open-set постановка не згадана взагалі.

## [2026-08-27] ingest | VoxWatch (arXiv:2307.00169)

Стаття AWS AI Labs: перший публічний open-set benchmark на VoxCeleb.
Створено: [[sources/voxwatch-osi-benchmark]], [[concepts/open-set-identification]],
[[concepts/eer-and-metrics]], [[concepts/score-normalization]], [[datasets/voxceleb]],
[[experiments/protocol]].
Оновлено: [[models/resnet]] (розведено 4 різні «ResNet»), [[models/wavlm]] (0,52% як
фронтенд проти 4,04% як готова голова), [[models/ecapa-tdnn]], [[models/comparison]].
Ключове: **FAR@FRR=5% росте 2,4% → 16,1% при watchlist 50 → 500**. Fusion дає −44% FAR,
калібрація −21%. **AS-Norm покращує лише слабкі моделі** — суперечить попередній літературі.

## [2026-08-27] ingest | Kaggle-ноутбук benchmarking-using-voxceleb-for-real-audio

⚠️ Задача інша — anti-spoofing (справжній голос vs ElevenLabs), не ідентифікація мовця.
Заінджещено як джерело інженерних практик і методологічних застережень.
Створено: [[sources/nb-voxceleb-real-audio-benchmark]], [[concepts/fine-tuning]],
[[concepts/anti-spoofing]], [[datasets/in-the-wild]].
Оновлено: [[concepts/domain-mismatch]], [[concepts/eer-and-metrics]],
[[concepts/audio-preprocessing]], [[models/wav2vec2]].
Ключове: розморожування 4 верхніх шарів замість 1 дало EER 10,00% → 1,67%.
Але на зовнішньому наборі — 20% пропусків: **перенавчання на домен**.
Знайдено 8 дефектів методології (баг форматування %% дав «8425,00%»; val використано
і для early stopping, і для вибору порога, і для звіту; тихе ковтання помилок читання).

## [2026-08-27] lint | Перший прохід після 4 ingest'ів

Зафіксовано 4 суперечності (рейтинг моделей, WavLM 4,04% vs 0,52%, AS-Norm, «ResNet» ×4)
і 19 відкритих питань у [[open-questions]] — з них 3 блокери, усі потребують відповіді людини.
Сформульовано робочу тезу з 6 тверджень у [[overview]].
Головний висновок: 🔴 **блокер проєкту — відсутність україномовних даних з мітками
дикторів**; етап 0 (санітарка на VoxCeleb) можна робити вже зараз, не чекаючи на них.

## [2026-08-27] ingest | Wespeaker toolkit (arXiv:2210.17016)

Нове джерело з'явилось у `docs/` під час сесії. Стаття SJTU X-LANCE / Tencent / WeNet
про toolkit для навчання ембедингів мовця.
Створено: [[sources/wespeaker-toolkit]], [[datasets/cnceleb]], [[concepts/loss-functions]].
Оновлено 13 сторінок: [[models/wespeaker]] (переписано, статус → кандидат №1),
[[models/resnet]], [[models/ecapa-tdnn]], [[models/comparison]], [[concepts/domain-mismatch]],
[[concepts/fine-tuning]], [[concepts/score-normalization]], [[concepts/audio-preprocessing]],
[[concepts/task-types]], [[datasets/voxceleb]], [[datasets/ukrainian-data]],
[[experiments/protocol]], [[overview]], [[index]], [[open-questions]].

Ключове:
1. **Перша таблиця з єдиним протоколом** — ResNet34 0,723 / ECAPA 0,728 на Vox1-O,
   але **ResNet попереду на Vox1-H** (1,532 vs 1,721). ✅ Розв'язано суперечність №16.
2. **Ціна зміни домену виміряна:** ResNet34 VoxCeleb 0,723% → CNCeleb **6,492%**, ×9.
3. **Реалізація важить як архітектура:** ResNet34 1,31 → 0,723 (−45%) від рецепту;
   на CNCeleb SpeechBrain 9,141 vs Wespeaker 6,492 (−29%).
   🔁 Шортліст перебудовано: №1 тепер `pyannote/wespeaker-voxceleb-resnet34-LM`.
4. **Multi-session enrollment** (усереднення ембедингів по сесіях) — новий дешевий важіль.
5. ✅ Закрито прогалину №15 (AAM-softmax) → [[concepts/loss-functions]].
6. Нова суперечність №21: AS-Norm корисна в Wespeaker (SV), шкідлива у VoxWatch (OSI).

## [2026-08-27] lint | Другий прохід (після 5 джерел)

Перевірено 36 сторінок: **0 биті посилання, 0 сторінок-сиріт**.
Хаби: [[sources/voxwatch-osi-benchmark]], [[models/resnet]], [[concepts/eer-and-metrics]].
Закрито 2 питання (№15, №16), відкрито 4 нові (№20–23). Разом: 23, з них 3 🔴 блокери.
Робочу тезу в [[overview]] оновлено: пункт 2 отримав точне число (×9), пункт 3
переформульовано з «заявлені EER ненадійні» на «обирати треба чекпойнт і рецепт».

## [2026-08-27] query | Brainstorm: як кластеризувати 2 ТБ Telegram-записів

🔁 **Постановка проєкту суттєво змінилась.** Уточнено вхідні дані: не україномовний
корпус, а **~2 ТБ нерозмічених записів з Telegram, здебільшого російською**.
Перша задача — **кластеризація**, не ідентифікація.

Створено: [[datasets/telegram-corpus]], [[concepts/speaker-clustering]],
[[experiments/clustering-pilot]].
Оновлено: [[overview]] (задача + план переписані), [[concepts/task-types]],
[[datasets/ukrainian-data]] (статус → непрямий доказ), [[open-questions]], [[index]].

Ключове:
1. **Метадані відправника = слабка розмітка в масштабі.** Пари «той самий / різні
   відправники» дають DET-криву на власному домені без ручної роботи.
   ✅ Знімає блокер «немає розмічених даних» (питання 1, 3, 6).
2. **Асиметрія помилок кластеризації**: хибне злиття руйнує кластери назавжди,
   хибне розділення — ні. Правило: краще перерозділити, ніж перезлити → поріг
   у зоні низького FAR, не EER.
3. **Масштабна кластеризація**: FAISS HNSW → mutual-kNN (захист від ланцюжків) →
   Leiden → AHC усередині великих спільнот. Індекс (2–22 ГБ) вміщується в RAM;
   вузьке місце — декодування та I/O, не GPU.
4. **Дві перевірки без розмітки**: split-half self-consistency і згода двох енкодерів.
5. 2 ТБ дрібних файлів → **UIO-шарди** з [[sources/wespeaker-toolkit]].
6. Нові питання 24–29; питання 1, 3, 6 закрито.

## [2026-08-27] query | Уточнення вхідних: без метаданих, 4 × RTX 4090

❌ **Ідею з безкоштовною розміткою знято.** JSON-експорту немає — лише .ogg без прив'язок.
✅ **Залізо:** 4 × RTX 4090. ✅ **Контекст:** дослідження.
❓ Можливо, вже є пораховані ембединги — потребує перевірки.

Створено: [[concepts/pseudo-label-training]].
Оновлено: [[datasets/telegram-corpus]] (секція метаданих переписана),
[[concepts/speaker-clustering]] (калібрація без міток), [[experiments/clustering-pilot]]
(P2 переписано, додано P5, бюджет під 4×4090), [[overview]] (тези 6–7), [[open-questions]], [[index]].

Ключове:
1. **Калібрація без міток можлива**: split-half (позитиви) + випадкові пари (негативи)
   дають наближену DET-криву; ~200 розмічених пар **у зоні невизначеності** — абсолютний якір.
   Зміщення split-half жене поріг у бік перерозділення — безпечний напрямок.
2. 🔁 **Головний важіль змінився.** Не перебір претренованих чекпойнтів, а
   **ітеративне навчання енкодера на псевдомітках** ([[concepts/pseudo-label-training]]).
   4 × 4090 + тисячі годин in-domain аудіо роблять це реальним; це закриває розрив
   доменів буквально, а не обхідним шляхом.
3. **Витяг ембедингів — не вузьке місце** (години на весь корпус). Вузьке місце —
   декодування OPUS та I/O по мільйонах дрібних файлів → tar-шарди (UIO).
4. Нові блокери 30–32: структура тек (чи = чати), шлях до даних, кількість CPU-ядер.

## [2026-08-27] research | Self-supervised мовець і кластеризація в масштабі

Веб-дослідження на замовлення. Закрито найбільшу діру корпусу: жодне з 5 джерел
у `docs/` не покривало ані кластеризацію, ані навчання без міток.

Створено: [[sources/lit-ssl-clustering]], [[datasets/voxblink2]], [[models/3d-speaker]].
Оновлено: [[concepts/pseudo-label-training]], [[concepts/speaker-clustering]],
[[models/comparison]] (шортліст перебудовано), [[overview]] (тези 7–8),
[[open-questions]], [[index]], `CLAUDE.md` (додано тип операції **Research**).

Ключове:
1. **~1,01% EER без жодних міток** (VoxCeleb2, після 2-ї ітерації), ~0,99% з LM
   fine-tuning. Обходить supervised TDNN (1,59%), майже наздоганяє ECAPA (0,87%).
   → цикл з псевдомітками більше не гіпотеза.
2. **Канонічний рецепт**: DINO → k-means(50k) → AHC → ResNet34+AAM, ⟳ ×3, сегменти 2 с.
3. **7 500 кластерів при 5 994 дикторах** — галузь навмисне перекластеризовує на ~25%.
   Емпіричне підтвердження правила «краще перерозділити, ніж перезлити».
4. **Масштаб вирішено**: exact memory-constrained UPGMA перевірено на **4 млн**
   висловлювань; HDBSCAN з ANN не потребує k.
5. 🔁 **Шортліст перебудовано.** Знайдено [[datasets/voxblink2]] (110 тис. дикторів,
   SOTA **0,170%**) і [[models/3d-speaker]] (200 тис. дикторів) — проти 5 994 у
   VoxCeleb2. Для чужого домену широта даних важить більше за архітектуру.
6. Методи проти шуму псевдоміток: DLG-LC, CA-DINO (+22–28% відн.), GCN, batch-scale correction.
7. Нові питання 33–37; блокер №33 — чи є публічні VoxBlink2-чекпойнти.

⚠️ Усе — з анотацій пошуку, PDF не читані. Числа як орієнтир, не як цитата.

## [2026-08-27] research | Перевірка чекпойнтів: VoxBlink2, ModelScope, Wespeaker

Створено: [[sources/lit-model-checkpoints]], [[concepts/short-duration]].
Оновлено: [[models/3d-speaker]], [[datasets/voxblink2]], [[models/wespeaker]],
[[models/comparison]] (шортліст перебудовано вдруге), [[overview]] (теза 8 переписана),
[[experiments/clustering-pilot]], [[open-questions]], [[index]].

Ключове:
1. ✅ **VoxBlink2-чекпойнти є** — офіційно в Wespeaker, у категорії **Multilingual**
   (SimAMResNet34/100). HF: `Wespeaker/wespeaker-voxlink2-samresnet34_ft`
   ⚠️ опечатка `voxlink2` — тому й не знаходився пошуком.
   Google-Drive-версія: **CC BY-NC-SA 4.0**, некомерційна.
2. 🔴 **Рейтинг моделей НЕ переноситься між доменами.** Таблиця 3D-Speaker з колонкою
   CNCeleb: ECAPA-TDNN 3-тя за Vox1-O (0,86%) → **найгірша за CNCeleb (8,01%)**;
   ERes2NetV2 3-тя → **найкраща (6,14%)**. Обирати за колонкою чужого домену.
3. 🔴 **Тривалість — забутий вимір.** ERes2Net на 2 с: 0,84% → **3,28%** (×4).
   ERes2NetV2 на 2 с: **1,48%**. Розрив між моделями подвоюється зі скороченням
   запису (27% → 55%). Жодне число в каталозі не міряне в нашому режимі.
   → нова сторінка [[concepts/short-duration]].
4. 🔁 **Шортліст: №1 тепер ERes2NetV2** — виграє за обома критеріями одночасно.
   CAM++ — 7,2 M параметрів проти 20,8 M в ECAPA і точніша (кандидат на повний прогін).
   ECAPA знижено.
5. Побічно: VoxBlink2 0,228% → 0,170% від Score-Norm+QMF = **−25%**, незалежно
   збігається з −21% у VoxWatch. Калібрація працює навіть на SOTA.
6. ✅ У 3D-Speaker є **готовий RDINO-рецепт** — стадію 1 псевдоміток не писати з нуля.
7. Закрито 33, 34, 37б. Нові 38–42.

## [2026-08-27] research | Короткі записи для SimAMResNet + огляд репозиторію Wespeaker

Створено: [[sources/lit-wespeaker-repo]], [[models/simam-resnet]].
Оновлено: [[concepts/short-duration]], [[concepts/pseudo-label-training]],
[[models/wespeaker]], [[models/comparison]], [[experiments/clustering-pilot]] (новий
етап P0-bis), [[overview]] (теза 9), [[open-questions]], [[index]].

Ключове:
1. ❌ **Числа для SimAM_ResNet на 2–3 с ніде не опубліковані** — перевірено README
   Wespeaker, статтю VoxBlink2, пошук. Через це вибір між кандидатами 1a/1b
   **не робиться з паперу**. → новий етап **P0-bis**: обрізати Vox1-O до 2с/3с
   і поміряти самим. Кілька GPU-годин, без доступу до Telegram.
2. 🔴 **SimAM_ResNet34 (VoxBlink2) б'є ResNet293 (VoxCeleb2)**: 0,372 vs 0,425 Vox1-O,
   0,997 vs 1,146 Vox1-H. **34 шари проти 293.** Найчистіший доказ тези 8.
   SimAM_ResNet100: **0,202% Vox1-O** — удвічі краще за ResNet293.
3. **Траєкторія SSL повна**: DINO сам = 2,627% → +псевдомітки = ~1,01% → +LM FT = ~0,99%.
   **Стадія 2 дає більший приріст, ніж стадія 1** — пріоритет циклу, не DINO.
4. **Wespeaker покриває ВЕСЬ пайплайн**: SSL (DINO/MoCo/SimCLR), quality-aware
   calibration (= QMF), **PLDA з unsupervised adaptation**, AS-Norm, діаризація,
   UIO, DDP, ONNX/MNN. Рецепти: VoxCeleb, CNCeleb, **NIST SRE16**, VoxConverse, SSL.
   → брати як хребет проєкту, не збирати зі шматків.
5. NIST SRE16 як ще один зріз чужого домену: 6,608% Pooled / 10,01% Tagalog /
   2,974% Cantonese.
6. Шортліст: 1a SimAM_ResNet і 1b ERes2NetV2 **нарівні** — виграють за різними
   критеріями, у жодної немає повного набору чисел. Обидві в пілот.
7. Нові питання 43–45 (SdSV Challenge 2020, Whisper-PMFA/w2v-bert2/ReDimNet2, PLDA).

## [2026-10-06] export | Оформлення бази знань та інтеграція з Google NotebookLM

Створено: директорія `notebooklm/` (9 тематичних досьє + README), повноцінний блокнот у хмарі Google NotebookLM (ID: `05bff619-45df-4903-b2ac-1bb8e7117a47`).
Оновлено: `CLAUDE.md` (додано шар NotebookLM Export), `wiki/index.md`, `wiki/log.md`.

Ключове:
1. **Синтез бази знань у 9 досьє:** подолано фрагментацію 47 файлів Obsidian wikilinks та нечитабельність `.ipynb` для Gemini RAG.
2. **Завантажено та класифіковано 14 джерел у NotebookLM:** 9 синтезованих досьє, 2 оригінальні наукові PDF (WeSpeaker, VoxWatch) та 2 первинні текстові звіти. Усі джерела проіндексовані у статусі `ready` та згруповані у 6 колірних категорій з емодзі (Стратегія, Моделі, Акустика, Корпус, Експерименти, PDF).
3. **Налаштовано Chat Persona:** Lead Audio AI & Speaker Biometrics Architect з розширеними відповідями (`longer`) українською мовою.
4. **Створено 6 закріплених нотаток:** Резюме тез, матриця моделей, план пілоту P0-bis→P5, блокери P0, рекомендовані запити, навігатор артефактів.
5. **Згенеровано максимальний пакет AI Studio артефактів:** 
   - 3 аналітичні звіти: Briefing Doc, Study Guide, Blog Post.
   - 5 інтерактивних форматів: Mind Map, Flashcards, Quiz, Data Table, Infographic, Slide Deck.
   - Deep Dive Audio Overview (подкаст українською мовою).
6. **Верифікація:** Успішні RAG-запити до Gemini з точним цитуванням джерел та повна працездатність сесії NotebookLM.

---

## 2026-10-06 — Синхронізація архітектури: VOICE_ID + OpenEvolve як Single Source of Truth

**Дія:** Реструктуризація бази знань та конвеєра навколо `VOICE_ID_OPENEVOLVE_MASTER.md`.  
**Результат:**
1. **Майстер-контекст інтегровано:** `docs/VOICE_ID_OPENEVOLVE_MASTER.md` та `[[master-context]]` офіційно встановлені як Single Source of Truth (SSOT) проєкту.
2. **Переорієнтація бейзлайн-енкодера:** Створено сторінку `[[models/redimnet2]]` (ReDimNet2-B6 продакшн, B3 експерименти) як основного бейзлайна з оптимальним Pareto accuracy/compute.
3. **Еволюційний рушій:** Створено сторінку `[[concepts/openevolve]]` — визначено ролі OpenEvolve (мутації коду, Island Model, MAP-Elites) та VoiceEvolve (governance, детермінований Evaluator, 5 Promotion Gates).
4. **Оновлення синтезу:** Повністю переписано `[[overview]]` та `CLAUDE.md` відповідно до закону *«False merge набагато гірший за false split»*, масштабу ~3 TB неоднорідного аудіо (RU, UA, Surzhyk) та 7 цілей мутацій коду.
5. **Google NotebookLM:** Завантажено `09 — VOICE_ID + OpenEvolve Master Technical Context`, створено окрему 7-му категорію `🧬 Алгоритмічна Еволюція (OpenEvolve)` та 7-му закріплену нотатку.


## [2026-10-14] query | Детальний аудит проєкту VoiceID

Створено: [[concepts/project-audit-2026-10-14]], [[sources/lit-2026-10-14-audit-upstream]].
Доповнено: [[index]], [[open-questions]]. Raw `docs/` не редагувались; попередні незакомічені зміни збережено. Хмарний NotebookLM не змінювався.

Перевірено локальну структуру/Git, master-контекст, ключові wiki та exports, code/outputs Kaggle-ноутбука, первинні PDF-таблиці WeSpeaker/VoxWatch, офіційний upstream ReDimNet2/OpenEvolve і FAISS/Ward documentation. На початок аудиту: 50 wiki-сторінок, 78 файлів без `.git/` і `temp/`; runnable production/evaluator code відсутній. Три master-копії SHA256-ідентичні; змістовні розходження — у тематичних сторінках.

Ключові знахідки: різні training sets у порівнянні 0,723%/6,492%; помилкова атрибуція 9,141% SpeechBrain; непідтверджені 512 dimensions та вбудована L2-normalization ReDimNet2; ризики split-half/random negatives/200 pairs; пропущена пам'ять графа; невизначені SafeCoverage/noise contracts. Minimal reproduction підтвердив invalid float format у cell 2; AST code cells валідний. ML inference/training/production тести не запускались, власні benchmark results не заявляються.

## [2026-10-14] lint | Детальний аналіз проєкту після синхронізації з майстер-документом

Створено: [[project-audit]]. Оновлено: [[open-questions]] (№46–54, технічний борг), [[index]].

Ключове:
1. Проєкт — база знань + специфікація; коду й власних прогонів немає.
2. Майстер-документ інтегровано 2026-10-06 без ingest: немає `sources/`-сторінки, старі
   сторінки не переглянуті → 6 суперечностей (корпус 2 ТБ vs 3 TB, «міток немає» vs
   Gold-Small 118 мовців, енкодер, порядок Level A/B, мін. тривалість, два плани).
3. Метрики майстра не готові до заморожування: SafeCoverage без формули, `safety_limit`
   без числа, «FMR» = 1 − pair precision.
4. Нові ризики: потужність Gold-Small (~24 мовці на розбиття), дедуплікація, чистота
   distractors, граф k=200 від ~19 до ~300 ГБ залежно від бітрейту.
5. Wiki-борг W1–W13: три копії майстра, два SSOT, втрачені тези [[overview]],
   непідкріплені твердження в [[models/redimnet2]] і [[concepts/openevolve]],
   NotebookLM 00–08 vs 09, 59 МБ m4a поза `.gitignore`.
6. Нічого не виправлено, крім службових сторінок — чекаємо відповідей на №46–48.


## [2026-10-14] query | Підготовка публічної публікації GitHub і детального README

На пряме прохання користувача підготовлено `README.md` українською: фактичний статус research/pre-implementation, постановка задачі, Mermaid pipeline, роль VoiceEvolve/OpenEvolve, evaluator/метрики, результати аудиту, навігація, локальне читання й дорожня карта. Додано посилання на README у [[index]]. Ціль публікації — публічний `cyberdid/VoiceID`.

Перевірено 24 локальні README links/anchors, Markdown fences та відсутність trailing whitespace у README; виконано pattern scan credentials для поточних текстових файлів і 51 historical text blobs, збігів не виявлено. Первинні `docs/` не змінювались. Model weights, embeddings і приватний цільовий корпус у snapshot не входять; наявні NotebookLM exports включено. Production тести не запускались: executable implementation у репозиторії відсутня.

Перед публікацією уточнено формат експортної таблиці: `notebooklm/reports/data-table.json` містив CSV; перейменовано на `data-table.csv` без зміни вмісту. Інші JSON exports перевіряються з підтримкою UTF-8 BOM.

Через HTTP 408 під час Git push великі бінарні файли винесено в [GitHub Release v0.1.0](https://github.com/cyberdid/VoiceID/releases/tag/v0.1.0): два source PDF, infographic та згенерований `audio_overview.m4a` (59 МБ). Публічна `main` містить README, wiki, NotebookLM текстові exports, JSON/CSV та кодові матеріали.
