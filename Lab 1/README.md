## Рекомендации по архитектуре программы

Эта справка описывает один из возможных вариантов организации программы и некоторые средства стандартной библиотеки C++, которые удобно использовать при выполнении лабораторной работы.

Приведенная архитектура не является обязательной. Основная цель — разделить исходное табличное представление данных, информацию о признаках и готовую числовую матрицу.

---

## 1. Небольшое введение в шаблоны

Шаблоны позволяют писать функции и классы, работающие с разными типами данных.

Например:

```cpp
template <typename T>
T square(T x)
{
    return x * x;
}
```

Одна функция может использоваться с разными типами:

```cpp
square(5);      // T = int
square(2.5);    // T = double
```

Шаблоны являются одним из механизмов **статического полиморфизма** в C++.

В отличие от динамического полиморфизма с `virtual`, конкретный вариант шаблона определяется во время компиляции программы.

Упрощенно:

```text
template<T>
    |
    +-- T = int
    |
    +-- T = double
```

В этой лабораторной работе писать сложные шаблонные конструкции не требуется. Однако многие контейнеры и алгоритмы стандартной библиотеки, например `std::vector`, `std::optional` и `std::variant`, сами являются шаблонными классами.

---

## 2. Представление исходного датасета

До выполнения preprocessing датасет еще не является обычной числовой матрицей.

В нем могут одновременно присутствовать:

- числовые признаки;
- категориальные признаки;
- пропущенные значения;
- строковые значения категорий.

Поэтому удобно хранить данные по столбцам.

Например:

```cpp
struct NumericColumn
{
    std::vector<std::optional<double>> values;
};

struct CategoricalColumn
{
    std::vector<std::optional<std::string>> values;
};

using Column =
    std::variant<NumericColumn, CategoricalColumn>;
```

Сам датасет можно представить как:

```cpp
class Dataset
{
private:
    std::unordered_map<std::string, Column> columns_;

public:
    // ...
};
```

Ключом `std::unordered_map` является название признака.

Например:

```text
"age"       -> NumericColumn
"income"    -> NumericColumn
"education" -> CategoricalColumn
"job"       -> CategoricalColumn
```

Отдельное поле с числом строк необязательно: количество объектов можно определить по размеру любого столбца. При загрузке следует проверить, что все столбцы имеют одинаковую длину.

---

## 3. Пропущенные значения: `std::optional`

Пропущенное значение удобно представлять явно:

```cpp
std::optional<double>
std::optional<std::string>
```

Например:

```cpp
std::vector<std::optional<double>> values = {
    10.0,
    15.0,
    std::nullopt,
    22.0
};
```

Такой вариант предпочтительнее специальных значений вроде:

```cpp
-9999
```

или специальных строк.

При чтении конкретного датасета программа должна знать, каким образом пропуск обозначен в исходном файле:

```text
?
undefined
NA
...
```

После загрузки такое значение удобно преобразовать в:

```cpp
std::nullopt
```

---

## 4. Различные типы столбцов: `std::variant`

Для хранения числового или категориального столбца можно использовать:

```cpp
using Column =
    std::variant<NumericColumn, CategoricalColumn>;
```

Чтобы определить, какой именно тип сейчас хранится внутри `std::variant`, удобно использовать:

```cpp
std::holds_alternative<T>
```

Например:

```cpp
if (std::holds_alternative<NumericColumn>(column))
{
    auto& numeric =
        std::get<NumericColumn>(column);

    // обработка числового столбца
}
else if (std::holds_alternative<CategoricalColumn>(column))
{
    auto& categorical =
        std::get<CategoricalColumn>(column);

    // обработка категориального столбца
}
```

`std::holds_alternative<T>(column)` возвращает `true`, если внутри `column` сейчас хранится объект типа `T`.

После проверки значение можно получить с помощью:

```cpp
std::get<T>
```

Важно не вызывать `std::get<T>` для неправильного типа без предварительной проверки.

Более компактный вариант — `std::get_if`:

```cpp
if (auto* numeric =
        std::get_if<NumericColumn>(&column))
{
    // numeric указывает на NumericColumn
}
else if (auto* categorical =
             std::get_if<CategoricalColumn>(&column))
{
    // categorical указывает на CategoricalColumn
}
```

`std::get_if<T>` возвращает указатель на объект типа `T`, если такой тип хранится внутри `std::variant`, и `nullptr` в противном случае.

Для первого знакомства рекомендуется использовать связку:

```cpp
std::holds_alternative<T>
std::get<T>
```

а `std::get_if<T>` рассматривать как более компактную альтернативу.

---

## 5. Информация о признаках: `FeatureInfo`

Статистика должна вычисляться **до заполнения пропусков и преобразования признаков**.

Полученную информацию желательно хранить отдельно от самих данных.

Для числового признака можно использовать:

```cpp
struct NumericFeatureInfo
{
    std::size_t missing_count;

    double min;
    double max;

    double mean;
    double variance;

    double median;

    double q05;
    double q25;
    double q75;
    double q95;
};
```

Для категориального признака:

```cpp
struct CategoricalFeatureInfo
{
    std::size_t missing_count;

    std::vector<std::string> categories;

    std::unordered_map<
        std::string,
        std::size_t
    > frequencies;
};
```

Общий тип:

```cpp
using FeatureInfo =
    std::variant<
        NumericFeatureInfo,
        CategoricalFeatureInfo
    >;
```

Информацию обо всех признаках можно хранить, например, так:

```cpp
using DatasetInfo =
    std::unordered_map<
        std::string,
        FeatureInfo
    >;
```

Метод датасета:

```cpp
DatasetInfo Dataset::analyze() const;
```

должен вычислять статистику исходных данных, не изменяя сам датасет.

Например:

```cpp
DatasetInfo info = dataset.analyze();
```

---

## 6. Статистика числовых признаков

Для числового признака полезно вычислить:

- количество пропусков;
- минимум;
- максимум;
- среднее;
- дисперсию;
- медиану;
- `q05`;
- `q25`;
- `q75`;
- `q95`;
- частотную гистограмму.

Среднее и дисперсию можно посчитать обычным циклом.

Для вычисления квантилей удобно сначала получить числовые значения без пропусков и отсортировать их.

Например:

```cpp
std::ranges::sort(values);
```

Минимум и максимум можно найти как вручную, так и с помощью:

```cpp
std::ranges::minmax_element(values);
```

---

## 7. Статистика категориальных признаков

Для категориального признака следует сохранить:

- количество пропусков;
- исходные категории;
- частоту появления каждой категории.

Для подсчета частот удобно использовать:

```cpp
std::unordered_map<
    std::string,
    std::size_t
>
```

Например:

```text
red   -> 42
blue  -> 31
green -> 17
```

Важно сохранить именно исходный набор категорий до заполнения пропущенных значений.

Если позже пропуск преобразуется в специальную категорию:

```text
__MISSING__
```

это не должно уничтожать информацию о том, какие категории присутствовали в исходных данных.

---

## 8. `std::span`

`std::span` представляет невладеющий непрерывный участок памяти.

Например, статистическую функцию можно объявить как:

```cpp
double variance(
    std::span<const double> values
);
```

В отличие от:

```cpp
double variance(
    const std::vector<double>& values
);
```

такая функция не привязана именно к `std::vector`.

`std::span`:

- не копирует элементы;
- не владеет памятью;
- хранит представление уже существующего непрерывного участка памяти.

Его удобно использовать для функций, которые работают с подготовленным массивом чисел.

Например:

```cpp
double variance(
    std::span<const double> values
);

double quantile(
    std::span<const double> values,
    double q
);
```

---

## 9. Загрузка чисел: `std::from_chars`

Для преобразования строки в число можно использовать:

```cpp
std::from_chars
```

Например:

```cpp
double value;

auto [ptr, ec] =
    std::from_chars(
        token.data(),
        token.data() + token.size(),
        value
    );
```

В отличие от некоторых более старых способов преобразования строк в числа, `std::from_chars` не использует исключения для сообщения об ошибке парсинга.

Его использование не является обязательным, но полезно познакомиться с этим интерфейсом.

---

## 10. Импутация

После вычисления `DatasetInfo` можно заполнить пропущенные значения.

Например:

```cpp
enum class NumericImputation
{
    Mean,
    Median,
    Zero
};
```

Метод:

```cpp
void Dataset::impute(
    const DatasetInfo& info,
    NumericImputation strategy
);
```

должен использовать уже рассчитанные статистики.

Например:

```cpp
DatasetInfo info = dataset.analyze();

dataset.impute(
    info,
    NumericImputation::Median
);
```

Не следует повторно вычислять медиану после изменения датасета.

Категориальные пропуски можно заменить дополнительной категорией:

```text
__MISSING__
```

---

## 11. Масштабирование числовых признаков

Можно использовать перечисление:

```cpp
enum class Scaling
{
    MinMax,
    Standard,
    Robust
};
```

### Min-max scaling

$$
x' =
\frac{x-x_{\min}}
{x_{\max}-x_{\min}}
$$

### Standard scaling

$$
x' =
\frac{x-\mu}{\sigma}
$$

### Robust scaling

$$
x' =
\frac{x-Q_{0.5}}
{Q_{0.75}-Q_{0.25}}
$$

Все необходимые значения уже содержатся в `DatasetInfo`.

---

## 12. One-hot encoding

Категориальный признак:

```text
color

red
blue
green
```

после one-hot encoding может превратиться в:

```text
color_red  color_blue  color_green

1          0           0
0          1           0
0          0           1
```

После такого преобразования число столбцов итоговой матрицы может быть больше числа исходных признаков.

---

## 13. Числовая матрица

После импутации, масштабирования и кодирования категорий данные можно представить обычной плотной числовой матрицей.

Например:

```cpp
enum class Layout
{
    RowMajor,
    ColumnMajor
};
```

и:

```cpp
class Matrix
{
private:
    std::vector<double> data_;

    std::size_t rows_;
    std::size_t cols_;

    Layout layout_;

public:
    double& operator()(
        std::size_t row,
        std::size_t column
    );

    const double& operator()(
        std::size_t row,
        std::size_t column
    ) const;

    // ...
};
```

Для `RowMajor` элементы хранятся примерно так:

```text
a00 a01 a02 a10 a11 a12 ...
```

Для `ColumnMajor`:

```text
a00 a10 a20 a01 a11 a21 ...
```

Логическая матрица при этом остается той же самой.

---

## 14. Результат преобразования

Результат преобразования удобно хранить в отдельной структуре:

```cpp
struct TransformedDataset
{
    Matrix matrix;

    std::vector<std::string> feature_names;
};
```

Здесь:

```cpp
feature_names[j]
```

содержит название признака, соответствующего `j`-му столбцу числовой матрицы.

Например:

```text
0 -> age
1 -> income
2 -> color_red
3 -> color_blue
4 -> color_green
```

Это будет полезно в следующих лабораторных работах, например при выводе информации о признаке, выбранном решающим деревом.

Метод преобразования может иметь интерфейс:

```cpp
TransformedDataset Dataset::transform(
    const DatasetInfo& info,
    Scaling scaling,
    Layout layout
) const;
```

---

## 15. Общая последовательность работы

Один из возможных вариантов:

```cpp
Dataset dataset =
    load_dataset(...);

DatasetInfo info =
    dataset.analyze();

print_statistics(info);

dataset.impute(
    info,
    NumericImputation::Median
);

TransformedDataset transformed =
    dataset.transform(
        info,
        Scaling::Standard,
        Layout::RowMajor
    );
```

То есть:

```text
CSV
 |
 v
Dataset
 |
 +---- analyze() ----> DatasetInfo
 |
 +---- impute()
 |
 v
Dataset без пропусков
 |
 +---- transform()
 |
 v
TransformedDataset
 |
 +---- Matrix
 |
 +---- feature_names
```

---

## 16. Сериализация матрицы в бинарном виде

После преобразования датасета полезно сохранить полученную числовую матрицу не только в CSV, но и в простом бинарном формате.

Это позволяет увидеть различие между текстовым и непосредственным машинным представлением чисел.

Например, матрицу:

```text
1.23456789,2.34567891,3.45678912
4.56789123,5.67891234,6.78912345
```

в CSV необходимо хранить как последовательность символов.

Число:

```text
1.23456789
```

занимает несколько байтов только для записи отдельных символов:

```text
'1' '.' '2' '3' '4' ...
```

В бинарном формате значение типа `double` обычно занимает фиксированные:

```text
8 bytes
```

Поэтому числовые матрицы большого размера в бинарном виде обычно оказываются компактнее и значительно быстрее загружаются.

### Простейший формат

Можно сохранить:

```text
rows
cols
layout
data
```

Например:

```cpp
void Matrix::save(
    const std::filesystem::path& path
) const;
```

и:

```cpp
static Matrix Matrix::load(
    const std::filesystem::path& path
);
```

Для бинарной записи файла используется:

```cpp
std::ofstream file(
    path,
    std::ios::binary
);
```

После записи размеров можно сохранить непрерывный массив `double`:

```cpp
file.write(
    reinterpret_cast<const char*>(
        data_.data()
    ),
    data_.size() * sizeof(double)
);
```

При загрузке используется аналогичный вызов:

```cpp
file.read(
    reinterpret_cast<char*>(
        data_.data()
    ),
    data_.size() * sizeof(double)
);
```

Следует сначала загрузить размеры матрицы, выделить необходимый объем памяти и только затем читать сами данные.

Такой формат предназначен для учебной работы и не является универсальным форматом обмена данными между различными программами и архитектурами.

В частности, при создании серьезного формата пришлось бы отдельно учитывать:

- размер используемых типов;
- порядок байтов;
- версию формата;
- проверку целостности файла.

Для данной лабораторной достаточно простого бинарного формата, созданного и прочитанного одной и той же программой.

### Сравнение с CSV

После сохранения рекомендуется сравнить размеры:

```text
dataset.csv
dataset.bin
```

и вывести их в консоль.

Например:

```text
CSV size:    184320 bytes
Binary size:  80664 bytes
```

Так можно непосредственно увидеть стоимость текстового представления числовых данных.

---

## 17. Полезные средства STL и C++20

В этой лабораторной особенно полезно познакомиться со следующими средствами:

| Средство | Где может использоваться |
|---|---|
| `std::vector` | значения столбцов, данные матрицы |
| `std::unordered_map` | признаки датасета, частоты категорий |
| `std::optional` | пропущенные значения |
| `std::variant` | числовой или категориальный столбец |
| `std::get_if` | получение конкретного типа из `std::variant` |
| `std::span` | передача непрерывного массива чисел без копирования |
| `std::ranges::sort` | сортировка для вычисления квантилей |
| `std::ranges::minmax_element` | поиск минимума и максимума |
| `std::from_chars` | преобразование текста в число |
| `enum class` | стратегии обработки и тип layout |
| `std::filesystem::path` | работа с путями к файлам |
| `std::filesystem::file_size` | сравнение размера CSV и бинарного файла |
| `std::ifstream` / `std::ofstream` | загрузка и сохранение данных |
| `std::ios::binary` | бинарный режим работы с файлами |

Не требуется использовать все перечисленные средства в каждом возможном месте. Следует выбирать стандартные средства там, где они делают код понятнее и избавляют от лишней ручной инфраструктуры.