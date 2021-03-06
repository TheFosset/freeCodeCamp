---
title: Streams
localeTitle: Streams
---
## Streams

Потоки доступны в базовом API Node.js как объекты, которые позволяют считывать или записывать данные непрерывным образом. В принципе, поток делает это в кусках по сравнению с буфером, который выполняет бит за бит, тем самым делая его медленным процессом.

Доступны четыре типа потоков:

*   Чтение (потоки, из которых считываются данные)
*   Writable (потоки, на которые записаны данные)
*   Дуплекс (потоки, которые являются читаемыми и записываемыми)
*   Трансформация (Дуплексные потоки, которые могут изменять данные по мере их чтения и записи)

У каждого доступного типа есть несколько методов. Некоторые из них:

*   данных (это выполняется, когда доступны данные)
*   end (это срабатывает, когда нет данных, оставшихся для чтения)
*   ошибка (это выполняется, когда есть ошибка приема или записи данных)

### труба

В программировании концепция `pipe` не нова. Системы на основе Unix прагматично использовали его с 1970-х годов. Что делает труба? `pipe` обычно соединяет источник и пункт назначения. Он передает выход одной функции в качестве входа другой функции.

В Node.js `pipe` используется одинаково, для сопряжения входов и выходов различных операций. `pipe()` доступна как функция, которая берет читаемый поток источника и присоединяет вывод к потоку назначения. Общий синтаксис может быть представлен как:

```javascript
src.pipe(dest); 
```

Функции нескольких `pipe()` также могут быть соединены вместе.

```javascript
a.pipe(b).pipe(c); 
 
 // which is equivalent to 
 
 a.pipe(b); 
 b.pipe(c); 
```

### Чтение потоков

Потоки, которые создают данные, которые могут быть присоединены как входные данные к записываемому потоку, называются Readable stream. Чтобы создать читаемый поток:

```javascript
const { Readable } = require('stream'); 
 
 const readable = new Readable(); 
 
 readable.on('data', chunk => { 
  console.log(`Received ${chunk.length} bytes of data.`); 
 }); 
 readable.on('end', () => { 
  console.log('There will be no more data.'); 
 }); 
```

### Считываемый поток

Это тип потока, который можно `pipe()` данные из читаемого источника. Чтобы создать поток, доступный для записи, мы используем конструкторский подход. Мы создаем объект из него и передаем несколько параметров. Метод принимает три аргумента:

*   кусок: буфер
*   кодирование: преобразование данных в удобочитаемую форму
*   callback: функция, которая вызывается, когда данные обрабатываются из блока

```javascript
const { Writable } = require('stream'); 
 const writable = new Writable({ 
  write(chunk, encoding, callback) { 
    console.log(chunk.toString()); 
    callback(); 
  } 
 }); 
 
 process.stdin.pipe(writable); 
```

### Дуплексные потоки

Дуплексные потоки помогают одновременно реализовать как считываемые, так и записываемые потоки.

```javascript
const { Duplex } = require('stream'); 
 
 const inoutStream = new Duplex({ 
  write(chunk, encoding, callback) { 
    console.log(chunk.toString()); 
    callback(); 
  }, 
 
  read(size) { 
    this.push(String.fromCharCode(this.currentCharCode++)); 
    if (this.currentCharCode > 90) { 
      this.push(null); 
    } 
  } 
 }); 
 
 inoutStream.currentCharCode = 65; 
 process.stdin.pipe(inoutStream).pipe(process.stdout); 
```

Поток `stdin` передает считываемые данные в дуплексный поток. Эта `stdout` помогает нам видеть данные. Читаемые и записываемые части дуплексного потока полностью независимы друг от друга.

### Преобразовать поток

Этот тип потока представляет собой более сложную версию дуплексного потока.

```javascript
const { Transform } = require('stream'); 
 
 const upperCaseTr = new Transform({ 
  transform(chunk, encoding, callback) { 
    this.push(chunk.toString().toUpperCase()); 
    callback(); 
  } 
 }); 
 
 process.stdin.pipe(upperCaseTr).pipe(process.stdout); 
```

Данные, которые мы потребляем, такие же, как в предыдущем примере дуплексного потока. Дело в том, что `transform()` не требует реализации методов `read` или `write` . Он объединяет оба метода.

### Зачем использовать потоки?

Поскольку Node.js является асинхронным, поэтому он взаимодействует, передавая обратные вызовы функциям с диском и сетью. Приведенный ниже пример читает данные из файла на диске и отвечает на него по сетевому запросу от клиента.

```javascript
const http = require('http'); 
 const fs = require('fs'); 
 
 const server = http.createServer((req, res) => { 
  fs.readFile('data.txt', (err, data) => { 
    res.end(data); 
  }); 
 }); 
 server.listen(8000); 
```

Вышеприведенный фрагмент кода будет работать, но все данные из файла сначала войдут в память для каждого запроса, прежде чем записывать результат обратно на запрос клиента. Если файл, который мы читаем, слишком велик, это может стать очень тяжелым и дорогостоящим вызовом сервера, так как он будет потреблять много памяти для продвижения процесса. Пользовательский опыт на стороне клиента также будет страдать от задержки.

В этом случае, если мы будем использовать потоки, данные будут отправляться на запрос клиента как один фрагмент за раз, как только они будут получены с диска.

```javascript
const http = require('http'); 
 const fs = require('fs'); 
 
 const server = http.createServer((req, res) => { 
  const stream = fs.createReadStream('data.txt'); 
  stream.pipe(res); 
 }); 
 server.listen(8000); 
```

Здесь `pipe()` заботится о записи или в нашем случае, отправляя данные с объектом ответа и как только все данные считываются из файла, чтобы закрыть соединение.

Примечание. `process.stdin` и `process.stdout` строятся в потоках в глобальном объекте `process` предоставляемом Node.js API.