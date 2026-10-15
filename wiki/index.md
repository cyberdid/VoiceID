---
type: overview
tags: [каталог]
updated: 2026-10-14
---

# Index

Каталог усіх сторінок wiki. Оновлюється при кожному ingest.
Правила ведення — `CLAUDE.md`. Хронологія — [[log]].

## Головне

| Сторінка | Про що |
| :-- | :-- |
| [README](../README.md) | Публічний вступ: стан проєкту, архітектура, результати аудиту, навігація та дорожня карта |
| [[master-context]] | 👑 **Головне джерело правди (SSOT)**: Master Technical Context VoiceID + OpenEvolve |
| [[overview]] | **Генеральний огляд.** Робоча теза, моделі, VoiceEvolve, план відкритої біометрії |
| [[project-audit]] | 🔎 **Аудит 2026-10-14**: 6 суперечностей майстра з wiki, діри в метриках, критичний шлях |
| [[open-questions]] | питання №1–54; 🔴 блокери — який корпус (№46), Gold-Small (№47), структура тек, шлях до даних |
| [[log]] | Хронологічний журнал: ingest / query / lint / experiment |
| [[concepts/project-audit-2026-10-14]] | Детальний аудит: фактична зрілість, помилки джерел, калібрація, RAM/graph, ReDimNet2, OpenEvolve та пріоритети реалізації |

## Джерела (9)

| Сторінка | Джерело в `docs/` | Тип |
| :-- | :-- | :-- |
| [[sources/pretrained-models-review]] | «Пре-треновані моделі (ідентифікація голосу).md» | внутрішній огляд 12 моделей ⭐ найпрактичніший |
| [[sources/voice-id-parameters-methods]] | «Ідентифікація голосу_ параметри та методології.md» | великий теоретичний звіт, 90 посилань |
| [[sources/voxwatch-osi-benchmark]] | `2307.00169v1.pdf` | стаття AWS AI Labs ⭐ найсильніша методологічно |
| [[sources/wespeaker-toolkit]] | `2210.17016v2.pdf` | стаття про toolkit Wespeaker ⭐ єдине порівняння за єдиним протоколом |
| [[sources/lit-ssl-clustering]] | *веб-дослідження* | ⭐ література з self-supervised мовця й кластеризації в масштабі |
| [[sources/lit-model-checkpoints]] | *веб-дослідження* | ⭐ доступність чекпойнтів, ліцензії, бенчмарк на чужому домені, короткі записи |
| [[sources/lit-wespeaker-repo]] | *веб-дослідження* | ⭐ огляд репозиторію: повна таблиця EER, SSL-рецепти, QMF, PLDA-адаптація |
| [[sources/nb-voxceleb-real-audio-benchmark]] | `benchmarking-using-voxceleb-for-real-audio.ipynb` | Kaggle-ноутбук ⚠️ суміжна задача (anti-spoofing) |
| [[sources/lit-2026-10-14-audit-upstream]] | *перевірка офіційного upstream* | ReDimNet2 output/frontend, OpenEvolve evaluator/config, FAISS та Ward |

## Модельні сторінки (14, включно зі зведеним порівнянням)

| Сторінка | Ембединг | Заявл. EER | Статус |
| :-- | --: | --: | :-- |
| [[models/redimnet2]] | 512 | SOTA Pareto | 👑 **Production Baseline** (ReDimNet2-B6 / B3) згідно з Master Context |
| [[models/comparison]] | — | — | **зведена таблиця всіх моделей + шортліст** |
| [[models/simam-resnet]] | — | **0,202 / 0,795** (O/H) | 🥇 **кандидат 1a** — єдина офіційно multilingual; б'є ResNet293 маючи 34 шари |
| [[models/3d-speaker]] | — | 0,61% / **6,14% CNCeleb** | 🥇 **кандидат 1b** — ERes2NetV2: чужий домен + короткі записи |
| [[models/wespeaker]] | 512 | **0,723** (Vox1-O) | 🥉 найкращий ResNet-рецепт, найлегше взяти — опорна точка |
| [[models/resnet]] | 256 / 512 | 0,41–9,14% | 🥈 кандидат №2 — контроль + виграв український експеримент |
| [[models/ecapa-tdnn]] | 192 | 0,728 (Vox1-O) | 🥉 кандидат №3 — нарівні на Vox1-O, найпростіше встановлення |
| [[models/titanet]] | 192 | 0,66% | кандидат №4 — телефонні корпуси в тренуванні |
| [[models/rawnet3]] | 256 | 0,73–0,89% | кандидат №5 — сирий waveform, пара для fusion |
| [[models/wavlm]] | 512 | 4,04–4,69% | як фронтенд для дотреновування — 0,52% |
| [[models/pyannote]] | 512 | 2,80% | не як енкодер — але потрібна для діаризації/VAD |
| [[models/resemblyzer]] | 256 | 3,55% | санітарний baseline на 10 хвилин |
| [[models/wav2vec2]] | 768 | — | найнижчий пріоритет |
| [[models/speakernet]] | 256 | 2,10–2,29% | ✗ відхилено (погано на телефонних) |

## Поняття (16)

| Сторінка | Про що |
| :-- | :-- |
| [[concepts/openevolve]] | 🧬 **Evolution Engine**: OpenEvolve (AlphaEvolve, MAP-Elites, 7 Mutation Targets, 5 Gates) |
| [[concepts/project-audit-2026-10-14]] | Детальний аудит проєкту: докази, методологічні ризики, масштабування та порядок реалізації |
| [[concepts/pseudo-label-training]] | ⭐ **головний важіль**: ітеративне навчання енкодера на псевдомітках з кластеризації |
| [[concepts/speaker-clustering]] | 🔴 **поточна задача**: кластеризація 2 ТБ без міток — пайплайн, ANN-граф, оцінка без розмітки |
| [[concepts/task-types]] | верифікація / closed-set / **open-set** / діаризація / кластеризація |
| [[concepts/open-set-identification]] | ⭐ watchlist і «false-alarm problem»: FAR росте всемеро при 50→500 |
| [[concepts/eer-and-metrics]] | ⭐ EER, FRR@FAR=0,5%, FAR@FRR=5%, DET — і чому одного EER мало |
| [[concepts/score-normalization]] | AS-Norm / QMF-калібрація / fusion — найдешевші виграші (−21%, −44% FAR) |
| [[concepts/short-duration]] | ⭐ **забутий вимір**: на 2 с EER гірший учетверо, розрив між моделями подвоюється |
| [[concepts/domain-mismatch]] | ⭐ головний ризик проєкту: розрив між тренувальними й цільовими даними |
| [[concepts/fine-tuning]] | три робочі рецепти дотреновування + чотири небезпеки |
| [[concepts/loss-functions]] | AAM-softmax, margin-розклади, large margin fine-tuning |
| [[concepts/speaker-embedding]] | базовий пайплайн `audio → features → DNN → embedding → classifier` |
| [[concepts/acoustic-features]] | F0, форманти, jitter/shimmer/HNR, MFCC, мел-спектрограма |
| [[concepts/audio-preprocessing]] | 16 кГц / mono / тривалість / VAD / кодеки / telephony filter + код |
| [[concepts/anti-spoofing]] | суміжна задача: чи це людина взагалі (не хто саме) |

## Датасети (6)

| Сторінка | Про що |
| :-- | :-- |
| [[datasets/telegram-corpus]] | 🔴 **цільові дані**: ~2 ТБ, нерозмічені, російська; метадані як слабка розмітка |
| [[datasets/voxblink2]] | 🥈 **110 тис. дикторів**, SOTA 0,170%; єдина офіційно **multilingual** лінійка Wespeaker |
| [[datasets/voxceleb]] | 1211 + 5994 диктори; тренувальна база майже всіх моделей; протоколи Vox1-O і VoxWatch |
| [[datasets/ukrainian-data]] | 🔴 **найбільша прогалина проєкту** — блокер етапу 1 |
| [[datasets/cnceleb]] | ⭐ вимірювач ціни зміни домену: 0,723% → 6,492% (×9); multi-session enrollment |
| [[datasets/in-the-wild]] | дотичний, для anti-spoofing |

## Експерименти (2)

| Сторінка | Про що |
| :-- | :-- |
| [[experiments/clustering-pilot]] | 🔴 **наступний крок**: пілот кластеризації, етапи P0–P4 |
| [[experiments/protocol]] | протокол оцінювання ідентифікації: 5 етапів, 3 метрики, 6 залізних правил |

Власних прогонів ще не було. Починати з **етапу P0-bis** — замір коротких записів на Vox1-O (не потребує доступу до корпусу).

## Google NotebookLM (Повний інтеграційний комплекс)

| Ресурс | Опис |
| :-- | :-- |
| `notebooklm/` | Оформлений та завантажений пакет у Google NotebookLM (15 джерел у 7 категоріях, 7 нотаток, 10 AI-артефактів, включно з OpenEvolve Master Doc) |
| `notebooklm/README.md` | Повний путівник по блокноту, інструкція Persona, реєстр артефактів Studio та CLI-довідка |

## Статистика

- Сторінок wiki: **53** · Первинних файлів у `docs/`: **6** · Сторінок джерел: **9** · Модельних сторінок: **14** (включно зі зведеним порівнянням)
- Експортний комплекс NotebookLM: **10 тематичних досьє** + **2 PDF** + **2 первинних звіти** + **1 OpenEvolve Master Doc** (всього 15 джерел у 7 категоріях)
- Артефакти AI Studio: **Mind Map**, **Flashcards**, **Quiz**, **Data Table**, **Infographic**, **Slide Deck**, **3 Reports**, **Audio Overview**
- Останнє локальне оновлення: 2026-10-14 (аудит; хмарний NotebookLM повторно не перевірявся)
