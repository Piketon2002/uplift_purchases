<link rel = "stylesheet" type="text/css" href="style.css">

# Предпосылки


&ensp; &ensp;В конце 2022 года я принял участие в увлекательном соревновании по Аплифт-моделированию. Эта задача мне очень понравилась, и я решил немного расширить тему и побольше попрактиковаться в анализе имеющихся данных, обучении различных ML моделей, подборе гиперпараметров и построении различных графиков.

# Суть задачи

&ensp; &ensp;Есть данные о том, кто из клиентов именно тот, кто без СМС не купит, а с СМС - купит!

&ensp; &ensp;Нужно из всех test-клиентов найти и пометить **единицей** тех, кто:<br/>
&ensp; &ensp;&ensp;- купит только при коммуникации;  
&ensp; &ensp;&ensp;- не купит без коммуникации.<br/>
&ensp; &ensp;Остальных пометить **нулями**.

&ensp; &ensp;Датасеты можно найти по [ссылке](https://www.kaggle.com/competitions/uplift-shift-23/data/).

#### Описание данных

 * **train** - набор клиентов для обучения, с указанием `treatment_flg` — была ли совершена коммуникация, `purchased` — была ли совершена покупка
 * **test** - список клиентов, для которых необходимо предсказать `target`
 * **clients** - информация о клиентах
 * **products** -  информация о товарах
 * **train_purch** - история покупок train клиентов
 * **test_purch** - история покупок test клиентов

#### Метрики

&ensp; &ensp;Оценка модели производится по трем метрикам:

 * **uplift_at_k** - uplift в первых k наблюдениях (k=30%)
 * **uplift_auc_score** - площадь под графиком аплифта (Area Under the Uplift Curve) 
 * **qini_auc_score** - Area Under the Qini curve.

# Стек технологий

`Python`, `Pandas`, `Matplotlib`, `Seaborn`, `sklearn`,
`Catboost`, `LightGBM`, `hyperopt`, `functools`,  `SciPy`, `scikit-uplift`,
`feature_selector`

# Используемые модели

&ensp; &ensp;В целом, решение поставленной задачи сводится к определению величины uplift для каждого клиента из тестового набора.
<div align="center"> uplift = A - B, где </div>

A - доля людей, совершивших покупку, от всех людей, с которые было произведено взаимодействие.<br/>
B - доля людей, совершивших покупку, от всех людей, с которые не было взаимодействия.

&ensp; &ensp;По сути, требуется определить, на сколько изменится вероятность покупки клиентом после воздействия на него. Существует несколько подходов к решению данной задачи:

* <u>Одна модель с признаком коммуникации</u>: модель обучается одновременно на двух группах, при этом бинарный флаг коммуникации выступает в качестве дополнительного признака. Каждый объект из тестовой выборки скорится дважды: с флагом коммуникации равным 1 и равным 0. Вычитая вероятности по каждому наблюдению, получается искомый uplift;
* <u>Одна модель с трансформацией классов</u>: данный подход основан на преобразовании двух таргетов в один и обучении одной модели машинного обучения на данных с изменённой целевой переменной;
* <u>Две независимые модели</u>: подход заключается в моделировании условных вероятностей тестовой и контрольной групп отдельно;
* <u>Две зависимые модели</u>: идея состоит в том, что в начале обучается классификатор по контрольным данным, затем используются предсказания в качестве нового признака для обучения второго классификатора на тестовых данных.

&ensp; &ensp;Для решения задачи применялись следующие модели машинного обучения:

 * LGBMClassifier
 * CatBoostClassifier
 * RandomForestClassifier

&ensp; &ensp;Подбор гиперпараметров осуществляется методом байесовской оптимизации с помощью `hyperopt`.

# Результаты и выводы

&ensp; &ensp;Была поставлена задача определить перечень клиентов, которым необходимо разослать СМС с целью побудить их к покупке в магазине. Для этого необходимо было определить величину uplift для каждого клиента, и тем у кого эта величина окажется положительная - отправить СМС. Задача решалась в несколько этапов: 

1. На первом этапе была выполнена загрузка данных и их первичный анализ. Были выявлены проблемные вопросы в данных и обозначены пути их решения.

2. Далее, была произведёна предобработка данных, генерация признаков, удаление лишних признаков и подготовка выборок к обучению моделей. В ходе предобработки были устранены выявленные на первом этапе проблемы в данных - пропуски и недостоверные данные.

3. На третьем этапе был произведен выбор подхода к определению аплифта и выбор оптимальной модели машинного обучения. Наибольшее значение аплифта удалось получить с помощью подхода с трансформацией классой (вместе с LGBM классификатором). Используя такую модель, удалось добиться следующих значений метрик на тестовых данных:

    - uplift_at_30% = (8.2 ± 0.6)%,
    - uplift_auc_score: (0.036 ± 0.005), 
    - uplift_qini_score: (0.025 ± 0.003)

    Сложность выбора модели и подбора гиперпараметров заключалась в том, что значение аплифта было очень небольшим (не больше 10%), а от этого данные сильно "шумели", приходилось долго подбирать оптимальные гиперпараметры. 

4. Стоит также заметить, что подбор оптимальных гиперпараметров для моделей с классификатором CatBoost происходил значительно быстрее, чем в других случаях. Поэтому, если встает вопрос об исследовании пространства  гиперпараметров наиболее подробно, то лучше, скорее всего, использовать этот классификатор (при использовании ClassTranformation с CatBoostClassifier значения метрик были лишь чуть-чуть хуже, чем у модели ClassTranformation с LGBM классификатором).

# Что дальше

&ensp; &ensp;Нет сомнения в том, что можно увеличить качество модели. Это можно сделать, например, путем  более тщательного отбора признаков. Также, после выяснения того факта, что метод трансформации классов наиболее хорош для решения данной задачи, можно заняться подбором гиперпараметров более аккуратно. Ну а потом, при желании, можно и подумать о внедрении модели)