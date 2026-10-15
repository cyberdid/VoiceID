---
type: concept
tags: [концепт, openevolve, voiceevolve, еволюція, алгоритми, llm-agent]
sources: [master-context]
updated: 2026-10-06
---

# OpenEvolve та VoiceEvolve

Еволюційний шар автоматичного відкриття та оптимізації алгоритмів у проєкті VoiceID.

## Розподіл ролей

- **OpenEvolve** (`algorithmicsuperintelligence/openevolve`, Apache-2.0) — *Evolution Engine*. Відкритий агентний рушій еволюції коду на базі LLM у стилі AlphaEvolve (мутації коду Python, Island Model, MAP-Elites).
- **VoiceEvolve** — повна дослідницька платформа та governance-система, що включає ізольоване середовище виконання, детермінований оцінювач (Evaluator) та 5 гейтів безпеки (Promotion Gates).
- **VoiceID** — кінцевий продакшн-пайплайн ідентифікації та кластеризації дикторів на масиві ~3 TB аудіо.

## Чому OpenEvolve, а не Optuna

Optuna оптимізує лише скалярні числа (пороги $\theta$, кількість сусідів $k$, марджіни).  
**OpenEvolve мутує саму алгоритмічну логіку коду Python**:
- Замінює стандартний косинус на складну топологічну метрику ребер:
  $\text{score} = \text{cosine} + \alpha \cdot \text{reciprocal\_support} + \beta \cdot \text{shared\_neighbors} - \gamma \cdot \text{local\_ambiguity}$
- Знаходить нові послідовності графів, відсікання ребер та динамічних порогів.

## 7 цілей мутацій (Mutation Targets)

1. **Target 1 — Similarity:** функція близькості ребер графа (косинус + взаємні сусіди + якість).
2. **Target 2 — Adaptive k:** динамічний підбір сусідів під локальну щільність.
3. **Target 3 — Edge Acceptance:** критерії валідації та збереження ребер графа.
4. **Target 4 — Conservative Cluster Merge:** консервативне об'єднання кластерів (правило: *False merge гірший за false split*).
5. **Target 5 — Split Detector:** виявлення забруднених кластерів та їх декомпозиція.
6. **Target 6 — Outlier Reassignment:** робота з поодинокими нерозв'язаними сегментами.
7. **Target 7 — Speaker Prototype:** багатовузлове представлення диктора замість єдиного центроїда.

## 5 гейтів просування (Promotion Gates)

1. **Gate 1 (Validity):** валідність синтаксису, детермінізм, відсутність NaN/Inf.
2. **Gate 2 (Safety):** нульовий витік (0 data leakage), суворий контроль $FAR_{merge} \approx 0$.
3. **Gate 3 (Quality):** перемога над еталонами (HDBSCAN, Sparse Ward, Leiden + CPM) на контрольних сплітах.
4. **Gate 4 (Robustness):** стійкість до 8 кГц GSM телефонії, шуму, коротких чанків 2 с та зміни мови.
5. **Gate 5 (Cost):** обмеження пам'яті та лінійний час виконання для масштабу 3 TB.
