## NIO
NIO = New Input/Output

[источник](http://tutorials.jenkov.com/java-nio/overview.html)

### Overview

**Java NIO** состоит из следующих ключевых компонент:
* Channels
* Buffers
* Selectors

В **NIO** больше классов, но именно эти три являются наиболее важными.

#### Channels and Buffers

**Channel**

Обычно все IO операции в **NIO** начинаются с `Channels`.

Есть несколько реализаций `Channels` в **NIO**:

* **FileChannel** - читать и записывать данные в файлы
* **DatagramChannel** - читать и записывать данные через сеть по протоколу **UDP**
* **SocketChannel** - читать и записывать данные через сеть по протоколу **TCP**
* **ServerSocketChannel** - позволяет слушать все входящие **TCP** соединения, как это делают сервера. Для каждого входящего соединения создается `SocketChannel`

То есть `Channels` поддерживают протоколы **UDP**, **TCP**, а также **File IO**

`Channels` читают данные из буфферов и записывают данные в буффер:

![Ch-B](./pictures/channels-buffers.png)

`Channels` похожи на `Stream` за некоторыми исключениями:
* можно писать и читать через `Channel`. `Steam` обычно только однонаправленные: либо только для чтения, либо только для записи
* `Channel` могут читать и писать асинхронно
* `Channel` проводят все операции через `Buffer`

**Buffer**

Есть некоторое количество реализаций `Buffer` в **Java NIO**:

* ByteBuffer
* CharBuffer
* DoubleBuffer
* FloatBuffer
* IntBuffer
* LongBuffer
* ShortBuffer

То есть буфферы работают со всеми примитивными типами.

**Пример**
```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
FileChannel inChannel = aFile.getChannel();

ByteBuffer buf = ByteBuffer.allocate(48);

int bytesRead = inChannel.read(buf);
while (bytesRead != -1) {

  System.out.println("Read " + bytesRead);
  buf.flip();

  while(buf.hasRemaining()) {
      System.out.print((char) buf.get());
  }

  buf.clear();
  bytesRead = inChannel.read(buf);
}
aFile.close();
```

Функция `flip()` переключает буффер между режимами чтения-записи.
