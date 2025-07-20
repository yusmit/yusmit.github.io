# 3. Объекты, классы и пакеты в Java

## 3.1. Основы ООП

- Абстракция
- Инкапсуляция
- Наследование
- Полиморфизм

## 3.2. Пакеты и модификаторы доступа

### Пакеты

{% raw %}
```java
package org.stepic.java;  // Принадлежность класса пакету

public class HelloWorld {
    // ...
}
```
{%endraw%}

Полное имя класса из примера выше: `org.stepic.java.HelloWorld`.

В отсутствие директивы принадлежность пакету класс принадлежит пакету по умолчанию.

Классы того же пакета могут ссылаться друг на друга по короткому имени. Классы разных пакетов должны ссылаться друг на
друга по полному имени или использовать импорт:

{% raw %}
```java
import org.stepic.java.HelloWorld;

import java.util.*;
```
{%endraw%}

### import static

Используется для импорта статических полей и методов:

{% raw %}
```java
import static java.lang.Math.sqrt;

import static java.lang.System.out;
```
{%endraw%}

### Пакеты стандартной библиотека

{% raw %}
```java
java.lang
java.io
java.nio
java.math
java.time
java.util
java.util.regex
javax.xml
...
```
{%endraw%}

### Пакеты для стороннего кода

{% raw %}
```java
org.stepik.java
com.google.common
org.apache.maven
com.intellij.idea
net.sf.json
io.netty
...
```
{%endraw%}

### Модификаторы доступа

{% raw %}
```java
public class ModifiedDemo {

    public static void visibleEverywhere() {}

    protected static void inSubclasses() {}

    static void inPackage() {}

    private void inClass() {}
}
```
{%endraw%}

- `public` - разрешен доступ отовсюду без ограничений
- `protecterd` - доступ разрешен только для классов наследников и для классов текущего пакета
- отсутствие модификатора - доступ только в пределах пакета
- `private` - доступ только в пределах класса

`protected` и `pivate` не пременимы к классам верхнего уровня.

## 3.3. Объявления класса

{% raw %}
```java
package java.lang;

public final class Integer {}
```
{%endraw%}

В файле может быть только один public класс. Ключевое слово final в объявлении класса означает что от данного класса
нельзя наследоваться.

### Поля

{% raw %}
```java
package java.lang;

public final class Integer {
    private final int value;
}
```
{%endraw%}

Модификатор final у поля означает что полю можно присвоить значение только один раз, после чего изменения будут
запрещены.

### Конструкторы

{% raw %}
```java
package java.lang;

public final class Integer {

    private final int value;

    public Integer(int value) {
        this.value = value;
    }
}
```
{%endraw%}

Без объявления конструктора в классе будет создан конструктор по умолчанию. Запрет на создание экземпляров класса:

{% raw %}
```java
package java.lang;

public final class Math {
    private Math() {}
}
```
{%endraw%}

В классе может быть несколько конструкторов с разными параметрами:

{% raw %}
```java
package java.math;

public class BigInteger {

    public BigInteger(String val) {
        this(val, 10);  // Вызов другого конструктора
    }

    public BigInteger(String val, int radix) {}
}
```
{%endraw%}

Для реализации деструктора стоит завести отдельный метод close и вызвать его самостоятельно:

{% raw %}
```java
package java.io;

public class FileInputStream {

    protected void finalize() {
        // Auto cleanup
    }

    public void close() {}
}
```
{%endraw%}

### Методы

{% raw %}
```java
package java.lang;

public final class Integer {

    private final int value;

    public int intValue() {
        return value;
    }
}
```
{%endraw%}

Модификатор final у метода означает что данный метод не может быть переопределен в классах-наследниках.

### Методы с одинаковыми именами

В классе может быть несколько методов с одинаковыми именами, но разным набором параметров:

{% raw %}
```java
package java.lang;

public final class String {

    public int indexOf(int ch) {
        return indexOf(ch, 0);
    }

    public int indexOf(int ch, int fromIndex) {}
}
```
{%endraw%}

### Статические поля и методы

{% raw %}
```java
package java.lang;

public final class Integer {

    public static final int MIN_VALUE = 0x80000000;  // Константа

    public static int rotateRight(int i, int distance) {
        return (i >>> distance) | (i << -distance);
    }
}
```
{%endraw%}

### Вложенные классы

{% raw %}
```java
package java.util;

public class ArrayList<E> {

    Object[] elementData;

    public Iterator<E> iterator() {
        return new Itr();
    }

    private class Itr implements Iterator<E> {
        int cursor;
    }
}
```
{%endraw%}

Вложенный класс, объявленный с модификатором `static` теряет возможность обращаться к нестатическим членам внешнего класса.

### Перечисления

{% raw %}
```java
public class BadExample {
    public static final int MONDAY = 1;
    public static final int TUESDAY = 1;
    public static final int WEDNESDAY = 1;
    public static final int THURSDAY = 1;
    public static final int FRIDAY = 1;
    public static final int SATURDAY = 1;
    public static final int SUNDAY = 1;
}
```
{%endraw%}

Пример с перечислением:

{% raw %}
```java
package java.time;

public enum DayOfWeek {
    MONDAY,  // public static final
    TUESDAY,
    WEDNESDAY,
    THURSDAY,
    FRIDAY,
    SATURDAY,
    SUNDAY

    // Дальше можно объявлять поля и методы как в обычном классе
    // Можно создать конструктор и передавать аргументы в элементы выше
}
```
{%endraw%}

Методы:

{% raw %}
```java
for (DayOfWeek day : DayOfWeek.values()) {
    System.out.println(day.ordinal() + " " + day.name());
}
```
{%endraw%}

- `name()` - возвращает строку - имя элемента как в исходном коде
- `ordinal()` - возвращает число - порядковый метод элемента перечисления, начиная с нуля
- `values()` - озвращает массив возможных элементов перечисления

### Аннотации

{% raw %}
```java
package java.lang;

public final class Character {

    @Deprecated
    public static boolean isJavaLetter(char c) {}

    @SuppressWarnings("unchecked")
    public static final Class<Character> TYPE = (Class<Character>) Class.getPrivitiveClass("char");
}
```
{%endraw%}

## 3.4. Наследование. Класс Object

Пример с геометрическими фигурами: [gitgist](https://gist.github.com/anonymous/4d90331db876ce843e4cc48a27964d84).

Унаследоваться можно только от одного класса.

{% raw %}
```java
package java.lang;

public final class BigDecimal extends Number {
    public int intValue() {}

    // No shortValue() method,
    // it's inherited from Number
}
```
{%endraw%}

### Переопределение

Возвращаемое значени подкласса должно быть того же класса что и оригинал или быть его подкласса.

{% raw %}
```java
package java.lang;

public final class StringBuilder extends AbstractStringBuilder {
    @Override
    public StringBuilder append(String str) {}

    // Base method in AbstractStringBuilder:
    // AbstractStringBuilder append(String str)
}
```
{%endraw%}

### Конструкторы

Создание экземпляра класса наследника всегда включает вызов конструктора родителя. Если у конструктора нет параметров,
то компилятор сам подставит вызов родительского конструктора первой строчкой в конструктор наследника, в противном
случае нужно сделать все это руками:

{% raw %}
```java
package java.lang;

public final class StringBuilder extends AbstractStringBuilder {
    public StringBuilder() {
        super(16);
    }

    @Override
    public StringBuilder append(String str) {
        super.append(str);  // Вызов оригинального метода
        return this;
    }
}
```
{%endraw%}

Использование ключевого слова `super` разрешено только к теле класса наследника.

### Наследование по умолчанию

{% raw %}
```java
package java.lang;

public final class String /* extends object */ {}
```
{%endraw%}

### Liskov Substitution Principle

Если S является подтипом T, то все объекты базового класса T в программе могут быть заменены на объекты класса S,
без изменения каких-либо желательных свойств программы. Другими словами поведение подтипов не должно противоречить
поведению базового класса.

## 3.5. Абстрактные классы и интерфейсы

Класс может соответствовать и абстрактному понятию. Для выражения этой концепции используется ключевое слово
`abstract`:

{% raw %}
```java
public abstract class Shape {

    private final Color color;

    public Shape(Color color) {
        this.color = color;
    }

    public Color getColor() {
        return color;
    }

    public double getArea() {
        return Double.NaN;
    }
}
```
{%endraw%}

Для абстрактного класса нельзя создавать экземпляры. У абстрактного класса могут быть абстрактные методы (без
реализации):

{% raw %}
```java
public abstract class Shape {
    ...

    public abstract double getArea();
```
{%endraw%}

### Интерфейс

Альтернатива абстрактному классу, в котором все методы публичные и абстрактные.

{% raw %}
```java
public interface OrderService {
    Order[] getOrdersByClient(long clientId);
}
```
{%endraw%}

Для обратной совместимости в интерфейсы добавлена возможность реализации default-методов:

{% raw %}
```java
package org.stepic.java.orders;

import java.time.LocalDate;

public interface OrderService {

    Order[] getOrdersByClient(long clientId);

    // Добавление этого метода сломает компиляцию в унаследованных классах, где не будет его реализации
    // Order[] getOrdersByClient(long clientId, LocalDate date);

    // Будет использоваться для классов наследников, где не переопределен
    default Order[] getOrdersByClient(long clientId, LocalDate date) {
        Order[] allOrders = getOrdersByClient(clientId);
        return Orders.filterByDate(allOrders, date);
    }
}
```
{%endraw%}

Реалиация интерфейса:

{% raw %}
```java
public class OrderServiceImpl
        extends ServiceBase
        implements OrderService {

    public Order[] getOrdersByClient(long clientId) {
        ...
    }

}
```
{%endraw%}

В стандартной библиотеки присутствует большое число интерфейсов.

#### Functional Interface

Интерфейс с единственным абстрактным методом:

{% raw %}
```java
package java.lang;

@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);
}
```
{%endraw%}

Пример использования:

{% raw %}
```java
package org.stepic.java.timer;

public class Timer {

    public long measureTime(Runnable runnable) {
        long startTime = Sytem.currentTimeMillis();
        runnable.run();
        return System.currentTimeMillis() - startTime();

}
```
{%endraw%}

Главная программа:

{% raw %}
```java
package org.stepic.java.timer;

import java.math.BigDecimal;

public class Main {

    public static void main(String[] args) {
        Timer timer = new Timer();
        long time = timer.measureTime(new BigDecimalPower());
        System.out.println(time);
    }

    private static class BigDecimalPower implements Runnable {

        @Override
        public void run() { new BigDecimal("1234567").pow(10000); }

    }
}
```
{%endraw%}

Упрощенно с ссылкой на метод:

{% raw %}
```java
public class Main {

    public static void main(String[] args) {
        Timer timer = new Timer();
        long time = timer.measureTime(Main::bigDecimalPower);
        System.out.println(time);
    }

    private static void bigDecimalPower {
        new BigDecimal("1234567").pow(10000);
    }
}
```
{%endraw%}

Упрощенно с lambda:

{% raw %}
```java
public class Main {

    public static void main(String[] args) {
        Timer timer = new Timer;
        long time = timer.measureTime(() -> new BigDecimal("1234567").pow(10000));
        System.out.println(time);
```
{%endraw%}
