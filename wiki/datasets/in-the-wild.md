---
type: dataset
tags: [anti-spoofing, deepfake, kaggle]
sources: [nb-voxceleb-real-audio-benchmark]
updated: 2026-08-27
---

# In-The-Wild (audio deepfake)

Kaggle: `abdallamohamed312/in-the-wild-audio-deepfake`, тека `release_in_the_wild/real`
(використовувалась лише «справжня» частина).

## Використання

У [[sources/nb-voxceleb-real-audio-benchmark]] — джерело bonafide-записів поряд з
[[datasets/voxceleb]]:

| Прогін | Склад |
| :-- | :-- |
| Тренування | 450 In-The-Wild + 450 VoxCeleb1 (з 900 human загалом) |
| Аудит cell 2 | 200 + 200 = 400 (seed 101) |
| Аудит cell 9 | 300 + 300 = 600 (seed 42) |

## Чому змішували з VoxCeleb

*(висновок агента)* Два різні джерела «справжніх» голосів зменшують шанс, що класифікатор
вивчить «акустику VoxCeleb» замість «ознак справжнього голосу». Це та сама логіка, що
й склеювання сегментів у [[datasets/voxceleb]] — боротьба з кореляцією між міткою та каналом.

Корисний принцип і для нас: **негативи мають бути різнорідними за каналом**, інакше
модель вчить канал, а не голос.

## Релевантність

Дотична — це набір для [[concepts/anti-spoofing]], не для ідентифікації мовця.
Для нашої основної задачі не потрібен.

## Пов'язані

[[datasets/voxceleb]] · [[concepts/anti-spoofing]]
