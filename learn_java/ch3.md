# 3. Объекты, классы и пакеты в Java

## 3.1. Основы ООП

- Абстракция
- Инкапсуляция
- Наследование
- Полиморфизм

## 3.2. Пакеты и модификаторы доступа

### Пакеты

```java
package org.stepic.java;  // Принадлежность класса пакету

public class HelloWorld {
    // ...
}
```

Полное имя класса из примера выше: `org.stepic.java.HelloWorld`.

В отсутствие директивы принадлежность пакету класс принадлежит пакету по умолчанию.

Классы того же пакета могут ссылаться друг на друга по короткому имени. Классы разных пакетов должны ссылаться друг на
друга по полному имени или использовать импорт:

```java
import org.stepic.java.HelloWorld;

import java.util.*;
```

### import static

Используется для импорта статических полей и методов:

```java
import static java.lang.Math.sqrt;

import static java.lang.System.out;
```

### Пакеты стандартной библиотека

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

### Пакеты для стороннего кода

```java
org.stepik.java
com.google.common
org.apache.maven
com.intellij.idea
net.sf.json
io.netty
...
```

### Модификаторы доступа

```java
public class ModifiedDemo {

    public static void visibleEverywhere() {}

    protected static void inSubclasses() {}

    static void inPackage() {}

    private void inClass() {}
}
```

- `public` - разрешен доступ отовсюду без ограничений
- `protecterd` - доступ разрешен только для классов наследников и для классов текущего пакета
- отсутствие модификатора - доступ только в пределах пакета
- `private` - доступ только в пределах класса

`protected` и `pivate` не пременимы к классам верхнего уровня.

## 3.3. Объявления класса

```java
package java.lang;

public final class Integer {}
```

В файле может быть только один public класс. Ключевое слово final в объявлении класса означает что от данного класса
нельзя наследоваться.

### Поля

```java
package java.lang;

public final class Integer {
    private final int value;
}
```

Модификатор final у поля означает что полю можно присвоить значение только один раз, после чего изменения будут
запрещены.

### Конструкторы

```java
package java.lang;

public final class Integer {

    private final int value;

    public Integer(int value) {
        this.value = value;
    }
}
```

Без объявления конструктора в классе будет создан конструктор по умолчанию. Запрет на создание экземпляров класса:

```java
package java.lang;

public final class Math {
    private Math() {}
}
```

В классе может быть несколько конструкторов с разными параметрами:

```java
package java.math;

public class BigInteger {

    public BigInteger(String val) {
        this(val, 10);  // Вызов другого конструктора
    }

    public BigInteger(String val, int radix) {}
}
```

Для реализации деструктора стоит завести отдельный метод close и вызвать его самостоятельно:

```java
package java.io;

public class FileInputStream {

    protected void finalize() {
        // Auto cleanup
    }

    public void close() {}
}
```

### Методы

```java
package java.lang;

public final class Integer {

    private final int value;

    public int intValue() {
        return value;
    }
}
```

Модификатор final у метода означает что данный метод не может быть переопределен в классах-наследниках.

### Методы с одинаковыми именами

В классе может быть несколько методов с одинаковыми именами, но разным набором параметров:

```java
package java.lang;

public final class String {

    public int indexOf(int ch) {
        return indexOf(ch, 0);
    }

    public int indexOf(int ch, int fromIndex) {}
}
```

### Статические поля и методы

```java
package java.lang;

public final class Integer {

    public static final int MIN_VALUE = 0x80000000;  // Константа

    public static int rotateRight(int i, int distance) {
        return (i >>> distance) | (i << -distance);
    }
}
```

### Вложенные классы

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

Вложенный класс, объявленный с модификатором `static` теряет возможность обращаться к нестатическим членам внешнего класса.

### Перечисления

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

Пример с перечислением:

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

Методы:

```java
for (DayOfWeek day : DayOfWeek.values()) {
    System.out.println(day.ordinal() + " " + day.name());
}
```

- `name()` - возвращает строку - имя элемента как в исходном коде
- `ordinal()` - возвращает число - порядковый метод элемента перечисления, начиная с нуля
- `values()` - озвращает массив возможных элементов перечисления

### Аннотации

```java
package java.lang;

public final class Character {

    @Deprecated
    public static boolean isJavaLetter(char c) {}

    @SuppressWarnings("unchecked")
    public static final Class<Character> TYPE = (Class<Character>) Class.getPrivitiveClass("char");
}
```

## 3.4. Наследование. Класс Object

Пример с геометрическими фигурами: [gitgist](https://gist.github.com/anonymous/4d90331db876ce843e4cc48a27964d84).

Унаследоваться можно только от одного класса.

```java
package java.lang;

public final class BigDecimal extends Number {
    public int intValue() {}

    // No shortValue() method,
    // it's inherited from Number
}
```

### Переопределение

Возвращаемое значени подкласса должно быть того же класса что и оригинал или быть его подкласса.

```java
package java.lang;

public final class StringBuilder extends AbstractStringBuilder {
    @Override
    public StringBuilder append(String str) {}

    // Base method in AbstractStringBuilder:
    // AbstractStringBuilder append(String str)
}
```

### Конструкторы

Создание экземпляра класса наследника всегда включает вызов конструктора родителя. Если у конструктора нет параметров,
то компилятор сам подставит вызов родительского конструктора первой строчкой в конструктор наследника, в противном
случае нужно сделать все это руками:

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

Использование ключевого слова `super` разрешено только к теле класса наследника.

### Наследование по умолчанию

```java
package java.lang;

public final class String /* extends object */ {}
```

### Liskov Substitution Principle

Если S является подтипом T, то все объекты базового класса T в программе могут быть заменены на объекты класса S,
без изменения каких-либо желательных свойств программы. Другими словами поведение подтипов не должно противоречить
поведению базового класса.

## 3.5. Абстрактные классы и интерфейсы

Класс может соответствовать и абстрактному понятию. Для выражения этой концепции используется ключевое слово
`abstract`:

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

Для абстрактного класса нельзя создавать экземпляры. У абстрактного класса могут быть абстрактные методы (без
реализации):

```java
public abstract class Shape {
    ...

    public abstract double getArea();
```

### Интерфейс

Альтернатива абстрактному классу, в котором все методы публичные и абстрактные.

```java
public interface OrderService {
    Order[] getOrdersByClient(long clientId);
}
```

Для обратной совместимости в интерфейсы добавлена возможность реализации default-методов:

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

Реалиация интерфейса:

```java
public class OrderServiceImpl
        extends ServiceBase
        implements OrderService {

    public Order[] getOrdersByClient(long clientId) {
        ...
    }

}
```

В стандартной библиотеки присутствует большое число интерфейсов.

#### Functional Interface

Интерфейс с единственным абстрактным методом:

```java
package java.lang;

@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);
}
```

Пример использования:

```java
package org.stepic.java.timer;

public class Timer {

    public long measureTime(Runnable runnable) {
        long startTime = Sytem.currentTimeMillis();
        runnable.run();
        return System.currentTimeMillis() - startTime();

}
```

Главная программа:

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

Упрощенно с ссылкой на метод:

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

Упрощенно с lambda:

```java
public class Main {

    public static void main(String[] args) {
        Timer timer = new Timer;
        long time = timer.measureTime(() -> new BigDecimal("1234567").pow(10000));
        System.out.println(time);
```
