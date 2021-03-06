## Sockets

Каждое приложение, использующее клиент-серверную парадигму, следует следующим принципам:
1. Сервер блокируется, ожидая, пока к нему не подключится клиент
2. Клиент подключается
3. Сервер и клиент обмениваются информацией столько, сколько необходимо
4. Сервер и клиент после этого оба прерывают соединение

`NB`: 
* сервер "слушает" определенный порт
* несколько клиентов могут быть соединены с одним сервером через один и тот же порт
* каждое соединение с клиентом происходит через свой сокет на одном порте
* то есть для успешного соединения клиента с сервером, клиенту необходимо получить
номер порта и сокет

**Sockets** - низкоуровневый API для передачи байт по сети,
поддерживающий TCP, UDP протоколы. Также поддерживается `IPv4`,
`IPv6`.

### UDP

пакет - `java.net.DatagramSocket`

#### Клиент

```java
try (DatagramSockets  =  new  DatagramSocket())  {
	DatagramPacket p  =  new  DatagramPacket(buf, buf.length, remoteAddress);
	s.send(p);
}
```

#### Сервер

```java
try (DatagramSockets  =  new  DatagramSocket(port)) {
	byte  []  buf =  new  byte  [1024];
	DatagramPacket p  =  new  DatagramPacket(buf, buf.length);
	s.receive(p);
}
```

### TCP

**Схема работы**
* Сервер  создает  «серверный  сокет»,  слушающий  конкретный  порт
* Сервер  вызывает  метод  `accept()`.  Данный  метод  ожидает  до  тех  пор  пока  клиент  не  подключится
* Клиент  создает  «клиентский  сокет»  и  задает  IP  адрес  и  порт  сервера
* Конструктор  сокета  в  процессе  работы  пытается  установить  соединение
* На  стороне  сервера  метод  `accept()`  возвращает  ссылку  на  Socket,  по  которому  сервер  может  общаться  с  подключившимся  клиентом

#### Клиент

пакет - `java.net.Socket`

* **public Socket(String host, int port) throws UnknownHostException, IOException** - этот метод создает соединение
с указанным сервером через определенный порт. Если метод бросил исключение, подключение завершилось неудачей.

* **public Socket(String host, intport,InetAddress localAddress, int localPort) throws IOException** - этот метод создает соединение
с указанным сервером через определенный порт, при этом на стороне клиента создается сокет с указанным адресом и портом.

* **public Socket()** - создается сокет, который пока ни с кем не соединено. Для подключения сокета клиента с сервером,
необходимо вызвать метод `connect()`

```java
// создаем сокет
Socket  socket =  new  Socket  ("localhost",  11111);

// создаем стрим, чтобы отправлять на сервер данные
OutputStream os =  socket.getOutputStream();
os.write(requestBytes);
os.flush();

// создаем сокет, чтобы получать с сервера данные
InputStream is  =  socket.getInputStream();
is.read(responseBytes);
```

#### Сервер

пакет - `java.net.ServerSocket`

* **public ServerSocket(int port) throws IOException** - этот метод создает сокет, который привязан к указанному порту. Если метод
бросил исключение, это значит, что другое приложение слушает указанный порт.

* **public ServerSocket(int port, int backlog) throws IOException** - этот метод аналогичен предыдущему. `backlog` - указывает
максимальное количество клиентов, которые могут храниться в очереди на подключение.

* **public ServerSocket(int port, int backlog, InetAddress address) throws IOException** - этот метод аналогичен предыдущему. 
`adress` - это локальный IP-адрес, к которому должен быть привязан созданный сокет. Данный конструктор используется для сервером,
имеющих несколько IP-адресов.

* **public ServerSocket() throws IOException** - создается сокет, который не привязан ни к чему. Для подключения необходимо использовать метод `bind()`.

```java
// создаем сервер-сокет 
ServerSocket server  =  new  ServerSocket(11111);

// подключаемся к клиент-сокету
Socket  socket =  server.accept();

// создаем стрим, чтобы получать данные
InputStream is  =  socket.getInputStream();
is.read(requestBytes);

// создаем стрим для отправки данных
OutputStream os=  socket.getOutputStream();
os.write(responseBytes);
os.flush(); 
```

другой пример, [источник](http://cs.lmu.edu/~ray/notes/javanetexamples/):
```java
/**
 * A TCP server that runs on port 9090.  When a client connects, it
 * sends the client the current date and time, then closes the
 * connection with that client.  Arguably just about the simplest
 * server you can write.
 */
public class DateServer {

    /**
     * Runs the server.
     */
    public static void main(String[] args) throws IOException {
        ServerSocket listener = new ServerSocket(9090);
        try {
            while (true) {
                Socket socket = listener.accept();
                try {
                    PrintWriter out =
                        new PrintWriter(socket.getOutputStream(), true);
                    out.println(new Date().toString());
                } finally {
                    socket.close();
                }
            }
        }
        finally {
            listener.close();
        }
    }
}
```

### Обработка большого количества клиентов

1. Обрабатываем запросы от всех клиентов в одном потоке:
	```
	while  (true)  {
		accept  a  connection;
		deal  with  the  client;
	}
	```
2. Работаем с каждым клиентом в отдельном, специально созданном потоке (стоит переиспользовать потоки)
	```
	while  (true)  {
		accept  a  connection;
		create  a  thread  to  deal  with  the  client;
	}
	```
3. Для работы с каждым клиентом создаем новое задание. Отправляем его в `ThreadPool`
	```
	while  (true)  {
		accept  connection;
		create  task  which  will  deal  with  the  client;
	}
	```

дальнейшую идеалогию работы с большим количеством клиентов см. NIO.md