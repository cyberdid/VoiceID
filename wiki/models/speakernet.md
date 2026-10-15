---
type: model
tags: [nemo, nvidia, quartznet]
sources: [pretrained-models-review]
status: rejected
updated: 2026-08-27
---

# SpeakerNet

| | |
| :-- | :-- |
| Модель | [NGC catalog: speakerverification_speakernet](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/nemo/models/speakerverification_speakernet) |
| Стаття | [arXiv:2010.12653](https://arxiv.org/abs/2010.12653) |
| Ембединг | 256 |
| Тренувальні дані | VoxCeleb |
| Архітектура | QuartzNet |
| Заявлений EER | 2,10–2,29% |

## Чому відхилено

> **Може гірше працювати на телефонних записах — їх не було в тренувальних даних**
> (застереження самої NVIDIA).

Якщо цільовий домен телефонний, це дискваліфікує модель ще до тестування. Плюс —
та сама важка залежність NeMo, що й у [[models/titanet]], але з гіршим заявленим EER
(2,1% проти 0,66%). За тієї ж ціни встановлення краще брати TitaNet.

**Статус можна переглянути**, якщо виявиться, що дані не телефонні.

## Джерела

[[sources/pretrained-models-review]]
