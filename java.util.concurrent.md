## java.util.concurrent

Это набор классов, облегчающих разработку многопоточных программ.

* Пакет  java.util.concurrent.locks
    * Работа  с  блокировками
* Пакет  java.util.concurrent.atomic
    * Атомарные  переменные
* Пакет  java.util.concurrent
    * Примитивы  синхронизации
    * Многопоточные  коллекции
    * Управление  заданиями

---

### Блокировки и условия

Только один поток может владеть блокировкой.

#### Интерфейс **Lock**

Методы:
* **lock()** - получить блокировку
* **lockInterruptibly()** - ожидает, пока не будет получена блокировка, если поток не прерван
* **unlock()** - отдать блокировку
* **tryLock()** - пытается получить блокировку,
если блокировка получена, то возвращает `true`.
Если блокировка не получена, то возвращает `false`.
В отличие от метода `lock()` не ожидает получения блокировки, если она недоступна

Передача событий:
* **newCondition()** - создать условие.
Это похоже на `wait-notify` модель с рядом дополнительных функций.
Объект `Condition` всегда создается с помощью объекта `Lock`.


#### Интерфейс **Condition**

Методы:
* **await()** - поток ожидает, пока не будет выполнено некоторое условие
и пока другой поток не вызовет методы signal/signalAll.
Во многом аналогичен методу wait класса Object
* **signal()** - сигнализирует, что поток, у которого ранее был
вызван метод await(), может продолжить работу.
Применение аналогично использованию методу notify класса Object
* **signalAll()** - сигнализирует всем потокам,
у которых ранее был вызван метод await(), что они могут продолжить работу.
Аналогичен методу notifyAll() класса Object

Пример "Ограниченный Буффер", [источник](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/Condition.html)
```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.Condition;

class BoundedBuffer {
   final Lock lock = new ReentrantLock();
   final Condition notFull  = lock.newCondition();
   final Condition notEmpty = lock.newCondition();

   final Object[] items = new Object[100];
   int putptr, takeptr, count;

   public void put(Object x) throws InterruptedException {
     lock.lock();
     // После этого только один поток имеет доступ к критической секции,
     // а остальные потоки ожидают снятия блокировки.

     try {
       while (count == items.length)
         notFull.await();
       items[putptr] = x;
       if (++putptr == items.length)
            putptr = 0;
       ++count;

       notEmpty.signal();
     } finally {

       lock.unlock();
       // Снимается блокировка обязательно в блоке finally,
       // так как в случае возникновения ошибки все остальные потоки окажутся заблокированными.
     }
   }

   public Object take() throws InterruptedException {
     lock.lock();
     // После этого только один поток имеет доступ к критической секции,
     // а остальные потоки ожидают снятия блокировки.

     try {
       while (count == 0)
         notEmpty.await();
       Object x = items[takeptr];
       if (++takeptr == items.length)
            takeptr = 0;
       --count;

       notFull.signal();
       return x;
     } finally {

       lock.unlock();
       // Снимается блокировка обязательно в блоке finally,
       // так как в случае возникновения ошибки все остальные потоки окажутся заблокированными.
     }
   }
 }
```

---

**ReentrantLock** - реализация интерфейса **Lock**.

[Источник](https://habr.com/post/187854/)

В отличие от `syncronized` блокировок,
`ReentrantLock` позволяет более гибко выбирать моменты снятия и получения блокировки,
т.к. использует обычные Java вызовы.

Также `ReentrantLock` позволяет получить информацию о текущем состоянии блокировки,
разрешает «ожидать» блокировку в течение определенного времени.
Поддерживает правильное рекурсивное получение и освобождение блокировки для одного
потока.

Если вам необходимы честные блокировки
(соблюдающие очередность при захвате монитора)
 — `ReentrantLock` также снабжен этим механизмом.

>Несмотря на то, что `syncronized` и `ReentrantLock` блокировки
очень похожи — реализация на уровне JVM отличается довольно сильно.
Не вдаваясь в подробности JMM:
использовать ReentrantLock вместо предоставляемой
 JVM `syncronized` блокировки стоит только в том случае,
 если у вас очень часто происходит битва потоков за монитор.
 В случае, когда в `syncronized` метод _обычно_ попадает
  лишь один поток — производительность ReentrantLock уступает механизму блокировок JVM.

Дополнительные методы:
* **isFair()** - «честность»  блокировки
* **isLocked()** - блокировка  занята
* множество методов, предоставляющих статистику во время выполнения:
    * **getQueuedThreads()**, **getQueueLength()**,
    **hasQueuedThread(thread)**, **hasQueuedThreads()** - потоки,  ждущие  блокировку
    * **getWaitingThreads(condition)**, **getWaitQueueLength(condition)** - потоки,  ждущие  условие

#### Интерфейс ReadWriteLock

**ReadWriteLock** - это интерфейс создания read/write блокировок,
который реализует один единственный класс `ReentrantReadWriteLock`.
Блокировку чтение-запись следует использовать при длительных
и частых операциях чтения и редких операциях записи.
Тогда при доступе к защищенному ресурсу используются разные
методы блокировки, как показано ниже

Методы:
* **readLock()** - получить блокировку на чтение
* **writeLock()** - получить блокировку на запись

Пример:
```java
ReadWriteLock rwl = new ReentrantReadWriteLock();
Lock  readLock    = rwl.readLock();
Lock  writeLock   = rwl.writeLock();
```

---

### Управление заданиями
[Источник](http://java-online.ru/concurrent.xhtml)

Многопоточный пакет `concurrent` включает средства,
называемые сервисами исполнения, позволяющие управлять потоковыми
задачами с возможностью получения результатов через интерфейсы
`Future` и `Callable`.

Какие интерфейсы будут использованы:
* **Runnable** - содержит только один метод `run()`, который выполняется
при запуске потока.

    Пример `Runnable`:
    ```java
    class MyThread implements Runnable
    {
        Thread thread;
        MyThread() {
            thread = new Thread(this, "Дополнительный поток");
            System.out.println("Создан дополнительный поток " + thread);
            thread.start();
        }
        @Override
        public void run() {
            try {
                for (int i = 5; i > 0; i--) {
                    System.out.println("\tдополнительный поток: " + i);
                    Thread.sleep(500);
                }
            } catch (InterruptedException e) {
                System.out.println("\tдополнительный поток прерван");
            }
            System.out.println("\tдополнительный поток завершён");
        }
    }
    public class RunnableExample
    {
        public static void main(String[] args)
        {
            new MyThread();
            try {
                for (int i = 5; i > 0; i--) {
                    System.out.println("Главный поток: " + i);
                    Thread.sleep(1000);
                }
            } catch (InterruptedException e) {
                System.out.println("Главный поток прерван");
            }
            System.out.println("Главный поток завершён");
        }
    }
    ```
* **Callable\<V>** - расширенный аналог интерфейса `Runnable`,
позволяющий возвращать типизированное значение.
В интерфейсе используется метод `call`, являющийся
аналогом метода `run` интерфейса `Runnable`.
* **Future\<V>** - интерфейс для получения результатов работы потока.
Объект данного типа возвращает сервис исполнения `ExecutorService`,
который в качестве параметра получает объект типа `Callable<V>`.
Метод `get` объекта `Future` блокирует текущий поток
(с таймаутом или без) до завершения работы потока `Callable`.
Кроме этого, интерфейс `Future` включает методы
для отмены операции и проверки текущего статуса.
В качестве имплементации часто используется класс `FutureTask`.

    * Методы, ([источник описания](http://java-online.ru/concurrent-callable.xhtml)):
        * **cancel(boolean mayInterruptIfRunning)** - попытка завершения задачи
        * **V get()** - ожидание (при необходимости) завершения задачи, после чего можно будет получить результат
        * **isDone()** - вернет `true`, если задача завершена
        * **isCancelled()** - вернет `true`, если выполнение задачи будет прервано прежде завершения
    * Пример для `Callable` и `Future`:
    ```java
    import java.util.Date;
    import java.util.List;
    import java.util.ArrayList;
    import java.util.concurrent.Future;
    import java.util.concurrent.Callable;
    import java.util.concurrent.Executors;
    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.ExecutionException;

    import java.text.SimpleDateFormat;

    public class CallableExample
    {
        public CallableExample()
        {
            // Определяем пул из трех потоков
            ExecutorService executor = Executors.newFixedThreadPool(3);

             // Список ассоциированных с Callable задач Future
            List<Future<String>>  futures;

            futures = new ArrayList<Future<String>>();

             // Создание экземпляра Callable класса
             Callable<String> callable = new CallableClass();

             for (int i = 0; i < 3; i++){
                 /*
                  * Стартуем возвращаюший результат исполнения
                  * в виде объекта Future поток
                  */
                 Future<String> future = executor.submit(callable);
                 /*
                  * Добавляем объект Future в список для отображения
                  * результат выполнения (получение наименования потока)
                  */
                 futures.add(future);
             }
             SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss  ");
             for (Future<String> future : futures){
                 try {
                     // Выводим в консоль полученное значение Future
                     String text = sdf.format(new Date()) + future.get();
                     System.out.println(text);
                 } catch (InterruptedException | ExecutionException e) {}
             }
             // Останавливаем пул потоков
             executor.shutdown();
        }
        //---------------------------------------------------------------
        // Класс, реализующий интерфейс Callable
        class CallableClass implements Callable<String>
        {
            @Override
            public String call() throws Exception {
                Thread.sleep(1000);
                // наименование потока, выполняющего callable задачу
                return Thread.currentThread().getName();
            }
        }
        //---------------------------------------------------------------
        public static void main(String args[])
        {
            new CallableExample();
        }
    }
    ```


* **FutureTask\<V>** - класс-оболочка, базирующаяся на конкретной
 реализации интерфейса `Future`.
 Чтобы создать реализацию класса `FutureTask` необходим объект `Callable`.
 `FutureTask` представляет удобный механизм для превращения `Callable`
 одновременно в `Future` и `Runnable`, реализуя оба интерфейса.
 Имплементация `FutureTask` может быть передана на выполнение классу,
 реализующему интерфейс `Executor`,
 либо запущена в отдельном потоке, как класс, реализующий интерфейс `Runnable`.

#### Интерфейс Executor и его реализация, ExecutorService

В интерфейсе `Executor` определен один метод:

```java
void execute(Runnable thread);
```

#### ExecutorService

[Ссылка](http://java-online.ru/concurrent-executor.xhtml)

В многопоточный пакет `concurrent` для управления потоками включено средство,
называемое сервисом исполнения `ExecutorService`.
Данное средство служит альтернативой классу `Thread`,
предназначенному для управления потоками.

**ExecutorServise** представляет собой интерфейс,
имплементация которого используется для запуска потоков.


Потоки можно запускать, используя методы `execute` и `submit`.
Оба метода в качестве параметра принимают объекты `Runnable`
или `Callable`.

Метод `submit` возвращает значение типа `Future`,
позволяющий получить результат выполнения потока.

Метод `invokeAll`
 работает со списками задач с блокировкой потока до
 завершения всех задач в переданном списке или до истечения заданного времени.

 Метод `invokeAny` блокирует вызывающий поток до завершения любой из переданных задач.
 Реализация данного интерфейса включает метод `shutdown`,
 позволяющий завершить все принятые на исполнение задачи и блокирует поступление новых.

Примеры:
 *  **Execute Runnable**:
    ```java
    ExecutorService executorService = Executors.newSingleThreadExecutor();

    executorService.execute(new Runnable() {
        public void run() {
            System.out.println("Asynchronous task");
        }
    });

    executorService.shutdown();
    ```

* **Submit Runnable**
    ```java
    Future future = executorService.submit(new Runnable() {
        public void run() {
            System.out.println("Asynchronous task");
        }
    });

    future.get();  //returns null if the task has finished correctly.
    ```

* **Submit Callable**
    ```java
    Future future = executorService.submit(new Callable(){
        public Object call() throws Exception {
            System.out.println("Asynchronous Callable");
            return "Callable Result";
        }
    });

    System.out.println("future.get() = " + future.get());
    ```

* **invokeAll()**
    ```java
    ExecutorService executorService = Executors.newSingleThreadExecutor();

    Set<Callable<String>> callables = new HashSet<Callable<String>>();

    callables.add(new Callable<String>() {
        public String call() throws Exception {
            return "Task 1";
        }
    });
    callables.add(new Callable<String>() {
        public String call() throws Exception {
            return "Task 2";
        }
    });
    callables.add(new Callable<String>() {
        public String call() throws Exception {
            return "Task 3";
        }
    });

    List<Future<String>> futures = executorService.invokeAll(callables);

    for(Future<String> future : futures){
        System.out.println("future.get = " + future.get());
    }

    executorService.shutdown();
    ```


**Итог по Executor:**
при запуске задач с помощью `Executor` пакета `java.util.concurrent`
 не требуется прибегать к низкоуровневой поточной функциональности класса `Thread`,
 достаточно создать объект типа `ExecutorService` с нужными свойствами
 и передать ему на исполнение задачу типа `Callable`.
  Впоследствии можно легко просмотреть результат выполнения
  этой задачи с помощью объекта `Future`.

