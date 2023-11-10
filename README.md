# Изучаем Q#. Не будь зашоренным ...

![КДПВ](https://github.com/dprotopopov/qantishor/blob/main/LqOttPVmV5I.jpg)

**Алгориитм Шора** — квантовый алгоритм факторизации (разложения числа на простые множители), позволяющий разложить число **M** за время **O((logM)^3)** используя **O(log M)** логических кубитов.

**Алгоритм Шора** был разработан **Питером Шором** в 1994 году. Семь лет спустя, в 2001 году, его работоспособность была продемонстрирована группой специалистов IBM. Число 15 было разложено на множители 3 и 5 при помощи квантового компьютера с 7 кубитами.

**Алгоритм Шора** состоит из 2-х частей - квантовых и классических вычислений.
- Квантовая часть алгоритма отвечает за определение периода функции с помощью квантовых вычислений.
- Классические вычисления решают задачу как по найденному периоду степенной функции найти разложение на сомножители.

![Алгоритм Шора. Квантовая часть](https://github.com/dprotopopov/qantishor/blob/main/Алгоритм_Шора.jpg):

Практически, схема этого алгоритма полностью повторяет схему **алгоритма Саймона**, с отличием в последнем шаге - вместо применения **оператора Адамара** перед измерением входного регистра, используется **оператор преобразования Фурье**.

А какие есть ещё варианты определить период функции используя квантовые вычисления?

Когда-то ранее я писал статьи про способы сравнения (поиска фрагментиа) изображений, поиска частоты сердечных сокращений с использованием операции вычисления скалярного произведения, которую я делал с помощью свёртки на основе БФП. \
- [Фурье-вычисления для сравнения изображений](https://habr.com/p/266129/)
- [Определение частоты сердечных сокращений методом корреляции с использованием быстрых Фурье преобразований](https://habr.com/p/597715/)

а так же начал повторять эту технологию при квантовых вычислениях
- [Изучаем Q#. Алгоритм Гровера. Не будите спящего Цезаря](https://habr.com/p/768666/)

Так почему бы не повторить успешный успех и, заодно, обобщить теорию вопроса?

## Начнём ...

Пусть у нас есть функция **f: {0,1}^n to {0,1}^m**. \
Очевидно, что мы можем её так же записать, и как **f: {0,1}^n to Z** и как **f: {0,1}^n to [0,1)** без потери общности рассуждений (то есть **{0,1}^m** можно рассматривать и как двоичное представление целого числа, и как рациональную дробь полученную делением целого числа на **2^m**)

### Вычисление скалярного произведения через инверсию и свёрку

Пусть у нас есть кольцо с операцией **op** и пара функций **a** и **b**, отображающих кольцо в поле, например комплексных чисел

Операцией свёрки **a** и **b** мы будем называть функцию **c = a x b**, такую что **c(i) = SUB a(ix)b(iy)|i=ix op iy**

Пусть у нас есть кольцо с операцией **op** и функция **f**, отображающих кольцо в поле, например комплексных чисел

Операцией инверсией аргумента назовём функцию **inv f** , такую что **(inv f)(i) = f( -i )**, где **i op -i == 0**

Скалярным произведением **a** и **b** называют **(a,b) = SUM a(i)b(i)** (=**(a,b)(0)** где **(a,b)(i) = SUM a(ix)b(iy)|i=ix-iy**)

Легко убедиться самостоятельно, что **(a,b) = a x (inv b)**

Пусть для регистров кубитов **a** и **b**

- **a = SUM A(i)|i>**
- **b = SUM B(i)|i>**

Тогда, если **op**:

- xor: **(a,b)(i) = a xor X(b) = SUM A(ix)B(iy)|i>|i=ix xor ~iy**
- add: **(a,b)(i) = a sub b = a add X(b) add |1> = SUM A(ix)B(iy)|i>|i=ix-iy**

Что даёт нам вычисление функции-скалярного произведения **a** и **b**?

Ответ - если **A(i)~=B(i op k)** для всех **i** и распределения отличны от тривиальных, то значение **(a,b)(k)** является максимальным (по модулю) среди других значений этой функции
А значит, если мы имеем пару регистров кубитов, содержащих частотные характеристики двух последовательностей, то 
1. вычисление свёртки этих двух регистров,
2. измерение результата, 
3. даст нам наиболее вероятное значение **|k>**

И, соответственно, если мы работаем с одной функцией, и **F(i)~=F(i (op k)^j)**, то
1. вычисление свёртки функции самой с собой
2. измерение результата
3. с большой вероятностью даст нам значение **0 (op k)^j** - то есть один из периодов функции

### И как это применить?

Предположим, что мы можем сформировать состояние квантового регистра по правилу **SUM SQRT(f(x))|x>** (что, собственно, предполагает и алгоритм Шора)\
Тогда, применив вышеизложенные рассуждения по свёртке функции **f** самой с собой, мы получим значения - кандидатуры на период функции **f** \
Таким образом, мы получили ещё один из вариантов для квантовой части **алгоритма Шора** - то есть алгоритм поиска периода функции, ~~со своим бэк-джеком и шл...~~ и без использования **Фурье-трансформаций**.

## А как выглядит трансформация для вычисления свёртки?

Да весьма просто - это либо реализация операции `арифметического вычитания` в кольце **2^n**, но реализованная на регистрах из кубитов, либо операция `исключающего или` над векторами длины **n**, реализованная на регистрах кубитов.

![Схема алгоритма поиска периода с использованием свёртки](https://github.com/dprotopopov/qantishor/blob/main/Алгоритм_свёртки.jpg)

Вот и всё ...

### И бонусом - как реализовать **SUM SQRT(f(x))|x>**, используя базисные гейты?

Если погуглить, то можно найти разные способы решения этой задачи, например [Decomposing unitary matrix into Q# quantum gates](https://codeforces.net/blog/entry/84655)

Но мы не ищем лёгких путей!!!

Например, ранее я писал статью, как задать требуемое состояние в кубите, а эту технику легко адаптировать к нашей задаче - задать состояние регистра кубитов равным значениям исследуемой функции.
- [Изучаем Q#. Орёл или решка?](https://habr.com/p/772722/)

И таки да - мы не занимаемся экономией кубитов - в данных рассуждениях потребовалось ещё **(m+1)*2^n** дополнительных кубитов, только для того, чтобы задать коэффициенты с нужным значением вероятности у регистра из **n** кубитов (но и сам алгорим Шора не отвечает на вопрос как задать требуемое состояние регистра кубитов).

## Ссылки
- https://github.com/dprotopopov/qrnd
- https://learn.microsoft.com/ru-ru/azure/quantum/tutorial-qdk-grovers-search?tabs=tabid-visualstudio
- https://learn.microsoft.com/ru-ru/azure/quantum/user-guide/host-programs?tabs=tabid-copilot
- https://learn.microsoft.com/ru-ru/training/modules/qsharp-create-first-quantum-development-kit/2-install-quantum-development-kit-code

## Ранее

- [Изучаем Q#. Орёл или решка?](https://habr.com/p/772722/)
- [Изучаем Q#. Обучаем перцептрон](https://habr.com/p/772172/)
- [Изучаем Q#. Статистическое сравнение двух последовательностей чисел](https://habr.com/p/769148/)
- [Изучаем Q#. Алгоритм Гровера. Не будите спящего Цезаря](https://habr.com/p/768666/)
- [Изучаем Q#. Делаем реализацию биноминального распределения](https://habr.com/p/766512/)
- [Первые шаги в Q#. Алгоритм Дойча](https://habr.com/p/759352/)







