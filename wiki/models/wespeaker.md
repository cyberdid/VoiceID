---
type: model
tags: [wespeaker, toolkit, resnet, ecapa, кандидат]
sources: [wespeaker-toolkit, pretrained-models-review]
status: candidate
updated: 2026-08-27
---

# Wespeaker

[github](https://github.com/wenet-e2e/wespeaker) · [huggingface](https://huggingface.co/Wespeaker) ·
[стаття](https://arxiv.org/pdf/2210.17016) · [демо](https://huggingface.co/spaces/wenet/wespeaker_demo)

**Не одна модель, а тулкіт + сімейство чекпойнтів.** Після ingest'у статті
([[sources/wespeaker-toolkit]]) статус змінено: з «резерв другої хвилі» на
**сильного кандидата №1**.

## Чому статус підвищено

**1. Найкращі числа на однаковому протоколі.** Wespeaker-реалізації обходять оригінальні
на тому самому тесті:

| Архітектура | Vox1-O | Vox1-E | Vox1-H |
| :-- | --: | --: | --: |
| ECAPA-TDNN (оригінальна) | 0,870 | 1,120 | 2,120 |
| **ECAPA-TDNN (Wespeaker)** | 0,728 | 0,929 | 1,721 |
| ResNet34 (оригінальна) | 1,31 | 1,38 | 2,50 |
| **ResNet34 (Wespeaker)** | **0,723** | **0,867** | **1,532** |
| ResNet221 | 0,505 | 0,676 | 1,213 |
| ResNet293 | **0,447** | 0,657 | 1,183 |

**2. Перевага на іншому домені ще більша.** На CNCeleb ([[datasets/cnceleb]]):
Wespeaker ResNet34 **6,492%** проти SpeechBrain ResNet34 **9,141%** — **−29% відносно**,
при однаковій архітектурі. Оскільки наш домен теж «не VoxCeleb», ця цифра релевантніша
за різницю на Vox1-O.

**3. Готовий до продакшну.** ONNX / TensorRT / TorchScript експорт, Triton, pip-біндинги,
DDP-навчання, on-the-fly аугментація, UIO для великих датасетів.

## Повний каталог чекпойнтів (перевірено 2026-08-27)

Wespeaker сам ділить моделі на три мовні категорії — і це найкорисніша інформація
з усього каталогу ([[sources/lit-model-checkpoints]]):

| Категорія | Моделі |
| :-- | :-- |
| **VoxCeleb — English** | ResNet34(_LM), ResNet152_LM, ResNet221_LM, **ResNet293_LM**, CAM++(_LM), ECAPA512(_LM), ECAPA1024(_LM), Gemini_DFResnet114_LM, W2V-BERT2.0, **ReDimNet-B6-LM** |
| **CNCeleb — CN** | ResNet34, ResNet34_LM |
| **VoxBlink2 — Multilingual** ⭐ | **SimAMResNet34**, **SimAMResNet100**, + варіанти «VoxBlink2 pretrain → VoxCeleb2 finetune» |

**Для російськомовного [[datasets/telegram-corpus]] брати треба з третього рядка.**
Це не інтерпретація — це власна класифікація авторів тулкіта.

HuggingFace: `Wespeaker/wespeaker-voxlink2-samresnet34_ft`
⚠️ **опечатка в назві** — `voxlink2`, не `voxblink2`. Пошук по «voxblink» його не
знаходить, натомість видає сторонні перезаливки без model card — їх не брати.

Побічно: SimAM-ResNet100-ASP на VoxCeleb2 + VoxBlink дає **EER 0,441%** на Vox1-O
*(з анотації пошуку, стосується VoxBlink v1; перевірити)*.
Підтримка SimAM_ResNet на VoxBlink2 — з ~вересня 2024.

**Нове в каталозі, чого не було в жодному документі `docs/`:** ReDimNet,
Gemini_DFResnet114, W2V-BERT2.0. Всього в HF-організації 16 моделей.

## Чекпойнти на VoxCeleb

- `pyannote/wespeaker-voxceleb-resnet34-LM` — **найлегший спосіб узяти Wespeaker-ResNet34**:
  через pyannote-обгортку, без збірки тулкіта. 512-вимірний ембединг.
  Суфікс `-LM` = large margin fine-tuning ([[concepts/loss-functions]]).
- HuggingFace `Wespeaker` — ~15 моделей, варіації ECAPA-TDNN і ResNet, треновані на
  VoxCeleb, **CNCeleb**, **VoxBlink2**.

> ⚠️ [[sources/pretrained-models-review]] застерігає: «для більшості з них відсутні
> інструкції встановлення та завантаження… завантаження ускладнене». Це стосується
> HF-колекції; чекпойнт через pyannote береться тривіально.

**Розшифровка «EER 0,867–1,959%»** з огляду: 0,867 **точно збігається** з Vox1-E EER
ResNet34 у таблиці статті. Отже це не інтервал невизначеності, а **різні протоколи
Vox1-O / E / H**, зведені в один діапазон. *(інференс агента з зіставлення двох джерел;
верхня межа 1,959 точного відповідника в таблиці не має — можливо, інший чекпойнт.)*

## Що взяти з тулкіта, навіть не використовуючи його моделі

Тренувальний рецепт ([[concepts/loss-functions]], [[concepts/fine-tuning]]):
80-dim Fbank 25/10 мс, чанки 200 кадрів, CMN без CVN, MUSAN **або** RIR з p=0,6,
speed perturb 0,9/1,0/1,1 у 1:1:1 **з трактуванням копій як нових дикторів**,
AAM-softmax з триетапним margin-розкладом, large margin fine-tuning (margin 0,5, 6 с).

## Тулкіт покриває весь наш пайплайн

Огляд репозиторію ([[sources/lit-wespeaker-repo]]) показав, що в Wespeaker вже є
кожен компонент плану:

- **SSL: DINO, MoCo, SimCLR** — DINO дає 2,627% на Vox1-O **без міток**;
- **quality-aware score calibration** = QMF з [[sources/voxwatch-osi-benchmark]];
- **PLDA з unsupervised adaptation** — адаптація до нерозміченого домену;
- **діаризація** (рецепт VoxConverse), рецепти **CNCeleb** і **NIST SRE16**;
- фронтенди **WavLM / Whisper-PMFA / w2v-bert2**, архітектури до **ReDimNet2**;
- DDP, on-the-fly аугментація, UIO-шарди, експорт ONNX/TensorRT/MNN.

Останнє оновлення — 2026.07.03. Проєкт живий.

## Оцінка для проєкту

**Кандидат №1 разом із [[models/resnet]]** — фактично це та сама ResNet34, але з кращим
рецептом. Мінімальний шлях: узяти `pyannote/wespeaker-voxceleb-resnet34-LM` і порівняти
з SpeechBrain-чекпойнтом на власних даних. Якщо різниця відтвориться — брати Wespeaker.

## Джерела

[[sources/wespeaker-toolkit]] · [[sources/pretrained-models-review]]
