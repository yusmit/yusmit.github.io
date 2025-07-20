# 2. Базовый синтаксис Java

## 2.1. Примитивные типы

- `boolean`
- `char`
- `byte`, `short`, `int`, `long`
- `float`, `double`

Переменные ссылочных типов представляют собой ячейку памяти, содержащую ссылку на участок памяти, представляющий
собой объект.

Ссылка может быть пустой, для этого переменной присваивается значение `null`

### boolean

В Java поддерживается 4 операции с `boolean`:

- "не" - `boolean haveSpareTime = !isBusy;`
- "и" - `boolean canGoToPark = haveSpareTime &&  weatherIsGood;`
- "или" - `boolean hadGoodTime = learnedJavaOnStepik || wentToPark;`
- "исключающее или" - `boolean tastesGood = addedKetchup ^ addedHoney;`

Вычисление по полной схеме для "и" и "или" доступно через операторы `&` и `|`.

Сокращенный вид записи (использует полную схему вычисления):

{% raw %}
```java
value &= expression;
value |= expression;
value ^= expression;
```
{%endraw%}

### Целочисленные типы

- `byte` - 8 бит, диапазон -128..+127
- `short` - 16 бит, диапазон -2^15..+2^15-1
- `int` - 32 бит, диапазон -2^31..+2^31-1
- `long` - 64 бит, диапазон -2^63..+2^63-1

Разные способы записи:

{% raw %}
```java
int decimal = 99;
int octal = 0755;
int hex = 0xFF;
int binary = 0b101;
int tenMillion = 10_000_000;
int tenBillion = 10_000_000_000L;
```
{%endraw%}

### char

- `char` - 16 бит, беззнаковый, 0..2^16 - 1
- Представляет номер символа в кодировке юникода

{% raw %}
```java
char literal = 'a'
char tab = '\t'
```
{%endraw%}

### Вещественные типы

- `float` - 32 бит, 1 знак, 23 мантисса, 8 экспонента
- `double` - 64 бит, 1 знак, 52 мантиса, 11 экспонента

Явное указание:

{% raw %}
```java
float floatWithSuffix = 36.6f;
double doubleWithSuffix = 36.6d;
```
{%endraw%}

Способы записи:

{% raw %}
```java
double simple = -1.234;
double exponential = -123.4e-2;
double hex = 0x1.Fp10;
float floatWithSuffix = 36.6f;
double withSuffix = 4d;
```
{%endraw%}

Особые случаи:

{% raw %}
```java
double positiveInfinity = 1.0 / 0.0;
double negativeInfinity = -1.0 / 0.0;
double nan = 0.0 / 0.0;
boolean notEqualInself = nan != nan;
```
{%endraw%}

### Класс Math

{% raw %}
```java
double s = Math.sin(Math.PI);
double q = Math.sqrt(16);
double r = Math.ceil(1.01);
int a = Math.abs(-13);
int m = Math.max(10, 20);
```
{%endraw%}

### Длинная арифметика

{% raw %}
```java
import java.math.*;

BigInteger two = BigInteger.valueOf(2);
BigInteger powerOfTwo = two.pow(100);

BigDecimal one = BigDecimal.valueOf(1);
BigDecimal divisionResult = one.divide(new BigDecimal(powerOfTwo));
```
{%endraw%}

## 2.2. Преобразование типов

### Неявное преобразование

Многие преобразования типов можно выполнять неявно присваивая переменной с одним типом значение другого типа.

{% raw %}
```java
byte byteValue = 123;
short shortValue = byteValue;
int intValue = byteValue;
long longValue = byteValue;

char charValue = '@';
int intFromChar = charValue;
long longFromChar = charValue;

float floatFromLong = longValue;
double doubleFromFloat = floatFromLong;
double doubleFromInt = intValue;
```
{%endraw%}

### Явное приведение

При потери точности:

{% raw %}
```java
int intValue = 1024;
byte byteValue = (byte) intValue;  // 0 - отбрасываются лишние старшие биты

double pi = 3.14;
int intFromDouble = (int) pi;  // 3 - отбрасывается дробная часть

float largeFloat = 1e20f;
int intFromLargeFloat = (int) largeFloat;  // Максимальное представимое целое число

double largeDouble = 1e100;
float floatFromLargeDouble = (float) largeDouble;  // Бесконечность
```
{%endraw%}

### Автоматическое расширение

При использовании бинарных арифметических или побитовых операторов:

1. Если один из операндов `double`, то другой тоже будет к нему приведен
2. Если один из операндов `float`, то оба приводятся к `float`
3. Если один из операндов `long`, то оба приводятся к `long`
4. Иначе оба приводятся к типу `int`

### Неявное приведение

{% raw %}
```java
byte a = 1;
a += 3
// a = (byte) (a + 3);

byte b = -1;
b >>>= 7;
// b = (byte) (b >>> 7);
```
{%endraw%}

### Классы-обертки

- boolean - Boolean
- byte - Byte
- short - Short
- int - Integer
- long - Long
- char - Character
- float - Float
- double - Double

{% raw %}
```java
int privitive = 0;
Integer reference = Integer.valueOf(primitive);  // boxing
int backToPrimitive = reference.intValue();      // unboxing
```
{%endraw%}

Нужны для:

- Хранения примитивных типов в коллекциях
- Нужно хранить факт отсутствия значения

### Конвертация в строку и обратно

{% raw %}
```java
long fromString = Long.parseLong("12345");
String fromLong = Long.toString(12345);
String concatenation = "area" + 51;
```
{%endraw%}

### Полезные методы

{% raw %}
```java
short maxShortValue = Short.MAX_VALUE;

int bitCount = Integer.bitCount(123);

boolean isLetter = Character.isLetter('a');

float floatInfinity = Float.POSITIVE_INFINITY;

double doubleNaN = Double.NaN;

boolean isNaN = Double.isNaN(doubleNaN);
```
{%endraw%}

## 2.3. Массивы и строки

### Массивы

{% raw %}
```java
int[] numbers = new int[100];

String[] args = new String[1];

boolean bits[] = new boolean[0];
```
{%endraw%}

### Массив ненулевых значений

{% raw %}
```java
int[] numbers = new int[] {1, 2, 3, 4, 5};

boolean[] bits = new boolean[] {true, false};

// Можно опустить new и тип массива в variable declaration
char[] digits = {'0', '1', '2', '3', '4', '5'};
```
{%endraw%}

### Работа с массивом

{% raw %}
```java
int[] numbers = {1, 2, 3, 4, 5};

int arrayLength = numbers.length;

int firstNumber = numbers[0];

int lastNumber = numbers[arrayLength - 1];

int indexOutOfBounds = numbers[5];
```
{%endraw%}

### Многомерные массивы

{% raw %}
```java
int [][] matrix1 = new int[2][2];
int [][] matrix2 = { {1, 2}, {3, 4} };

int[] fristRow = matrix2[0];      // Get firt row
int someElement = matrix2[1][1];  // Get one element
```
{%endraw%}

### Ступенчатые массивы

{% raw %}
```java
int[][] triangle = {
    {1, 2, 3, 4, 5},
    {6, 7, 8, 9},
    {10, 11, 12},
    {13, 14},
    {15}
};

int theSecondRowLength = triangle[1].length;
```
{%endraw%}

### Объявление метода, принимающего переменное число параметров

{% raw %}
```java
static int maxArray(int[] numbers) { ... };
maxArray(new int[] {1, 2, 3});

static int maxVarargs(int... numbers) { ... };
maxVarargs(1, 2, 3);  // Компилятор сам упакует аргументы в массив
```
{%endraw%}

### Сравнение двух массивов

{% raw %}
```java
import Java.util.Arrays;

int[] a = {1, 2, 3};
int[] b = {4, 5, 6};

boolean equals1 = a == b;                  // Сравнение ссылок - ссылаются ли на один и тот же объект
boolean equals2 = a.equals(b);             // Для массивов также сравнивает ссылки
boolean equals3 = Array.equals(a, b);      // Хорошо работает для одномерных массивов
boolean equals4 = Array.deepEquals(a, b);  // Решение для многомернх массивов
```
{%endraw%}

### Распечатка массива

{% raw %}
```java
int[] a = {1, 2, 3};

// Плохой метод
System.out.println(a);

// Работает с одномерными массивами
System.out.println(Arrays.toString(a));

// Работает с многомерными массивами
System.out.println(Arrays.deepToString(a));
```
{%endraw%}

### Строки

{% raw %}
```java
String hello = "Hello";
String specialChars = "\r\n\t\"\\";
String empty = "";

char[] charArray = {'a', 'b', 'c'};
String string = new String(charArray);
char[] charsFromString = string.toCharArray();

// Строка не должны заканчиваться нулевым символом
String zeros = "\u0000\u0000";
```
{%endraw%}

### Неизменяемость строк

{% raw %}
```java
String s = "stringIsImmutable";

int length = s.length();

char firstChar = s.charAt(0);

boolean endsWithTable = s.endsWith("table");

boolean containsIs = s.contains("Is");

String substring = s.substring(0, 6);

String afterReplace = s.replace("Imm", "M");

String allCapitals = s.toUpperCase();
```
{%endraw%}

Соединение двух строк также создает новую:

{% raw %}
```java
String hello = "Hello ";
String world = "world!";
String helloWorld = hello + world;
```
{%endraw%}

Что эквивалентно:

{% raw %}
```java
StringBuilder sb = new StringBuilder();
sb.append(hello);
sb.append(world);
String helloWorld = sb.toString();
```
{%endraw%}

### Сравнение строк

Нужно делать через метод, чтобы не проверять ссылки на один объект:

{% raw %}
```java
boolean contentEquals = s1.equals(s2);

boolean contentEqualsIgnoreCase = s1.equalsIgnoreCase(s2);
```
{%endraw%}

## 2.4. Управляющие конструкции: условные операторы и циклы

### Условный оператор

{% raw %}
```java
if (weatherIsGood) {
    walkInThePark();
} else {
    learnJavaOnStepic();
}
```
{%endraw%}

### Тернарный условный оператор

{% raw %}
```java
System.out.println("Weather is " + (weatherIsGood ? "good" : "bad"));
```
{%endraw%}

### Оператор switch

{% raw %}
```java
switch (digit) {
    case 0:
        text = "zero";
        break;

    case 1:
        text = "one";
        break;

    // ...

    default:
        text = "???";
}
```
{%endraw%}

### Цикл while

{% raw %}
```java
while (haveTime() && haveMoney()) {
    goShopping();
}
```
{%endraw%}

### Цикл do while

{% raw %}
```java
do {
    goShopping();
} while (haveTime() && haveMoney());
```
{%endraw%}

### Цикл for

{% raw %}
```java
for (int i = 0; i < args.length; i++) {
    System.out.println(args[i]);
}
```
{%endraw%}

### Цикл foreach

{% raw %}
```java
for (String arg : args) {
    System.out.println(arg);
}
```
{%endraw%}

### Оператор break

{% raw %}
```java
boolean found = false;

for (String element : haystack) {
    if (needle.equals(element)) {
        found = true;
        break;
    }
}
```
{%endraw%}

### Оператор continue

{% raw %}
```java
int count = 0;

for (String element : haystack) {
    if (!needle.equals(element)) {
        continue;
    }
    count++;
}
```
{%endraw%}

### Метки

{% raw %}
```java
boolean found = false;

outer:
for (int[] row : matrix) {
    for (int x : row) {
        if (x > 100) {
            found = true;
            break outer;
        }
    }
}
```
{%endraw%}
