# **Text classification task**

## **Задание:**

Имеется датафрейм train.csv, содержащий 2 столбца: 
1) ***Description*** (далее переименнованный в ***'text'***) - содержащий тексты на естественном языке произвольной длины 
2) ***Label*** - определяющий класс каждого текста.

Необходимо:
1) Обучить модель классификации
2) Сделать скрипт для предсказания моделью класса фразы из test.csv и подсчёта метрик(accuracy, precision, recall, macro-f1)
Замечание: Нельзя использовать LLM не для получения эмбеддингов => для получения эмбеддингов можно воспользоваться готовыми предобученными LLM, что мы и сделаем.

## **Структура:**
Второстепенные файлы:
- train.csv - исходные данные
- 1_getting_embeddings.ipynb - код для создания датафреймов с эмбеддингами исходного и предобработанного текста
- 2_all_models.ipynb - код для построения всех тестируемых моделей и итоговой таблицы с результатами качества классификации моделей на тестовой выборке (таблица продемонстрирована ниже)
- 3_creating_best_models.ipynb - код для получения файлов лучших моделей: наилучшей по качеству и оптимальной по соотношению качество-скорость
- models_results.html - html файл с результатами с результатами качества классификации моделей на тестовой выборке (в более красивом виде)
  
### **Основные файлы:**
- **4_testing_best_models.ipynb** - скрипт для предсказания моделями классов фраз из test.csv и вывод таблиц с метриками качества моделей.
- **best_model.h5** - файл с лучшей по качеству моделью
- **fast_model.h5**- файл с оптимальной моделью по соотношению качество-скорость 

## **Решение:**

**Шаг 1: Изучение данных**

- ***Label*** - представляет из себя массив, содержащий классы текстов (всего 4 уникальных класса)
- ***text*** - представляет из себя массив, содержащий тексты произвольной длины
- Исходный размер выборки = 45381 наблюдений

Первое что необходимо сделать, это проверить данные на наличие пропусков(в том числе пустых строк) и дубликатов:
1) Количество пропусков/пустых строк = 0 
2) Количество дубликатов = 19051 - почти **42%** выборки дубликаты, которые необходимо удалить для получения репрезентативных результатов

Таким образом, общее количество наблюдений после удаления выбросов будет равно: **26330**.

Посмотрим, как распределяются классы в итоговом датафрейме:
![image](https://github.com/IlinMatvey/Text_classification/assets/112751928/94b25f6c-8702-471b-aac8-39d0e60e2910)

Как видно из гистаграммы, в данных преобладает класс 0, однако критичного дисбаланса классов не наблюдается.

**Шаг 2: Предобработка исходного текста**

Препроцессинг помогает подготовить текстовые данные к следующим этапам анализа или обработки и позволяет привести текст к единой форме. Основные этапы препроцессинга:
1) Приведение текста к нижнему регистру
2) Удаление лишних пробелов
3) Удаление стоп-слов
4) Удаление знаков препинания
5) Лемматизация токенов(слов)

В дальнейшем мы будем оценивать качество моделей классификации на исходном и предобработанном тексте для оценки влияния препроцессинга на качество итоговых моделей.

P.s. Для векторизации текста мы будем использовать уже готовые LLM модели, в которых уже зашита примитивная предобработка текста. (например в модели *universal-sentence-encoder* предложения cначала переводятся в нижний регистр и разбиваются на лексемы с помощью токенизатора Penn Treebank(PTB)). Данный этап можно считать показательным и, как мы увидим в дальнейшем, это никак значимо не повлияет на качество моделей классификации в данной задаче.

**Шаг 3: Векторизация текста: получение эмбеддингов**

Для преобразования текста в числовой формат в данном тестовом задании будут использоваться 3 предобученные LLM для создания эмбеддингов предложений:
1) **all-mpnet-base-v2** - преобразует текст произвольной длины в числовой вектор размером 768.
2) **all-MiniLM-L6-v2** - преобразует текст произвольной длины в числовой вектор размером 384.
3) **universal-sentence-encoder** - преобразует текст произвольной длины в числовой вектор размером 512.

Links:

- https://sbert.net/docs/pretrained_models.html
- https://tfhub.dev/google/universal-sentence-encoder/4

Данные модели были выбраны для сравнения из следующих соображений:
1) ***all-mpnet-base-v2*** - является лучшей моделью для создания эмбедингов предложений по оценки её качества для решения различных задач в сфере NLP
2) ***all-MiniLM-L6-v2***  - работает в 5 раз быстрее чем all-mpnet-base-v2, и тоже показывает хорошие результаты
3) ***universal-sentence-encoder*** - топовая модель от компании Google

P.s. Данные модели будут сравниваться между собой для выявления наиболее подходящей для решения поставленной задачи.

**Шаг 4: Выбор наилучшей модели классификации текстов**

Разделение данных на *обучающую*, *варидационную* и *тестовую* выборки:
1) Данные разбиваются на 2 части: тренировочная выборка - **80%**, тестовая выборка - **20 % с сохранением распределения классов относительно исходного соостношения.**
2) Тренировочная выборка с использованием метода кросс-валидации разделяется на 4 равных части, и итеративно каждая из этих частей выступает в качестве валидационной выборки, в то время как оставшиеся части формируют обучающую выборку.

Таким образом в обучении модели учавствует только **80%** данных, а **20%** используются для итогового тестирования.

P.s. После выбора оптимальных моделей для решения поставленных задач для их обучения будут использоваться **все** данные из train.csv

В качестве алгоритмов классификации были выбраны следующие модели: 
Модель | Количество блоков | Выходной слой | Функция активации 
:-------|:--------:|-------:|-------
LSTM | 64 | 4 | softmax
LSTM | 128 | 4 | softmax
LSTM | 256 | 4 | softmax
GRU | 64 | 4 | softmax
GRU | 128 | 4 | softmax
GRU | 256 | 4 | softmax

Модель | Количество нейронов скрытого слоя | Выходной слой | Функция активации 
:-------|:--------:|-------:|-------
Perceptron | 64 | 4 | softmax
Perceptron | 128 | 4 | softmax
Perceptron | 256 | 4 | softmax
Perceptron | 512 | 4 | softmax

В каждой из этих моделей используется ***2 метода борьбы с переобучением***:
1) Наличие метода ***Dropout*** со значение 0.5 перед выходным слоем
2) Наличие ***EarlyStopping***

Таблица с результатами оценки моделей классификации на тестовой выборке в порядке убывания значения f1-macro:

<table id="T_0d63d">
  <thead>
    <tr>
      <th class="blank level0" >&nbsp;</th>
      <th id="T_0d63d_level0_col0" class="col_heading level0 col0" >embeddings_model</th>
      <th id="T_0d63d_level0_col1" class="col_heading level0 col1" >classification_model</th>
      <th id="T_0d63d_level0_col2" class="col_heading level0 col2" >preprocessing_has</th>
      <th id="T_0d63d_level0_col3" class="col_heading level0 col3" >accuracy</th>
      <th id="T_0d63d_level0_col4" class="col_heading level0 col4" >precision_macro</th>
      <th id="T_0d63d_level0_col5" class="col_heading level0 col5" >recall_macro</th>
      <th id="T_0d63d_level0_col6" class="col_heading level0 col6" >f1_macro</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th id="T_0d63d_level0_row0" class="row_heading level0 row0" >0</th>
      <td id="T_0d63d_row0_col0" class="data row0 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row0_col1" class="data row0 col1" >Perceptron_512</td>
      <td id="T_0d63d_row0_col2" class="data row0 col2" >no</td>
      <td id="T_0d63d_row0_col3" class="data row0 col3" >0.957650</td>
      <td id="T_0d63d_row0_col4" class="data row0 col4" >0.957690</td>
      <td id="T_0d63d_row0_col5" class="data row0 col5" >0.956160</td>
      <td id="T_0d63d_row0_col6" class="data row0 col6" >0.956850</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row1" class="row_heading level0 row1" >1</th>
      <td id="T_0d63d_row1_col0" class="data row1 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row1_col1" class="data row1 col1" >Perceptron_256</td>
      <td id="T_0d63d_row1_col2" class="data row1 col2" >no</td>
      <td id="T_0d63d_row1_col3" class="data row1 col3" >0.955560</td>
      <td id="T_0d63d_row1_col4" class="data row1 col4" >0.956240</td>
      <td id="T_0d63d_row1_col5" class="data row1 col5" >0.953390</td>
      <td id="T_0d63d_row1_col6" class="data row1 col6" >0.954720</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row2" class="row_heading level0 row2" >2</th>
      <td id="T_0d63d_row2_col0" class="data row2 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row2_col1" class="data row2 col1" >Perceptron_512</td>
      <td id="T_0d63d_row2_col2" class="data row2 col2" >yes</td>
      <td id="T_0d63d_row2_col3" class="data row2 col3" >0.954420</td>
      <td id="T_0d63d_row2_col4" class="data row2 col4" >0.953640</td>
      <td id="T_0d63d_row2_col5" class="data row2 col5" >0.953000</td>
      <td id="T_0d63d_row2_col6" class="data row2 col6" >0.953310</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row3" class="row_heading level0 row3" >3</th>
      <td id="T_0d63d_row3_col0" class="data row3 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row3_col1" class="data row3 col1" >Perceptron_256</td>
      <td id="T_0d63d_row3_col2" class="data row3 col2" >yes</td>
      <td id="T_0d63d_row3_col3" class="data row3 col3" >0.953850</td>
      <td id="T_0d63d_row3_col4" class="data row3 col4" >0.953860</td>
      <td id="T_0d63d_row3_col5" class="data row3 col5" >0.952300</td>
      <td id="T_0d63d_row3_col6" class="data row3 col6" >0.953040</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row4" class="row_heading level0 row4" >4</th>
      <td id="T_0d63d_row4_col0" class="data row4 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row4_col1" class="data row4 col1" >Perceptron_64</td>
      <td id="T_0d63d_row4_col2" class="data row4 col2" >no</td>
      <td id="T_0d63d_row4_col3" class="data row4 col3" >0.952720</td>
      <td id="T_0d63d_row4_col4" class="data row4 col4" >0.953840</td>
      <td id="T_0d63d_row4_col5" class="data row4 col5" >0.950600</td>
      <td id="T_0d63d_row4_col6" class="data row4 col6" >0.952130</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row5" class="row_heading level0 row5" >5</th>
      <td id="T_0d63d_row5_col0" class="data row5 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row5_col1" class="data row5 col1" >Perceptron_128</td>
      <td id="T_0d63d_row5_col2" class="data row5 col2" >no</td>
      <td id="T_0d63d_row5_col3" class="data row5 col3" >0.952340</td>
      <td id="T_0d63d_row5_col4" class="data row5 col4" >0.951820</td>
      <td id="T_0d63d_row5_col5" class="data row5 col5" >0.951720</td>
      <td id="T_0d63d_row5_col6" class="data row5 col6" >0.951750</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row6" class="row_heading level0 row6" >6</th>
      <td id="T_0d63d_row6_col0" class="data row6 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row6_col1" class="data row6 col1" >LSTM_64</td>
      <td id="T_0d63d_row6_col2" class="data row6 col2" >no</td>
      <td id="T_0d63d_row6_col3" class="data row6 col3" >0.952150</td>
      <td id="T_0d63d_row6_col4" class="data row6 col4" >0.953870</td>
      <td id="T_0d63d_row6_col5" class="data row6 col5" >0.949390</td>
      <td id="T_0d63d_row6_col6" class="data row6 col6" >0.951500</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row7" class="row_heading level0 row7" >7</th>
      <td id="T_0d63d_row7_col0" class="data row7 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row7_col1" class="data row7 col1" >Perceptron_512</td>
      <td id="T_0d63d_row7_col2" class="data row7 col2" >no</td>
      <td id="T_0d63d_row7_col3" class="data row7 col3" >0.952340</td>
      <td id="T_0d63d_row7_col4" class="data row7 col4" >0.951250</td>
      <td id="T_0d63d_row7_col5" class="data row7 col5" >0.951970</td>
      <td id="T_0d63d_row7_col6" class="data row7 col6" >0.951490</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row8" class="row_heading level0 row8" >8</th>
      <td id="T_0d63d_row8_col0" class="data row8 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row8_col1" class="data row8 col1" >Perceptron_128</td>
      <td id="T_0d63d_row8_col2" class="data row8 col2" >yes</td>
      <td id="T_0d63d_row8_col3" class="data row8 col3" >0.952150</td>
      <td id="T_0d63d_row8_col4" class="data row8 col4" >0.951850</td>
      <td id="T_0d63d_row8_col5" class="data row8 col5" >0.950530</td>
      <td id="T_0d63d_row8_col6" class="data row8 col6" >0.951140</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row9" class="row_heading level0 row9" >9</th>
      <td id="T_0d63d_row9_col0" class="data row9 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row9_col1" class="data row9 col1" >LSTM_256</td>
      <td id="T_0d63d_row9_col2" class="data row9 col2" >no</td>
      <td id="T_0d63d_row9_col3" class="data row9 col3" >0.951200</td>
      <td id="T_0d63d_row9_col4" class="data row9 col4" >0.949800</td>
      <td id="T_0d63d_row9_col5" class="data row9 col5" >0.950770</td>
      <td id="T_0d63d_row9_col6" class="data row9 col6" >0.950210</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row10" class="row_heading level0 row10" >10</th>
      <td id="T_0d63d_row10_col0" class="data row10 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row10_col1" class="data row10 col1" >LSTM_128</td>
      <td id="T_0d63d_row10_col2" class="data row10 col2" >yes</td>
      <td id="T_0d63d_row10_col3" class="data row10 col3" >0.950820</td>
      <td id="T_0d63d_row10_col4" class="data row10 col4" >0.950400</td>
      <td id="T_0d63d_row10_col5" class="data row10 col5" >0.949850</td>
      <td id="T_0d63d_row10_col6" class="data row10 col6" >0.950120</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row11" class="row_heading level0 row11" >11</th>
      <td id="T_0d63d_row11_col0" class="data row11 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row11_col1" class="data row11 col1" >Perceptron_512</td>
      <td id="T_0d63d_row11_col2" class="data row11 col2" >no</td>
      <td id="T_0d63d_row11_col3" class="data row11 col3" >0.950630</td>
      <td id="T_0d63d_row11_col4" class="data row11 col4" >0.951400</td>
      <td id="T_0d63d_row11_col5" class="data row11 col5" >0.948910</td>
      <td id="T_0d63d_row11_col6" class="data row11 col6" >0.950090</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row12" class="row_heading level0 row12" >12</th>
      <td id="T_0d63d_row12_col0" class="data row12 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row12_col1" class="data row12 col1" >Perceptron_256</td>
      <td id="T_0d63d_row12_col2" class="data row12 col2" >no</td>
      <td id="T_0d63d_row12_col3" class="data row12 col3" >0.950440</td>
      <td id="T_0d63d_row12_col4" class="data row12 col4" >0.951880</td>
      <td id="T_0d63d_row12_col5" class="data row12 col5" >0.948360</td>
      <td id="T_0d63d_row12_col6" class="data row12 col6" >0.950040</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row13" class="row_heading level0 row13" >13</th>
      <td id="T_0d63d_row13_col0" class="data row13 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row13_col1" class="data row13 col1" >GRU_64</td>
      <td id="T_0d63d_row13_col2" class="data row13 col2" >no</td>
      <td id="T_0d63d_row13_col3" class="data row13 col3" >0.950820</td>
      <td id="T_0d63d_row13_col4" class="data row13 col4" >0.951610</td>
      <td id="T_0d63d_row13_col5" class="data row13 col5" >0.948340</td>
      <td id="T_0d63d_row13_col6" class="data row13 col6" >0.949900</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row14" class="row_heading level0 row14" >14</th>
      <td id="T_0d63d_row14_col0" class="data row14 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row14_col1" class="data row14 col1" >Perceptron_64</td>
      <td id="T_0d63d_row14_col2" class="data row14 col2" >yes</td>
      <td id="T_0d63d_row14_col3" class="data row14 col3" >0.950820</td>
      <td id="T_0d63d_row14_col4" class="data row14 col4" >0.950440</td>
      <td id="T_0d63d_row14_col5" class="data row14 col5" >0.949390</td>
      <td id="T_0d63d_row14_col6" class="data row14 col6" >0.949840</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row15" class="row_heading level0 row15" >15</th>
      <td id="T_0d63d_row15_col0" class="data row15 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row15_col1" class="data row15 col1" >Perceptron_128</td>
      <td id="T_0d63d_row15_col2" class="data row15 col2" >no</td>
      <td id="T_0d63d_row15_col3" class="data row15 col3" >0.950440</td>
      <td id="T_0d63d_row15_col4" class="data row15 col4" >0.951100</td>
      <td id="T_0d63d_row15_col5" class="data row15 col5" >0.948700</td>
      <td id="T_0d63d_row15_col6" class="data row15 col6" >0.949810</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row16" class="row_heading level0 row16" >16</th>
      <td id="T_0d63d_row16_col0" class="data row16 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row16_col1" class="data row16 col1" >Perceptron_256</td>
      <td id="T_0d63d_row16_col2" class="data row16 col2" >no</td>
      <td id="T_0d63d_row16_col3" class="data row16 col3" >0.950630</td>
      <td id="T_0d63d_row16_col4" class="data row16 col4" >0.951400</td>
      <td id="T_0d63d_row16_col5" class="data row16 col5" >0.948340</td>
      <td id="T_0d63d_row16_col6" class="data row16 col6" >0.949720</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row17" class="row_heading level0 row17" >17</th>
      <td id="T_0d63d_row17_col0" class="data row17 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row17_col1" class="data row17 col1" >LSTM_256</td>
      <td id="T_0d63d_row17_col2" class="data row17 col2" >yes</td>
      <td id="T_0d63d_row17_col3" class="data row17 col3" >0.950630</td>
      <td id="T_0d63d_row17_col4" class="data row17 col4" >0.950600</td>
      <td id="T_0d63d_row17_col5" class="data row17 col5" >0.948870</td>
      <td id="T_0d63d_row17_col6" class="data row17 col6" >0.949710</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row18" class="row_heading level0 row18" >18</th>
      <td id="T_0d63d_row18_col0" class="data row18 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row18_col1" class="data row18 col1" >LSTM_128</td>
      <td id="T_0d63d_row18_col2" class="data row18 col2" >no</td>
      <td id="T_0d63d_row18_col3" class="data row18 col3" >0.950440</td>
      <td id="T_0d63d_row18_col4" class="data row18 col4" >0.949690</td>
      <td id="T_0d63d_row18_col5" class="data row18 col5" >0.949570</td>
      <td id="T_0d63d_row18_col6" class="data row18 col6" >0.949620</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row19" class="row_heading level0 row19" >19</th>
      <td id="T_0d63d_row19_col0" class="data row19 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row19_col1" class="data row19 col1" >GRU_256</td>
      <td id="T_0d63d_row19_col2" class="data row19 col2" >no</td>
      <td id="T_0d63d_row19_col3" class="data row19 col3" >0.950250</td>
      <td id="T_0d63d_row19_col4" class="data row19 col4" >0.951380</td>
      <td id="T_0d63d_row19_col5" class="data row19 col5" >0.947510</td>
      <td id="T_0d63d_row19_col6" class="data row19 col6" >0.949360</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row20" class="row_heading level0 row20" >20</th>
      <td id="T_0d63d_row20_col0" class="data row20 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row20_col1" class="data row20 col1" >GRU_64</td>
      <td id="T_0d63d_row20_col2" class="data row20 col2" >yes</td>
      <td id="T_0d63d_row20_col3" class="data row20 col3" >0.949490</td>
      <td id="T_0d63d_row20_col4" class="data row20 col4" >0.949300</td>
      <td id="T_0d63d_row20_col5" class="data row20 col5" >0.948070</td>
      <td id="T_0d63d_row20_col6" class="data row20 col6" >0.948650</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row21" class="row_heading level0 row21" >21</th>
      <td id="T_0d63d_row21_col0" class="data row21 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row21_col1" class="data row21 col1" >Perceptron_512</td>
      <td id="T_0d63d_row21_col2" class="data row21 col2" >yes</td>
      <td id="T_0d63d_row21_col3" class="data row21 col3" >0.949110</td>
      <td id="T_0d63d_row21_col4" class="data row21 col4" >0.950480</td>
      <td id="T_0d63d_row21_col5" class="data row21 col5" >0.946710</td>
      <td id="T_0d63d_row21_col6" class="data row21 col6" >0.948540</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row22" class="row_heading level0 row22" >22</th>
      <td id="T_0d63d_row22_col0" class="data row22 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row22_col1" class="data row22 col1" >LSTM_64</td>
      <td id="T_0d63d_row22_col2" class="data row22 col2" >yes</td>
      <td id="T_0d63d_row22_col3" class="data row22 col3" >0.949300</td>
      <td id="T_0d63d_row22_col4" class="data row22 col4" >0.950060</td>
      <td id="T_0d63d_row22_col5" class="data row22 col5" >0.946430</td>
      <td id="T_0d63d_row22_col6" class="data row22 col6" >0.948100</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row23" class="row_heading level0 row23" >23</th>
      <td id="T_0d63d_row23_col0" class="data row23 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row23_col1" class="data row23 col1" >GRU_256</td>
      <td id="T_0d63d_row23_col2" class="data row23 col2" >yes</td>
      <td id="T_0d63d_row23_col3" class="data row23 col3" >0.948730</td>
      <td id="T_0d63d_row23_col4" class="data row23 col4" >0.949920</td>
      <td id="T_0d63d_row23_col5" class="data row23 col5" >0.945720</td>
      <td id="T_0d63d_row23_col6" class="data row23 col6" >0.947750</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row24" class="row_heading level0 row24" >24</th>
      <td id="T_0d63d_row24_col0" class="data row24 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row24_col1" class="data row24 col1" >GRU_128</td>
      <td id="T_0d63d_row24_col2" class="data row24 col2" >yes</td>
      <td id="T_0d63d_row24_col3" class="data row24 col3" >0.948540</td>
      <td id="T_0d63d_row24_col4" class="data row24 col4" >0.949120</td>
      <td id="T_0d63d_row24_col5" class="data row24 col5" >0.946370</td>
      <td id="T_0d63d_row24_col6" class="data row24 col6" >0.947700</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row25" class="row_heading level0 row25" >25</th>
      <td id="T_0d63d_row25_col0" class="data row25 col0" >all_mpnet_base_v2</td>
      <td id="T_0d63d_row25_col1" class="data row25 col1" >GRU_128</td>
      <td id="T_0d63d_row25_col2" class="data row25 col2" >no</td>
      <td id="T_0d63d_row25_col3" class="data row25 col3" >0.948350</td>
      <td id="T_0d63d_row25_col4" class="data row25 col4" >0.950780</td>
      <td id="T_0d63d_row25_col5" class="data row25 col5" >0.944790</td>
      <td id="T_0d63d_row25_col6" class="data row25 col6" >0.947610</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row26" class="row_heading level0 row26" >26</th>
      <td id="T_0d63d_row26_col0" class="data row26 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row26_col1" class="data row26 col1" >Perceptron_256</td>
      <td id="T_0d63d_row26_col2" class="data row26 col2" >yes</td>
      <td id="T_0d63d_row26_col3" class="data row26 col3" >0.947590</td>
      <td id="T_0d63d_row26_col4" class="data row26 col4" >0.946050</td>
      <td id="T_0d63d_row26_col5" class="data row26 col5" >0.947260</td>
      <td id="T_0d63d_row26_col6" class="data row26 col6" >0.946520</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row27" class="row_heading level0 row27" >27</th>
      <td id="T_0d63d_row27_col0" class="data row27 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row27_col1" class="data row27 col1" >Perceptron_64</td>
      <td id="T_0d63d_row27_col2" class="data row27 col2" >no</td>
      <td id="T_0d63d_row27_col3" class="data row27 col3" >0.946450</td>
      <td id="T_0d63d_row27_col4" class="data row27 col4" >0.947600</td>
      <td id="T_0d63d_row27_col5" class="data row27 col5" >0.944070</td>
      <td id="T_0d63d_row27_col6" class="data row27 col6" >0.945730</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row28" class="row_heading level0 row28" >28</th>
      <td id="T_0d63d_row28_col0" class="data row28 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row28_col1" class="data row28 col1" >Perceptron_128</td>
      <td id="T_0d63d_row28_col2" class="data row28 col2" >no</td>
      <td id="T_0d63d_row28_col3" class="data row28 col3" >0.946260</td>
      <td id="T_0d63d_row28_col4" class="data row28 col4" >0.948080</td>
      <td id="T_0d63d_row28_col5" class="data row28 col5" >0.943460</td>
      <td id="T_0d63d_row28_col6" class="data row28 col6" >0.945610</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row29" class="row_heading level0 row29" >29</th>
      <td id="T_0d63d_row29_col0" class="data row29 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row29_col1" class="data row29 col1" >Perceptron_256</td>
      <td id="T_0d63d_row29_col2" class="data row29 col2" >yes</td>
      <td id="T_0d63d_row29_col3" class="data row29 col3" >0.944170</td>
      <td id="T_0d63d_row29_col4" class="data row29 col4" >0.944480</td>
      <td id="T_0d63d_row29_col5" class="data row29 col5" >0.942660</td>
      <td id="T_0d63d_row29_col6" class="data row29 col6" >0.943520</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row30" class="row_heading level0 row30" >30</th>
      <td id="T_0d63d_row30_col0" class="data row30 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row30_col1" class="data row30 col1" >LSTM_64</td>
      <td id="T_0d63d_row30_col2" class="data row30 col2" >no</td>
      <td id="T_0d63d_row30_col3" class="data row30 col3" >0.944170</td>
      <td id="T_0d63d_row30_col4" class="data row30 col4" >0.944850</td>
      <td id="T_0d63d_row30_col5" class="data row30 col5" >0.942390</td>
      <td id="T_0d63d_row30_col6" class="data row30 col6" >0.943490</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row31" class="row_heading level0 row31" >31</th>
      <td id="T_0d63d_row31_col0" class="data row31 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row31_col1" class="data row31 col1" >Perceptron_512</td>
      <td id="T_0d63d_row31_col2" class="data row31 col2" >yes</td>
      <td id="T_0d63d_row31_col3" class="data row31 col3" >0.943600</td>
      <td id="T_0d63d_row31_col4" class="data row31 col4" >0.945300</td>
      <td id="T_0d63d_row31_col5" class="data row31 col5" >0.940980</td>
      <td id="T_0d63d_row31_col6" class="data row31 col6" >0.943030</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row32" class="row_heading level0 row32" >32</th>
      <td id="T_0d63d_row32_col0" class="data row32 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row32_col1" class="data row32 col1" >GRU_256</td>
      <td id="T_0d63d_row32_col2" class="data row32 col2" >no</td>
      <td id="T_0d63d_row32_col3" class="data row32 col3" >0.943600</td>
      <td id="T_0d63d_row32_col4" class="data row32 col4" >0.944880</td>
      <td id="T_0d63d_row32_col5" class="data row32 col5" >0.940970</td>
      <td id="T_0d63d_row32_col6" class="data row32 col6" >0.942840</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row33" class="row_heading level0 row33" >33</th>
      <td id="T_0d63d_row33_col0" class="data row33 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row33_col1" class="data row33 col1" >LSTM_256</td>
      <td id="T_0d63d_row33_col2" class="data row33 col2" >yes</td>
      <td id="T_0d63d_row33_col3" class="data row33 col3" >0.943410</td>
      <td id="T_0d63d_row33_col4" class="data row33 col4" >0.944000</td>
      <td id="T_0d63d_row33_col5" class="data row33 col5" >0.941400</td>
      <td id="T_0d63d_row33_col6" class="data row33 col6" >0.942540</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row34" class="row_heading level0 row34" >34</th>
      <td id="T_0d63d_row34_col0" class="data row34 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row34_col1" class="data row34 col1" >Perceptron_64</td>
      <td id="T_0d63d_row34_col2" class="data row34 col2" >no</td>
      <td id="T_0d63d_row34_col3" class="data row34 col3" >0.942840</td>
      <td id="T_0d63d_row34_col4" class="data row34 col4" >0.942830</td>
      <td id="T_0d63d_row34_col5" class="data row34 col5" >0.942260</td>
      <td id="T_0d63d_row34_col6" class="data row34 col6" >0.942450</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row35" class="row_heading level0 row35" >35</th>
      <td id="T_0d63d_row35_col0" class="data row35 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row35_col1" class="data row35 col1" >LSTM_128</td>
      <td id="T_0d63d_row35_col2" class="data row35 col2" >no</td>
      <td id="T_0d63d_row35_col3" class="data row35 col3" >0.943410</td>
      <td id="T_0d63d_row35_col4" class="data row35 col4" >0.943170</td>
      <td id="T_0d63d_row35_col5" class="data row35 col5" >0.942090</td>
      <td id="T_0d63d_row35_col6" class="data row35 col6" >0.942450</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row36" class="row_heading level0 row36" >36</th>
      <td id="T_0d63d_row36_col0" class="data row36 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row36_col1" class="data row36 col1" >LSTM_128</td>
      <td id="T_0d63d_row36_col2" class="data row36 col2" >no</td>
      <td id="T_0d63d_row36_col3" class="data row36 col3" >0.942460</td>
      <td id="T_0d63d_row36_col4" class="data row36 col4" >0.942560</td>
      <td id="T_0d63d_row36_col5" class="data row36 col5" >0.941940</td>
      <td id="T_0d63d_row36_col6" class="data row36 col6" >0.942230</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row37" class="row_heading level0 row37" >37</th>
      <td id="T_0d63d_row37_col0" class="data row37 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row37_col1" class="data row37 col1" >GRU_64</td>
      <td id="T_0d63d_row37_col2" class="data row37 col2" >no</td>
      <td id="T_0d63d_row37_col3" class="data row37 col3" >0.942650</td>
      <td id="T_0d63d_row37_col4" class="data row37 col4" >0.943360</td>
      <td id="T_0d63d_row37_col5" class="data row37 col5" >0.941020</td>
      <td id="T_0d63d_row37_col6" class="data row37 col6" >0.941980</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row38" class="row_heading level0 row38" >38</th>
      <td id="T_0d63d_row38_col0" class="data row38 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row38_col1" class="data row38 col1" >GRU_256</td>
      <td id="T_0d63d_row38_col2" class="data row38 col2" >yes</td>
      <td id="T_0d63d_row38_col3" class="data row38 col3" >0.942840</td>
      <td id="T_0d63d_row38_col4" class="data row38 col4" >0.943030</td>
      <td id="T_0d63d_row38_col5" class="data row38 col5" >0.940740</td>
      <td id="T_0d63d_row38_col6" class="data row38 col6" >0.941840</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row39" class="row_heading level0 row39" >39</th>
      <td id="T_0d63d_row39_col0" class="data row39 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row39_col1" class="data row39 col1" >LSTM_256</td>
      <td id="T_0d63d_row39_col2" class="data row39 col2" >no</td>
      <td id="T_0d63d_row39_col3" class="data row39 col3" >0.942460</td>
      <td id="T_0d63d_row39_col4" class="data row39 col4" >0.943010</td>
      <td id="T_0d63d_row39_col5" class="data row39 col5" >0.940860</td>
      <td id="T_0d63d_row39_col6" class="data row39 col6" >0.941830</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row40" class="row_heading level0 row40" >40</th>
      <td id="T_0d63d_row40_col0" class="data row40 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row40_col1" class="data row40 col1" >Perceptron_128</td>
      <td id="T_0d63d_row40_col2" class="data row40 col2" >yes</td>
      <td id="T_0d63d_row40_col3" class="data row40 col3" >0.942650</td>
      <td id="T_0d63d_row40_col4" class="data row40 col4" >0.941980</td>
      <td id="T_0d63d_row40_col5" class="data row40 col5" >0.941510</td>
      <td id="T_0d63d_row40_col6" class="data row40 col6" >0.941690</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row41" class="row_heading level0 row41" >41</th>
      <td id="T_0d63d_row41_col0" class="data row41 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row41_col1" class="data row41 col1" >GRU_128</td>
      <td id="T_0d63d_row41_col2" class="data row41 col2" >yes</td>
      <td id="T_0d63d_row41_col3" class="data row41 col3" >0.942080</td>
      <td id="T_0d63d_row41_col4" class="data row41 col4" >0.944380</td>
      <td id="T_0d63d_row41_col5" class="data row41 col5" >0.938080</td>
      <td id="T_0d63d_row41_col6" class="data row41 col6" >0.941040</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row42" class="row_heading level0 row42" >42</th>
      <td id="T_0d63d_row42_col0" class="data row42 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row42_col1" class="data row42 col1" >LSTM_256</td>
      <td id="T_0d63d_row42_col2" class="data row42 col2" >no</td>
      <td id="T_0d63d_row42_col3" class="data row42 col3" >0.941130</td>
      <td id="T_0d63d_row42_col4" class="data row42 col4" >0.940270</td>
      <td id="T_0d63d_row42_col5" class="data row42 col5" >0.941840</td>
      <td id="T_0d63d_row42_col6" class="data row42 col6" >0.941010</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row43" class="row_heading level0 row43" >43</th>
      <td id="T_0d63d_row43_col0" class="data row43 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row43_col1" class="data row43 col1" >GRU_128</td>
      <td id="T_0d63d_row43_col2" class="data row43 col2" >no</td>
      <td id="T_0d63d_row43_col3" class="data row43 col3" >0.941700</td>
      <td id="T_0d63d_row43_col4" class="data row43 col4" >0.944140</td>
      <td id="T_0d63d_row43_col5" class="data row43 col5" >0.938010</td>
      <td id="T_0d63d_row43_col6" class="data row43 col6" >0.940790</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row44" class="row_heading level0 row44" >44</th>
      <td id="T_0d63d_row44_col0" class="data row44 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row44_col1" class="data row44 col1" >GRU_64</td>
      <td id="T_0d63d_row44_col2" class="data row44 col2" >no</td>
      <td id="T_0d63d_row44_col3" class="data row44 col3" >0.940370</td>
      <td id="T_0d63d_row44_col4" class="data row44 col4" >0.941110</td>
      <td id="T_0d63d_row44_col5" class="data row44 col5" >0.939330</td>
      <td id="T_0d63d_row44_col6" class="data row44 col6" >0.940130</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row45" class="row_heading level0 row45" >45</th>
      <td id="T_0d63d_row45_col0" class="data row45 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row45_col1" class="data row45 col1" >LSTM_64</td>
      <td id="T_0d63d_row45_col2" class="data row45 col2" >yes</td>
      <td id="T_0d63d_row45_col3" class="data row45 col3" >0.940940</td>
      <td id="T_0d63d_row45_col4" class="data row45 col4" >0.941140</td>
      <td id="T_0d63d_row45_col5" class="data row45 col5" >0.939060</td>
      <td id="T_0d63d_row45_col6" class="data row45 col6" >0.940030</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row46" class="row_heading level0 row46" >46</th>
      <td id="T_0d63d_row46_col0" class="data row46 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row46_col1" class="data row46 col1" >LSTM_64</td>
      <td id="T_0d63d_row46_col2" class="data row46 col2" >no</td>
      <td id="T_0d63d_row46_col3" class="data row46 col3" >0.939990</td>
      <td id="T_0d63d_row46_col4" class="data row46 col4" >0.939230</td>
      <td id="T_0d63d_row46_col5" class="data row46 col5" >0.940440</td>
      <td id="T_0d63d_row46_col6" class="data row46 col6" >0.939780</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row47" class="row_heading level0 row47" >47</th>
      <td id="T_0d63d_row47_col0" class="data row47 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row47_col1" class="data row47 col1" >Perceptron_64</td>
      <td id="T_0d63d_row47_col2" class="data row47 col2" >yes</td>
      <td id="T_0d63d_row47_col3" class="data row47 col3" >0.940180</td>
      <td id="T_0d63d_row47_col4" class="data row47 col4" >0.940150</td>
      <td id="T_0d63d_row47_col5" class="data row47 col5" >0.939180</td>
      <td id="T_0d63d_row47_col6" class="data row47 col6" >0.939510</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row48" class="row_heading level0 row48" >48</th>
      <td id="T_0d63d_row48_col0" class="data row48 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row48_col1" class="data row48 col1" >LSTM_256</td>
      <td id="T_0d63d_row48_col2" class="data row48 col2" >yes</td>
      <td id="T_0d63d_row48_col3" class="data row48 col3" >0.939800</td>
      <td id="T_0d63d_row48_col4" class="data row48 col4" >0.939580</td>
      <td id="T_0d63d_row48_col5" class="data row48 col5" >0.938980</td>
      <td id="T_0d63d_row48_col6" class="data row48 col6" >0.939240</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row49" class="row_heading level0 row49" >49</th>
      <td id="T_0d63d_row49_col0" class="data row49 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row49_col1" class="data row49 col1" >LSTM_128</td>
      <td id="T_0d63d_row49_col2" class="data row49 col2" >yes</td>
      <td id="T_0d63d_row49_col3" class="data row49 col3" >0.939990</td>
      <td id="T_0d63d_row49_col4" class="data row49 col4" >0.940880</td>
      <td id="T_0d63d_row49_col5" class="data row49 col5" >0.937730</td>
      <td id="T_0d63d_row49_col6" class="data row49 col6" >0.939220</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row50" class="row_heading level0 row50" >50</th>
      <td id="T_0d63d_row50_col0" class="data row50 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row50_col1" class="data row50 col1" >GRU_256</td>
      <td id="T_0d63d_row50_col2" class="data row50 col2" >no</td>
      <td id="T_0d63d_row50_col3" class="data row50 col3" >0.939610</td>
      <td id="T_0d63d_row50_col4" class="data row50 col4" >0.941720</td>
      <td id="T_0d63d_row50_col5" class="data row50 col5" >0.936920</td>
      <td id="T_0d63d_row50_col6" class="data row50 col6" >0.939190</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row51" class="row_heading level0 row51" >51</th>
      <td id="T_0d63d_row51_col0" class="data row51 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row51_col1" class="data row51 col1" >Perceptron_128</td>
      <td id="T_0d63d_row51_col2" class="data row51 col2" >yes</td>
      <td id="T_0d63d_row51_col3" class="data row51 col3" >0.939610</td>
      <td id="T_0d63d_row51_col4" class="data row51 col4" >0.941850</td>
      <td id="T_0d63d_row51_col5" class="data row51 col5" >0.936710</td>
      <td id="T_0d63d_row51_col6" class="data row51 col6" >0.939120</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row52" class="row_heading level0 row52" >52</th>
      <td id="T_0d63d_row52_col0" class="data row52 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row52_col1" class="data row52 col1" >GRU_128</td>
      <td id="T_0d63d_row52_col2" class="data row52 col2" >no</td>
      <td id="T_0d63d_row52_col3" class="data row52 col3" >0.939230</td>
      <td id="T_0d63d_row52_col4" class="data row52 col4" >0.938900</td>
      <td id="T_0d63d_row52_col5" class="data row52 col5" >0.939160</td>
      <td id="T_0d63d_row52_col6" class="data row52 col6" >0.938980</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row53" class="row_heading level0 row53" >53</th>
      <td id="T_0d63d_row53_col0" class="data row53 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row53_col1" class="data row53 col1" >LSTM_128</td>
      <td id="T_0d63d_row53_col2" class="data row53 col2" >yes</td>
      <td id="T_0d63d_row53_col3" class="data row53 col3" >0.938470</td>
      <td id="T_0d63d_row53_col4" class="data row53 col4" >0.938360</td>
      <td id="T_0d63d_row53_col5" class="data row53 col5" >0.938230</td>
      <td id="T_0d63d_row53_col6" class="data row53 col6" >0.938270</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row54" class="row_heading level0 row54" >54</th>
      <td id="T_0d63d_row54_col0" class="data row54 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row54_col1" class="data row54 col1" >Perceptron_64</td>
      <td id="T_0d63d_row54_col2" class="data row54 col2" >yes</td>
      <td id="T_0d63d_row54_col3" class="data row54 col3" >0.938660</td>
      <td id="T_0d63d_row54_col4" class="data row54 col4" >0.941250</td>
      <td id="T_0d63d_row54_col5" class="data row54 col5" >0.935240</td>
      <td id="T_0d63d_row54_col6" class="data row54 col6" >0.938100</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row55" class="row_heading level0 row55" >55</th>
      <td id="T_0d63d_row55_col0" class="data row55 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row55_col1" class="data row55 col1" >LSTM_64</td>
      <td id="T_0d63d_row55_col2" class="data row55 col2" >yes</td>
      <td id="T_0d63d_row55_col3" class="data row55 col3" >0.937900</td>
      <td id="T_0d63d_row55_col4" class="data row55 col4" >0.938080</td>
      <td id="T_0d63d_row55_col5" class="data row55 col5" >0.937610</td>
      <td id="T_0d63d_row55_col6" class="data row55 col6" >0.937780</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row56" class="row_heading level0 row56" >56</th>
      <td id="T_0d63d_row56_col0" class="data row56 col0" >all_MiniLM_L6_v2</td>
      <td id="T_0d63d_row56_col1" class="data row56 col1" >GRU_64</td>
      <td id="T_0d63d_row56_col2" class="data row56 col2" >yes</td>
      <td id="T_0d63d_row56_col3" class="data row56 col3" >0.938280</td>
      <td id="T_0d63d_row56_col4" class="data row56 col4" >0.937280</td>
      <td id="T_0d63d_row56_col5" class="data row56 col5" >0.937870</td>
      <td id="T_0d63d_row56_col6" class="data row56 col6" >0.937490</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row57" class="row_heading level0 row57" >57</th>
      <td id="T_0d63d_row57_col0" class="data row57 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row57_col1" class="data row57 col1" >GRU_64</td>
      <td id="T_0d63d_row57_col2" class="data row57 col2" >yes</td>
      <td id="T_0d63d_row57_col3" class="data row57 col3" >0.937710</td>
      <td id="T_0d63d_row57_col4" class="data row57 col4" >0.938730</td>
      <td id="T_0d63d_row57_col5" class="data row57 col5" >0.936170</td>
      <td id="T_0d63d_row57_col6" class="data row57 col6" >0.937360</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row58" class="row_heading level0 row58" >58</th>
      <td id="T_0d63d_row58_col0" class="data row58 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row58_col1" class="data row58 col1" >GRU_256</td>
      <td id="T_0d63d_row58_col2" class="data row58 col2" >yes</td>
      <td id="T_0d63d_row58_col3" class="data row58 col3" >0.936190</td>
      <td id="T_0d63d_row58_col4" class="data row58 col4" >0.937860</td>
      <td id="T_0d63d_row58_col5" class="data row58 col5" >0.933900</td>
      <td id="T_0d63d_row58_col6" class="data row58 col6" >0.935760</td>
    </tr>
    <tr>
      <th id="T_0d63d_level0_row59" class="row_heading level0 row59" >59</th>
      <td id="T_0d63d_row59_col0" class="data row59 col0" >Universal-sentence-encoder</td>
      <td id="T_0d63d_row59_col1" class="data row59 col1" >GRU_128</td>
      <td id="T_0d63d_row59_col2" class="data row59 col2" >yes</td>
      <td id="T_0d63d_row59_col3" class="data row59 col3" >0.933920</td>
      <td id="T_0d63d_row59_col4" class="data row59 col4" >0.932250</td>
      <td id="T_0d63d_row59_col5" class="data row59 col5" >0.934480</td>
      <td id="T_0d63d_row59_col6" class="data row59 col6" >0.933310</td>
    </tr>
  </tbody>
</table>


Из таблицы результатов можно сделать следующие **выводы**:
1) Модель *all_mpnet_base_v2* показывает наилучшие результаты в алгоритмах классификации, что в целом и следовало ожидать, но стоить отметить относительно медленную скорость формирования эмбеддингов!
2) Предварительная обработка текста не дает прироста качества в итоговых алгоритмах классификации.
3) Модель Перцептрон показывает наилучшие результаты в классификации текстов.

Итоговыми алгоритмами классификации станут 2 модели:
1) Лучший алгоритм по качеству (best_model.h5): **модель 0** (all_mpnet_base_v2 наилучший алгоритм для построения эмбеддингов, но довольно медленный) 
2) Лучший алгоритм по скорости (fast_model.h5): **модель 7** (all_MiniLM_L6_v2 формирует эмбеддинги намного быстрее all_mpnet_base_v2 без существенной потери качества)

**Шаг 5: Обучение выбранных моделей**

После выбора оптимальных моделей для решения поставленной задачи необходимо обучить выбранные модели на всех доступных данных.

**Шаг 6: Тестирование моделей**

Для тестирования качества моделей на датафрейме test.csv предоставляю готовый скрипт (**4_testing_best_models.ipynb**) и файлы с уже обученными моделями. 

## **Возможные улучшения**
1) Тестирование алгоритмов классификации на других предобученных LLM для получения эмбеддингов текстов. Вот другие возможные алгоритмы: https://sbert.net/docs/pretrained_models.html
2) Преобразование каждого слова в отдельные эмбеддинги для дальнейшего обучения различных вариаций моделей LSTM и GRU
3) Использование в качестве алгоритма классификации моделей, основанных на алгоритме трансформер (LLM)
4) Более глубокий анализ смыслового содержания текстов, при решении задачи мы не знали смыслового содержания каждого класса. => возможное формирование дополнительных фич
5) Более качественная предобработка текста: удаление слов с ошибками, слов содержащие числа и т.д.
   

