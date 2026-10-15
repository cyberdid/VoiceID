---
type: source
raw: web
tags: [чекпойнти, доступність, ліцензії, benchmark, веб-дослідження]
retrieved: 2026-08-27
confidence: "Офіційні сторінки моделей і arXiv HTML. Вища за lit-ssl-clustering, але самі ваги не завантажувались і не перевірялись."
updated: 2026-08-27
---

# Довідка: доступність чекпойнтів (VoxBlink2, 3D-Speaker, Wespeaker)

Перевірка на замовлення 2026-08-27. Закриває блокер №33 і питання №34 з [[open-questions]].

## ✅ VoxBlink2 — чекпойнти є, з ліцензійним обмеженням

Джерело: <https://voxblink2.github.io>

| | |
| :-- | :-- |
| Завантаження | [Google Drive](https://drive.google.com/drive/folders/19uUmlnSjmFhffO7vHXO7upZJI77OOmrP), «моделі різного розміру» |
| Найбільша модель, без пост-обробки | **EER 0,228%** на Vox1-O |
| + Score-Norm і **QMF** | **EER 0,170%**, minDCF 0,006% |
| Тренувальні дані | VoxBlink2 + VoxCeleb (обидва з YouTube) |
| **Ліцензія** | **CC BY-NC-SA 4.0 — комерційне використання заборонене** |

Сторінка не перелічує архітектури окремих чекпойнтів — лише «різного розміру».

> 💡 **Побічне підтвердження:** 0,228% → 0,170% дає **−25% відносно** від самих лише
> Score-Norm + QMF. Незалежно збігається з −21% у [[sources/voxwatch-osi-benchmark]].
> Калібрація працює навіть на SOTA-моделі — [[concepts/score-normalization]].

Контекст проєкту — дослідження ([[datasets/telegram-corpus]]), тож NC-ліцензія не блокує.
Але це треба знати наперед, якщо колись зайде мова про застосування.

## ✅ Wespeaker — VoxBlink2-моделі є, і саме як «Multilingual»

Джерело: [docs/pretrained.md](https://github.com/wenet-e2e/wespeaker/blob/master/docs/pretrained.md)

Wespeaker ділить свої чекпойнти на три категорії — і це найважливіше:

| Категорія | Моделі |
| :-- | :-- |
| **VoxCeleb (English)** | ResNet34, ResNet34_LM, ResNet152_LM, ResNet221_LM, **ResNet293_LM**, CAM++, CAM++_LM, ECAPA512(_LM), ECAPA1024(_LM), Gemini_DFResnet114_LM, W2V-BERT2.0 |
| **CNCeleb (CN)** | ResNet34, ResNet34_LM |
| **VoxBlink2 (Multilingual)** ⭐ | **SimAMResNet34**, **SimAMResNet100** — і варіанти «VoxBlink2 (pretrain) + VoxCeleb2 (finetune)» |

**Це прямо відповідає на мовне питання проєкту.** У власній таксономії Wespeaker
англомовна лінійка — це VoxCeleb, китайська — CNCeleb, а **багатомовна — VoxBlink2**.
Для російськомовного корпусу перший кандидат очевидний.

**Чекпойнт на HuggingFace:** `Wespeaker/wespeaker-voxlink2-samresnet34_ft`
⚠️ Зверніть увагу на **опечатку в назві репозиторію**: `voxlink2`, а не `voxblink2`.
Через це він не знаходиться пошуком по «voxblink» — саме тому в попередньому раунді
знайшлись лише сторонні перезаливки (`gaunernst/wespeaker-voxblink2-samresnet34`,
**без model card** — брати не варто).

Побічно: **SimAM-ResNet100-ASP, тренована на VoxCeleb2 + VoxBlink, дає EER 0,441%
на Vox1-O** *(з анотації пошуку, стосується VoxBlink v1, не v2 — перевірити)*.

Також в організації є `wespeaker-voxceleb-redimnet2-B6-LM` — **ReDimNet**, архітектура,
якої немає в нашому каталозі. Всього в організації 16 моделей.
Підтримка SimAM_ResNet на VoxBlink2 з'явилась ~вересень 2024.

## ⭐ 3D-Speaker — бенчмарк за єдиним протоколом на ТРЬОХ доменах

Джерело: [README](https://github.com/modelscope/3D-Speaker/blob/main/README.md)

| Модель | Параметрів | Vox1-O | **CNCeleb** | 3D-Speaker |
| :-- | --: | --: | --: | --: |
| Res2Net | 4,03 M | 1,56% | 7,96% | 8,03% |
| ResNet34 | 6,34 M | 1,05% | 6,92% | 7,29% |
| **ECAPA-TDNN** | 20,8 M | 0,86% | **8,01%** ⚠️ | 8,87% |
| ERes2Net-base | 6,61 M | 0,84% | 6,69% | 7,21% |
| **CAM++** | **7,2 M** | **0,65%** | 6,78% | 7,75% |
| **ERes2NetV2** | 17,8 M | 0,61% | **6,14%** ⭐ | **6,52%** |
| ERes2Net-large | 22,46 M | **0,52%** | 6,17% | 6,34% |

> 🔴 **Найважливіше відкриття цього раунду: рейтинг НЕ переноситься між доменами.**
>
> **ECAPA-TDNN** — 3-тя за Vox1-O (0,86%), але **найгірша за CNCeleb (8,01%)** —
> гірша навіть за Res2Net (7,96%), яка на Vox1-O удвічі слабша.
> **CAM++** — 2-га за Vox1-O (0,65%), але лише 4-та за CNCeleb (6,78%).
> **ERes2NetV2** — 3-тя за Vox1-O (0,61%), але **найкраща за CNCeleb (6,14%)**.
>
> Для [[datasets/telegram-corpus]] (російська, чужий домен для всіх) обирати треба
> **за колонкою CNCeleb, а не Vox1-O**. Це переставляє шортліст.

**CAM++ — 7,2 M параметрів проти 20,8 M в ECAPA-TDNN, і при цьому точніша**
(0,65% vs 0,86%). Утричі менша й краща. На мільйонах записів це доби прогону.

Третє незалежне підтвердження ціни домену: Vox1-O 0,52–1,56% → CNCeleb 6,14–7,96%,
тобто ~×10. Збігається з [[sources/wespeaker-toolkit]] ([[datasets/cnceleb]]).

⚠️ Порівнювати **всередині** таблиці — коректно (один тулкіт, один протокол).
Зіставляти з таблицею Wespeaker — обережно: там ResNet34 дає 0,723% / 6,492%,
тут 1,05% / 6,92%. Різні рецепти.

**ModelScope ID:**
- `iic/speech_eres2netv2_sv_zh-cn_16k-common` — ERes2NetV2, **200 тис. дикторів**
- `iic/speech_campplus_sv_zh-cn_16k-common` — CAM++, **200 тис. дикторів**
- `pip install modelscope`

⚠️ Заголовок моделі на ModelScope: **«ERes2NetV2说话人确认-中文-通用-200k-Spkrs»** —
тобто **200 тис. КИТАЙСЬКИХ дикторів**, не багатомовних. Це суттєве уточнення:
за кількістю дикторів вона попереду, за мовою — далі від російської, ніж VoxBlink2.
Чесна відповідь: **перевіряти обидві**.

**Бонус:** у 3D-Speaker є готовий рецепт **RDINO** (`egs/voxceleb/sv-rdino`) —
self-supervised DINO для мовця. Це готова реалізація стадії 1 з
[[concepts/pseudo-label-training]].

## ⭐ ERes2NetV2 і короткі записи

[arXiv:2406.02167](https://arxiv.org/html/2406.02167v1) (Interspeech 2024) —
«Boosting Short-Duration Speaker Verification Performance with Computational Efficiency».

| Тривалість | ERes2Net | ERes2NetV2 | Покращення |
| :-- | --: | --: | --: |
| повна | 0,84% | 0,61% | −27% |
| **3 с** | 1,89% | **0,98%** | **−48,1%** |
| **2 с** | 3,28% | **1,48%** | **−54,8%** |

Архітектурно: **BDFF** (bottom-up dual-stage feature fusion, злиття стадій 3 і 4 з
обрізанням надлишкових зв'язків) + **BLFF** (bottleneck-подібне локальне злиття,
expand-reduce-expand усередині residual-блоків).
Вартість: 17,8 M параметрів / 12,6 GFLOPs проти 6,61 M / 5,16 GFLOPs в ERes2Net.

**Це найважливіше число раунду для нашого корпусу** — голосові Telegram короткі.
Наслідки винесені в окрему сторінку: [[concepts/short-duration]].

## Що це змінює

1. ✅ Блокер №33 закрито: **VoxBlink2-чекпойнти доступні**, офіційно в Wespeaker,
   категорія «Multilingual», HF-репозиторій `wespeaker-voxlink2-samresnet34_ft`.
2. ✅ Питання №34 закрито: EER для CAM++ і ERes2NetV2 отримані.
3. 🔴 **Рейтинг не переноситься між доменами** → шортліст перебудувати за CNCeleb.
4. 🔴 **Тривалість — окремий критерій відбору** → [[concepts/short-duration]].
5. ⚠️ VoxBlink2 — **CC BY-NC-SA 4.0**, некомерційна.
6. Нові моделі поза каталогом: **ReDimNet**, **Gemini_DFResnet114**, **W2V-BERT2.0**.

## Пов'язані

[[concepts/short-duration]] · [[models/3d-speaker]] · [[datasets/voxblink2]] ·
[[models/wespeaker]] · [[models/comparison]] · [[sources/lit-ssl-clustering]] ·
[[concepts/pseudo-label-training]]
