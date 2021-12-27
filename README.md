# Аспектно-ориентированная оценка тональности отзывов на рестораны

## Извлечение эксплицитных упоминаний аспектов

### Корпус
В качестве обучающих данных для задачи выделения эксплицитных аспектов мы не использовали никаких дополнительных данных, соответственно, объемы обучающего корпуса:

|data|size|
|----|----|
|train_reviews|284|
|train_aspects|4763|

### Модели и метод

Данные были размечены BIO-разметкой при помощи библиотеки `stanza`, в результате чего был получен корпус из 47243 размеченных токенов для обучения классификатора для разметки. Код для трансформирования данных и дальнейших экспериментов с классификаторами можно найти [по ссылке](https://github.com/aaaksenova/NLP_ABSA_project/blob/change/aspect_extraction_as_classification.ipynb).

В качестве собственно классификаторов были взяты несколько моделей разной архитектуры: обычный `перцептрон`, модель стохастического градиентного спуска (`SGDClassifier`), наивный байес (`MultinomialNB`) и `CRF`. Результаты классификации (взвешенные) для лучших из них представлены в таблице ниже. (**NB:** другие модели показали слишком низкое качество, чтобы даже приводить их результаты здесь). 

|model|precision|recall|f1|
|-----|---------|------|--|
|MultinomialNB|0.74|0.83|0.78|
|CRF|0.96|0.95|0.95|


### Результаты

Далее в таблице представлены результаты вывделения эксплицитных аспектов. `fm` -- full match, `pm` -- partial match.

|model|fm-precision|fm-recall|pm-ratio|fm-accuracy|pm-accuracy|
|---|---|---|---|---|---|
|`Baseline`|0.48|0.716|0.62|0.464|0.603|
|`MultinomialNB`|0.842|0.839|0.995|0.825|0.987|
|`CRF`|**0.996**|**0.938**|**0.999**|**0.986**|**0.998**|

**Анализ ошибок**

`MultinomialNB` ошибся 603 раза. Он часто классифицирует глаголы и отглагольные слова как Сервис, при том, что в оригинале они так не размечены. Кроме того, он плохо справляется с пунктуацией. Например, когда аспект в скобках. На confision матрице видно, что путаются B и I теги внутри категории.

![img](https://github.com/aaaksenova/NLP_ABSA_project/blob/change/img/NBconfusion.png)

`CRF`, как видно из таблицы с результатами, почти идеально справляется с задачей, однако и он допускает ошибки. Всего CRF сделал 138 ошибок. В принципе в некоторых местах есть спорные моменты, например, слова "еда" и "кухня" в датасете без тега food, слово "ресторан" без тега whole. Но мы пришли к выводу, что CRF разметка в этих случаях кажется нам даже более подходящей.

![img](https://github.com/aaaksenova/NLP_ABSA_project/blob/change/img/CRFconfusion.png)

P.S.: Более подробные примеры ошибок есть в [тетрадке](https://github.com/aaaksenova/NLP_ABSA_project/blob/change/aspect_extraction_as_classification.ipynb)

***[Что пробовали](https://colab.research.google.com/drive/15xomghptvQ8k_28_A7TCWBKoQiIUE6hT?usp=sharing): извлечение аспектов по словарю***

Дополнительные данные: словарь, собранный из [YARN](https://russianword.net) и [Карты слов](https://github.com/dkulagin/kartaslov) (синсеты, содержащие *пища*, *ресторан*, *интерьер*, *цена*, *официант*). Предпологалось, что так можно будет выделить не только самые частотные слова, но и непопулярные названия блюд и т.д., а слова, неотносящиеся к теме ресторанов, просто не попадутся в текстах отзывов. К сожалению, это не оправдалось, и в текстах встречается много слов не относящихся непосредственно к теме, как, например, *друг*, *трасса*, *пробка*. Кроме того, проблему составляли лемматизированные неграмматичные варианты написаний слов (эмоциональные *нравитсяяяяяяя* и т.д.), леммы которых иногда встречались в словаре. Список слов нужно было корректировать по частотности встречаемости в тексте (из-за чего теряется смысл использовать этот список), по частям речи, леммам, семантике - поскольку вышеописанный метод извлечения аспектов себя, однозначно, оправдал, работа со словарем не была продолжена.

## Оценка тональности упоминания аспекта + оценка тональности всего отзыва по категориям

Для задачи оценки тональности упоминания аспекта и отзыва по категориям мы пробовали несколько разных подходов. Кратко опишем их далее.

### Rule-based
Первый метод для определения тональности аспекта был так называемый лингвистико-инженерный способ, который далее мы будем называть `TDS`.
Код можно найти по [ссылке](https://github.com/aaaksenova/NLP_ABSA_project/blob/change/tonal_dict_syntax.ipynb).

#### Корпус
`TDS` -- это unsupervised метод, поэтому, обучающих данных мы не использовали, а работали сразу с предсказанными в первой части выделенными аспектами.

#### Метод
Суть метода заключается в том, что для классификации тональности мы используем тональный словарь (мы брали [этот](https://github.com/dkulagin/kartaslov/tree/master/dataset/kartaslovsent)) и набор правил, опирающихся на синтаксическую разметку предложения (отсюда название метода -- tonal dict + syntax). За основу для метода мы брали [доступный код](https://intellica-ai.medium.com/aspect-based-sentiment-analysis-everything-you-wanted-to-know-1be41572e238) для английского языка, который затем адаптировали для русского. 

**Анализ ошибок еще до применения модели или обреченность метода**

Тут же мы столкнулись с несколькими проблемами, которые, к сожалению, не так просто решить:

1. Основная проблема для этого метода -- это собственно модель для разметки синтаксиса, которую мы использовали. В нашем случе это была `spacy ru_core_news`. Несмотря на то, что модель сама по себе имеет высокое качество, на текстах отзывов она справляется с разметкой не совсем хорошо, поэтому для правильной работы метода извлечения тональных слов, относящихся к нашим аспектам приходилось придумывать нескоторые (не самые красивые) эвристики. 
2. В частности, т.к. этот метод плохо справлялся с би+ граммами в упоминаниях аспектов, мы решили при помощи разметки выделять только вершину сочетания и классифицировать именно ее (соотвественно в сочетании *девушка хостес* мы оставляли *девушка*). Однако раззметка сильно менялась в зависимости от контекста, часто вершины приписывались прилагательным, а не существительным, отсюда было много проблем с выделением именно нужных нам аспектов уже при классификации тональности. В итоге даже большая модель справлялась с этой задачей не лучшим образом.
3. Другая проблема, с которой мы столкнулись, -- наличие метки `both` в нашем корпусе. К сожалению, такой метод мог определить только положительные или отриуательные отзывы (категория `нейтральный` присваивалась нами, соотвественно, в случае, когда ни тот, ни другой класс не приписывался выделенному аспекту). Однако с `both` в таком методе не оченб понятно что делать и она просто не присутсвует в возможных классах для этого метода, что, конечно, влияет на качество (всего в тестовом корпусе около 90 объектов с тегом `both`)
4. Следующая трудность -- собственно тональный словарь, который мы использовали для выделения сентиментов. Т.к. он не контексто зависимый, там не было важных слов для отзывов на рестораны (что-то типа *высокий* как отрицатльная тональность для *высокая цена* и похожее). Более того, некоторые слова были указаны как положительные или отрицательные, хотя по сути своей явля.тся нейтральными (так *ресторане* классифицировалось как положительный аспект в предложении *Мы были в ресторане в субботу*, потому что в словаре *суббота* имела положительную коннотацию + тут можно увидеть неостатки разметки, т.к. по-хорошему это зависимое не должно влиять на *ресторан*). Этот недостаток мы пытались исправить подбором порога (в словаре слова имеют значение положительности и отрицательности), но в итоге эта фильтрация только портила качество, поэтому мы остановились на полном списке.

### Bert-based

#### Корпус
В качестве обучающих данных для этого задания мы так же не использовали дополнительных данных. 

#### Метод

За основу мы взяли [Entity-based sentiment analysis using BERT](https://github.com/deep-nlp-spring-2020/dialog-sent/blob/master/notebooks/9-bert-masked.ipynb). Метод заключается в маскировании токена-упоминания аспекта и предсказывании его сентимента по `[CLS]` токену предложения с маской. 

**Анализ ошибок**
![image](https://user-images.githubusercontent.com/42929264/147478637-6d02967d-bc96-4b79-9c4a-f0c34a584ca7.png)
Примеров с положительными отзывами оказалось в несколько раз больше, чем с остальными, поэтому на валидации BERT показал байес в сторону положительного класса.


### Результаты
В таблице ниже представлены результаты оценки тональности упоминания аспекта и оценки тональности всего отзыва по категориям для наших методов. `fm` -- full match, `pm` -- partial match, `overall` -- тональность отзыва

|Модель|fm-accuracy|pm-accuracy|overall|
|---|---|---|---|
|`Baseline`|0.677|**0.572**|0.524|
|`TDS`|0.573|0.33|0.408|
|`BERT`|**0.677**|0.33|**0.7267**|
