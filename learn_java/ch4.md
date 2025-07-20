# 4. Обработка ошибок, исключения, отладка

## 4.1. Исключения

### Примеры работы с ошибками

{% raw %}
```java
public interface Calculator {
    double calculate(String expr);
}
```
{%endraw%}

- Самый простой способ обработать ошибочное значение в методе - завершить исполнение с `System.exit(1)`
- Лучший способ возвращать специальное значение, например `Double.NAN`.
- Можно также завести поле под признак ошибки внутри класса

### Примеры исключений

{% raw %}
```java
Object nullRef = null;

// java.lang.NullPointerException
nullRef.toString();
```
{%endraw%}

{% raw %}
```java
int[] array = {1, 2, 3};

// java.lang.ArrayIndexOutOfBoundsException
array[10];
```
{%endraw%}

{% raw %}
```java
// java.io.FileNotFoundException
new FileInputStream("not_existing_file");
```
{%endraw%}

### `java.lang.Throwable`

Все исключения в Java наследуются от `java.lang.Throwable`

{% raw %}
```java
throw new IllegalStateException("Invalid user.");
```
{%endraw%}

### Методы `Throwable`

- `public String getMessage()` получить строку из конструктора исключения
- `public void printStackTrace()` печатает stacktrace в stderr
- `public StackTraceElement[] getStackTrace()` возвращает stacktrace в виде массива
- `public Throwable getCause()` - возвращает другое исключение, явившееся причиной данного или `null`, если данное исключение является первичным. Причина может быть задана при инициализации исключения
- `public Trowable[] getSuppressed()` - возвращает исключения, которые были заглушены в процессе обработки основного исключения

### Классификация исключений

- Исключительные ситуации в JVM - `java.lang.Error`
- Исключительные ситуации в пользовательском коде
    - Проверяемые (checked) - `java.lang.Exception`
    - Непроверяемые (unchecked) - `java.lang.RuntimeException`, наследует `java.lang.Exception`
    
#### `java.lang.Error`

- `java.lang.OutOfMemoryError`
- `java.lang.NoClassFoundError`
- `java.lang.VeirfyError`

#### `java.lang.Exception`

{% raw %}
```java
import java.io.IOException;

public class ExceptionDemo {
    public void someMethod() throws IOException {
        throw new IOException("Failed to read file.");
    }
}
```
{%endraw%}

#### `java.lang.RuntimeException`

- `java.lang.NullPointerException`
- `java.lang.ArrayIndexOufOfBoundException`
- `java.lang.ArithmeticException`

### Собственное исключение

{% raw %}
```java
public class CalculatorException extends RuntimeException {
    public CalculatorException(String message) {
        super(message);
    }
    
    public CalculatorException(String message, Throwable cause) {
        super(message, cause);
    }
}
```
{%endraw%}

## 4.2. Обработка исключений. Try-catch

### Базовый пример

{% raw %}
```java
for (;;) {
    System.out.print("Enter expression: ");
    String expression = readUserInput();
    if (expr == null || "exit".equalsIgnoreCase(expr)) {
        break;
    }
    try {
        double result = calculator.calculat(expr);
        System.out.println("Result: " + result);
    } catch (CalculatorException e) {
        System.out.print("Bad expression: " + e.getMessage());
        System.out.print("Please try again: ");
    }
}
```
{%endraw%}

### Перехват нескольких исключений

{% raw %}
```java
try {
    // ...
} catch (FirstException e) {
    e.printStackTrace();
} catch (SecondException e) {
    e.printStackTrace();
}
```
{%endraw%}

или в одном блоке `catch`

{% raw %}
```java
try {
    // ...
} catch (FirstException | SecondException e) {
    e.printStackTrace();
}
```
{%endraw%}

### Блок `finally`

{% raw %}
```java
InputStream is = new FileInputStream("a.txt");
try {
    readFromInputStream(is);
} finally {
    is.close();
}
```
{%endraw%}

`finally` обрабатывает всегда вне зависимости от вызванных исключений. Обычно предназначен для снятия блокировок, закрытия ресурсов.

В примере выше, если исключение будет брошено внутри `try` и внутри `finally`, то оригинальное исключение будет потеряно. Для решения этой проблемы используется следующий паттерн

{% raw %}
```java
InputStream is = new FileInputStream("a.txt");
try {
    readFromInputStream(is);
} finally {
    try {
        is.close();
    } catch (IOException e) {
        // ignore
    }
}
```
{%endraw%}

### `try` с ресурсами

{% raw %}
```java
try (InputStream is = new FileInputStream("a.txt")) {
    readFromInputStream(is);
}
```
{%endraw%}

- Несколько ресурсов перечисляется через `;`
- При выходе из блока все ресурсы будут освобождены (вызван метод `close()`)
- Исключение из `close()` не перебьет первое, а будет добавлено как заглушенное

#### Идея за реализацией `try` с ресурсами

{% raw %}
```java
InputStream is = new FileInputStream("a.txt");
try {
    readFromInputStream(is);
} catch (Throwable t) {
    try {
        is.close();
    } catch (Trhowable t2) {
        t.addSuppressed(t2);
    }
    throw t;
}
is.close();
```
{%endraw%}

#### Ресурс

**Ресурс** - любой объект реализующий `java.lang.AutoCloseable`.

{% raw %}
```java
package java.lang;

public static AutoClosable {
    void close() throws Exception;
}
```
{%endraw%}

### Обработка исключения

{% raw %}
```java
try {
    Double.parseDouble(...);
} catch (NumberFormatException e) {
    throw new CalculatorException(e);
}
```
{%endraw%}

Для простого логирования ошибки: `e.printStackTrace()`.

### Плохой пример

{% raw %}
```java
String string;
try {
    string = object.toString();
 } catch (NullPointerException e) {
    string = "null";
 }
 System.out.println(string);
```
{%endraw%}

### Хороший пример

{% raw %}
```java
String string = object == null ? "null" : object.toString();
System.out.println(string);
```
{%endraw%}

Можно также воспользоваться `Objects.toString(object)` из `java.util`, который реализует туже логику.

## 4.3. Логирование

### Пример

{% raw %}
```java
package org.stepik.java.logging;

import java.util.logging.*;

public class LogDemo {
    
    private static final Logger LOGGER = Logger.getLogger(LogDemo.class.getName());
}
```
{%endraw%}

### Логирование сообщений

{% raw %}
```java
LOGGER.log(Level.INFO, "I'm logging");

// SEVERE, WARNING, INFO, CONFIG, FINE, FINER, FINEST

LOGGER.warning("We have a problem!);
```
{%endraw%}

### Настройка уровня логирования

{%raw%}
```
LOGGER.setLevel(Level.WARNING);
```
{%endraw%}

### Добавление в сообщение дополнительной информации

{%raw%}
```
LOGGER.log(Level.FINEST, "Current value of x is " + x);
LOGGER.log(Level.FINEST, "Current value of x is {0}", x);
LOGGER.log(Level.FINEST, "Point coordinates are ({0}, {1})", new Object[] {x, y});
LOGGER.log(Level.SEVERE, "Unexpected exception", e);
```
{%endraw%}

### Обработчики сообщений

Определяют куда будет записано сообщение.

- `java.util.logging.ConsoleHandler`
- `java.util.logging.FileHandler`
- `java.util.logging.SocketHandler`

### Форматеры сообщений

Определяют формат вывода.

- `java.util.logging.SimpleFormatter`
- `java.util.logging.XMLFormatter`

### Полный пример

{% raw %}
```java
Logger logger = Logger.getLogger(Stepik.class.getName());
Handler handler;
try {
    handler = new FileHandler("log.xml");
} catch (Exception e) {
    handler = new ConsoleHandler();
}
Formatter formatter = new XMLFormatter();

handler.setFormatter(formatter);
handler.setLevel(Level.WARNING);
logger.addHandler(handler);  
    
logger.log(Level.INFO, "test info");
logger.log(Level.WARNING, "test warning"); // Выводится в файл
```
{% endraw %}