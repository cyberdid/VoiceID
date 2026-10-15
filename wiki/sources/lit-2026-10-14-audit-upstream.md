---
type: source
tags: [аудит, redimnet2, openevolve, faiss, clustering]
raw: web
retrieved: 2026-10-14
confidence: "Прочитано офіційні README, Python-код, YAML-рецепт та документацію. Ваги не завантажувались; інференс і навчання не запускались."
updated: 2026-10-14
---

# Перевірка upstream для аудиту VoiceID

Контекст і висновки: [[concepts/project-audit-2026-10-14]]. Ця довідка перевіряє конкретні технічні твердження; не оголошує жодну модель переможцем на приватному корпусі.

## ReDimNet2

Офіційний upstream: [PalabraAI/redimnet2](https://github.com/PalabraAI/redimnet2). Перевірений commit: `c5bbe0b76e37df698c403f8844e41304ceab6307`.

- У [рецепті B6 LM для VoxCeleb2](https://github.com/PalabraAI/redimnet2/blob/c5bbe0b76e37df698c403f8844e41304ceab6307/applications/speaker_rec/examples/vox2/recipes/redimnet2/b6/lm.yaml) встановлено `embed_dim: 192`, `F: 72`, `feat_type: tf`.
- [ReDimNet2Wrap](https://github.com/PalabraAI/redimnet2/blob/c5bbe0b76e37df698c403f8844e41304ceab6307/redimnet2/redimnet2.py) має дефолт `embed_dim=192`. `forward` повертає проєкцію без явного L2-normalize.
- [Завантажувач](https://github.com/PalabraAI/redimnet2/blob/c5bbe0b76e37df698c403f8844e41304ceab6307/redimnet2/__init__.py) створює модель із `model_config` конкретного checkpoint. Остаточну розмірність конкретних ваг потрібно перевірити після завантаження.
- README розрізняє набори `vox2`, `vb2+vox2_v0`, `vb2+vox2+cnc2_v0` та типи `ptn`, `lm`, `dis`. Назва «B6» сама по собі не ідентифікує систему.

## OpenEvolve

Перевірено поточні `evaluator.py` і `configs/default_config.yaml`; commit upstream під час перевірки: `a1e9d91eecde612f0efc8e8e31f59b36998248c0`.

- [Evaluator](https://github.com/algorithmicsuperintelligence/openevolve/blob/a1e9d91eecde612f0efc8e8e31f59b36998248c0/openevolve/evaluator.py) використовує `combined_score` для проходження cascade, а за його відсутності — усереднення числових метрик. У цьому коді optional LLM feedback може змінювати `combined_score` у пропорції 70/30. Пряме виконання evaluator запускається через executor із async timeout.
- [Конфіг](https://github.com/algorithmicsuperintelligence/openevolve/blob/a1e9d91eecde612f0efc8e8e31f59b36998248c0/configs/default_config.yaml) розрізняє fitness і `feature_dimensions` MAP-Elites; `use_llm_feedback` за замовчуванням вимкнено. Коментар конфіга зазначає, що `memory_limit_mb` і `cpu_limit` не реалізовані.

**Висновок агента:** VoiceEvolve має сам забезпечити hard gates, sandbox, process termination, контроль RAM/CPU та окреме збереження Pareto frontier. Сам timeout чи MAP-Elites цього не доводять.

## FAISS та Ward

- [Офіційна таблиця FAISS](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes) указує FP32-вектори плюс графові структури для `IndexHNSWFlat`; це не розмір FP16-файлу ембедингів. Для inner product як cosine вектори слід нормалізувати заздалегідь.
- [Документація AgglomerativeClustering](https://scikit-learn.org/stable/modules/generated/sklearn.cluster.AgglomerativeClustering.html) обмежує Ward евклідовою метрикою. Sparse connectivity обмежує структуру агломерації, але не є доказом лінійної масштабованості.

## Межі перевірки

Не перевірено checkpoint binaries, GPU throughput, приватний корпус, Gold-Small, хмарний NotebookLM та сумісність усього майбутнього стеку. Посилання на `main` і stable-документацію можуть змінюватися; для відтворюваних експериментів потрібні commit/version pins.
