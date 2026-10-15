---
type: source
raw: web
tags: [wespeaker, репозиторій, ssl, dino, calibration, веб-дослідження]
retrieved: 2026-08-27
confidence: "Офіційний README та examples/voxceleb/v2/README.md. Код не запускався."
updated: 2026-08-27
---

# Довідка: репозиторій Wespeaker (огляд можливостей)

<https://github.com/wenet-e2e/wespeaker> · перевірено 2026-08-27

> **Головний висновок: Wespeaker покриває весь наш пайплайн — від SSL-навчання без
> міток до калібрації й деплою. Збирати систему з окремих шматків не треба.**
> *(висновок агента)*

## ⭐ Повна таблиця результатів VoxCeleb

Конфіг: `fbank80, num_frms200, epoch150, ArcMargin, aug_prob0.6, speed_perturb, no spec_aug`.
Скоринг: `cosine (sub mean of vox2_dev) + AS-Norm + QMF`.

| Модель | LM | AS-Norm | QMF | vox1-O | vox1-E | vox1-H |
| :-- | :-: | :-: | :-: | --: | --: | --: |
| **SimAM_ResNet100** (VoxBlink2 pretrain → Vox2 FT) | ✓ | ✓ | ✓ | **0,202** | **0,421** | **0,795** |
| **SimAM_ResNet34** (VoxBlink2 pretrain → Vox2 FT) | ✓ | ✓ | ✓ | **0,372** | **0,559** | **0,997** |
| ResNet293-TSTP-emb256 | ✓ | ✓ | ✓ | 0,425 | 0,641 | 1,146 |
| ResNet221-TSTP-emb512 | ✓ | ✓ | ✗ | 0,505 | 0,676 | 1,213 |
| ResNet34-TSTP-emb256 | ✓ | ✓ | ✓ | 0,659 | 0,821 | 1,437 |
| ECAPA_TDNN_GLOB_c1024-ASTP-emb192 | ✓ | ✓ | ✓ | 0,707 | 0,894 | 1,615 |

> 🔴 **Головне: SimAM_ResNet34, попередньо натренована на VoxBlink2, б'є ResNet293,
> натреновану лише на VoxCeleb2** — 0,372 проти 0,425 на Vox1-O і 0,997 проти 1,146
> на Vox1-H. **34 шари обходять 293 шари.**
>
> Різниця не в архітектурі й не в розмірі — а в тому, що модель бачила 110 тис.
> дикторів замість 6 тис. Це найчистіший доказ тези «широта тренувальних даних
> важить більше за архітектуру» ([[overview]], теза 8).
>
> SimAM_ResNet100 іде далі: **0,202% на Vox1-O** — удвічі краще за ResNet293.

## ⭐ SSL: DINO, MoCo, SimCLR — вбудовані

Wespeaker реалізує три SSL-методи. Результат **без жодних людських міток**:

| Метод | vox1-O | vox1-E | vox1-H |
| :-- | --: | --: | --: |
| ECAPA_TDNN_GLOB_c1024 + **DINO** | **2,627%** | 2,665% | 4,644% |

DINO (без негативних пар) істотно кращий за контрастивні SimCLR і MoCo.

> 💡 **Це дає повну траєкторію циклу з псевдомітками:**
>
> | Стадія | EER на Vox1-O |
> | :-- | --: |
> | DINO сам по собі (стадія 1) | **2,627%** |
> | + ітеративні псевдомітки, 2-га ітерація ([[sources/lit-ssl-clustering]]) | **~1,01%** |
> | + large margin fine-tuning | **~0,99%** |
> | *для порівняння — supervised* | *0,425–0,723%* |
>
> **Стадія 2 дає більший приріст, ніж стадія 1**: DINO знімає 2,6%, а кластеризація
> з псевдомітками ріже це більш ніж удвічі. Тобто основна цінність — саме в циклі,
> а не в самому DINO. Важливо для пріоритетів у [[concepts/pseudo-label-training]].

## Що ще є в тулкіті

| Можливість | Чому це нам потрібно |
| :-- | :-- |
| **Quality-aware score calibration** (з 2024.05.15) | це і є **QMF** з [[sources/voxwatch-osi-benchmark]] — реалізовувати не треба |
| **PLDA з unsupervised adaptation** (Kaldi-сумісна) | адаптація бекенда до нерозміченого домену — рівно наш випадок |
| Score calibration, AS-Norm | [[concepts/score-normalization]] |
| **Діаризація** (рецепт VoxConverse) | якщо в записах кілька мовців ([[concepts/task-types]]) |
| Фронтенди: **WavLM**, **Whisper-PMFA**, **w2v-bert2** | великі ASR-моделі як фронтенд — див. [[models/wavlm]] |
| Архітектури: ResNet, ECAPA-TDNN, **ERes2Net**, Res2Net, **ReDimNet2** | ERes2Net доступна і тут, не лише в [[models/3d-speaker]] |
| Runtime: PyTorch, **MNN**, ONNX/TensorRT | деплой, якщо дійде |
| Онлайн-ознаки або готові kaldi-формат | [[concepts/audio-preprocessing]] |

## Рецепти

**VoxCeleb**, **CNCeleb**, **NIST SRE16**, **VoxConverse** (діаризація), **SSL**.

Результати NIST SRE16 — ще один зріз чужого домену (телефон + інші мови):
**6,608%** Pooled, **10,01%** Tagalog, **2,974%** Cantonese.
Той самий порядок деградації, що на CNCeleb ([[concepts/domain-mismatch]]).

## Хронологія оновлень

| Дата | Що |
| :-- | :-- |
| 2026.07.03 | **ReDimNet2** з фронтендом TFMel |
| 2025.12.05 | підтримка **w2v-bert2** |
| **2024.09.03** | **SimAM_ResNet і претреновані моделі VoxBlink2** |
| 2024.08.30 | **Whisper-PMFA** |
| 2024.05.15 | **quality-aware score calibration** |

Проєкт активний — останнє оновлення за півтора місяця до сьогодні.

## Суміжні репозиторії

- [`BUTSpeechFIT/wespeaker_ssl_public`](https://github.com/BUTSpeechFIT/wespeaker_ssl_public) —
  використання претренованих SSL-трансформерів для верифікації.
- [`theolepage/wavlm_ssl_sv`](https://github.com/theolepage/wavlm_ssl_sv) — заявлено як
  SOTA для self-supervised верифікації через великий претренований ASR;
  ймовірно, реалізація [arXiv:2406.02285](https://arxiv.org/html/2406.02285v1).
- [`wsstriving/wespeaker_plda`](https://github.com/wsstriving/wespeaker_plda) — PLDA-гілка.
- **SSPS** ([arXiv:2505.14561](https://arxiv.org/pdf/2505.14561)) — self-supervised
  positive sampling; **curriculum learning** для SSL SV
  ([arXiv:2203.14525](https://arxiv.org/pdf/2203.14525)).

## ❌ Чого немає: короткі записи для SimAMResNet

**Числа EER для SimAM_ResNet на 2–3-секундних висловлюваннях не опубліковані** —
ані в README тулкіта, ані в статті VoxBlink2, ані деінде в знайденому.
Опубліковані короткі-тривалості числа є лише для ERes2Net/ERes2NetV2
([[concepts/short-duration]]).

Це не глухий кут: **міряється самостійно за кілька GPU-годин** — обрізати
випробування Vox1-O до 2 с і 3 с та прогнати обидві моделі. Такий замір закриває
питання №40 і одразу входить в етап P0 [[experiments/clustering-pilot]].

Побічний орієнтир зі знайденого: перехід з 3,59 с на 2,05 с у сторонній системі
дав **+46% відносно** EER (8,72% → 12,8%). Порядок деградації такий самий, як
в ERes2Net.

Окремий бенчмарк для цієї задачі: **SdSV Challenge 2020** (Short-duration Speaker
Verification) — не досліджений, може бути корисним джерелом протоколу.

## Пов'язані

[[models/simam-resnet]] · [[models/wespeaker]] · [[concepts/short-duration]] ·
[[concepts/pseudo-label-training]] · [[concepts/score-normalization]] ·
[[datasets/voxblink2]] · [[models/comparison]] · [[experiments/clustering-pilot]]
