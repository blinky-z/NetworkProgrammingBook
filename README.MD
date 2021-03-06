## Помощник в освоении сетевого программирования

### Что такое многопоточность
**Многопоточность** - это технология, которая позволяет существовать множеству тредов внутри одного процесса.
Все треды могут работать с другими запущенными тредами в одном процессе, также они разделяют память между собой (shared память).

Чтобы увидеть это на деле, давайте напишем и запустим такую простую программу, которая создаст много тредов и поставим таймаут, чтобы не дать тредам сразу закрыться:

```c++
#include <iostream>
#include <cstddef>
#include <unistd.h>
#include <thread>

void print(int i) {
    std::cout << i << std::endl;
}

void threadCreate(int i, int j) {
    for (i; i <= j; i++) {
        std::thread new_thread(print, i);
        new_thread.detach();
    }
}

int main() {
    std::thread new_thread1(threadCreate, 0, 5);
    new_thread1.detach();
    std::thread new_thread2(threadCreate, 6, 10);
    new_thread2.detach();

    sleep(10);

    return 0;
}

```

Откроем *htop* и посмотрим на поведение программы:

![htop](https://i.imgur.com/JFOwFDC.png "htop screen")

Мы видим, что создано множество тредов, и они имеют общую память.

---

### Что такое concurrency, parallelism
**Concurrency** - означает то, что задачи могут запускаться и **работать, прерываясь, вместе**. То есть какой-то промежуток времени работает одна программа, потом другая.

**Parallelism** - означает то, что задачи работают буквально **одновременно** (для этого нужен многоядерный процессор, на одноядерном такое невозможно).

Обе технологии на примере:

![concurrency and parallelism comparison](https://i.stack.imgur.com/V5sMZ.png "concurrency and parallelism comparison")

---

### Как связаны многопоточность и concurrency
Если параллелизм невозможен, то треды не будут выполняться параллельно, но будут выполняться concurrenly, то есть какой-то отрезок времени столько тредов, сколько возможно.
То есть, если имеется 4-ядерный процессор - то выполняется 4 треда параллельно, остальные ожидают, потом выполняются другие треды и т.д.

---

### Как связаны многопоточность и parallelism
Многопоточность на многоядерном процессоре - пример параллелизма. Треды могут выполняться сразу одновременно на нескольких ядрах.

---

### Как связаны многопоточность и асинхронность
Ассинхронная работа задач может быть достигнута несколькими способами: передать обработку задачи другому процессу, системе, другому коммпьютеру в сети, но основной способ -
дать другому треду обработать данную задачу.

---

### Как связаны concurrency и асинхронность
Так как асинхронных операций может быть множество, они будут выполняться concurrently.

---

### Что такое асинхронность
**Асинхронность** - означает то, что при выполнении какой-либо задачи не ожидается немедленный результат, а обработка данного действия продолжит выполняться в фоне,
тогда как вызывающий данную задачу поток программы продолжит работать дальше и получит результат ранее вызванного API позднее, когда результат будет готов.

---

### Что такое блокирующий I/O
**Блокирующий I/O** - при вызове I/O функций на блокирующем сокете поток, из которого была вызвана функция, заблокируется, ожидая ответа.
То есть поток будет простаивать в это время, не делая какую либо полезную работу.

Например, функции чтения и записи при выполнении на блокирующем сокете заблокируют поток в ожидании ответа, пока другая сторона соединения не пришлет данные.
Блокировки и проблема пустой, неэффективной траты времени могут быть решены 2 способами:
1) Сделать non-blocking I/O
2) Сделать мультиплексирование

Мы разберем эти способы далее.

---

### Что такое неблокирующий I/O
**Неблокирующий I/O** - при вызове I/O функций на неблокирующем сокете поток, из которого была вызвана функция, не будет блокироваться и после вызова к нему сразу же вернется
управление.

Однако, пока данные не будут готовы, вместо ответа на I/O функцию будет присылаться специальная ошибка, означающая то, что *буфер отправки сокета* слишком мал для отправляемого
сообщения), или же *буфер приема сокета* пуст (данные еще не были готовы для отправки другой стороной, т.е. читать нечего).
Значит, нам придется опрашивать все файловые дескрипторы в цикле, и только при готовности данных обработать их.

Но такой подход имееет минусы, поскольку:
1) Потребуется многократно опрашивать сокеты и вследствие этого возникнет большая нагрузка на процессор
2) При увеличении кол-ва сокетов, с таким подходом нагрузка возрастет еще сильнее.

То есть опрашивать постоянно в цикле - неэффективно, и принципиальных отличий в времени от блокирующего режима в принципе то и нет, а нагрузка может быть выше.
Поэтому тут приходит на помощь решение в виде *мультиплексирования*.

---

### Что такое мультиплексирование I/O
**Мультиплексирование I/O** - передает задачу проверки готовности сокетов для операций чтения/записи **ядру**, избавляя разработчика от самостоятельной проверки сокетов.

В Linux доступны следующие механизмы мультиплексирования:
 * *select*
 * *poll*
 * *epoll*

Все эти механизмы могут работать как с блокирующими сокетами, так и неблокирующими (в случае epoll существует исключение - см. ниже). Далее мы разберем все 3 мультиплексора и их
особенности.

---

### Модели I/O

Рассмотрим все модели I/O, доступные в Linux, а также то, с помощью чего они имплементируются.

#### Блокирующий синхронный I/O
Одна из наиболее распространненых моделей I/O.

Поток блокируется при вызове I/O сисколла в ожидании ответа, т.е. не происходит никакой работы, пока системный вызов I/O (read/write) не будет завершен.

Такая модель называется синхронной, ведь результат происходит в то же время. Результат никак не ожидается позже, он придет сразу же после вызова функции. Сразу - **не означает
немедленно**, это означает то, что результат придет **сразу же, как будет готов ответ**, и работа программы продолжится дальше.

Полностью понять синхронность можно, если рассмотреть работу со *стороны самой программы*:

Что же происходит при вызове API чтения или записи на блокирующем сокете на стороне программы? Да, поток, из которого была вызвана функция, блокируется, однако, со стороны
программы это просто выглядит как вызов функции, который занимает долгое время, и после прихода ответа работа продолжается дальше. То есть, можно сказать, что действия произошли
синхронно, ответ пришел сразу же после вызова.

**Что нужно для реалзиации данной модели I/O:**

* Блокирующие сокеты

#### Неблокирующий синхронный I/O
В данной модели I/O ответ возвращается **немедленно** после вызова сисколла, и **поток не блокируется**. В зависимости от готовности I/O придет или ошибка, означающая то,
что чтение/запись еще не готовы, или же сам результат.

Такая модель все еще остается синхронной, ведь результат приходит в то же время вызова API, как и в случае с блокирующим синхронным I/O.

Данная модель менее эффективна, чем блокирующий синхронный I/O, ведь придется опрашивать сокеты постоянно, ожидая видеть там результат и программа будет тратить процессорное
время в отличие от блокирующего I/O, где программа просто спала в ожидании результата.
Однако, здесь существует и достоинство: мы имеем больше возможностей для дополнительной работы между опросами, ведь поток не блокируется.
Конечно, мы могли бы создавать отдельный тред для дополнительной работы в случае с блокирующим I/O, однако тогда появятся расходы на создание тредов и переключение между тредами.

**Что нужно для реалзиации данной модели I/O:**

* Неблокирующие сокеты

#### Блокирующий асинхронный I/O
В данной модели поток блокируется не на самом вызове сисколла чтения/записи, а на вызове блокирующего сисколла мультиплексора (select, poll, epoll_wait).
Данный I/O позволяет отслеживать не только один файловый дескриптор, а сразу множество. Например, в non-blocking I/O приходилось проверять все сокеты с цикле, тут же
данную работу выполняет ядро, что будет намного эффективнее самостоятельной проверки всех дескрипторов.

Однако, почему же данная модель называется асинхронной?

Для начала, *асинхронность* - буквально означает **не синхронность**.

Рассмотрим на примере мультиплексора *select*. Дело в том, что результат получается не сразу же после вызова API, а только после вызова select(). Select дает знать программе, что
на определенном декскрипторе есть возможность прочитать/отправить данные, и только после этого программа совершит действия с сокетами, т.е. результат получается не сразу же после
вызова системного вызова I/O, а только после вызова select.

Полностью понять данную модель можно понять исходя из самого определения асинхронности - *получаем результат не сразу, оставляя обработку вызова в фоне*, и продолжаем работу
программы, получая результат фоновой работы позже.
**Select** - и есть та фоновая работа, ведь **мы отдаем задачу по определению сокетов мультиплексору**.

**Что нужно для реалзиации данной модели I/O:**

* Блокирующие/неблокирующие сокеты
* Механизмы мультиплексирования (select, poll. epoll).

#### Неблокирующий асинхронный I/O
В данной модели **поток не блокируется никогда** при вызове сисколла I/O и продолжает свою работу, в то время как работа API будет выполняться в фоне на стороне ядра, и, когда
результат будет готов, ядро вернет его.

Это наиболее эффективная модель. Эффективна она потому, что поток не блокируется и программа может выполнять полезную работу (например, запись каких-то файлов в бд), ожидая
ответа.

**Что нужно для реалзиации данной модели I/O:**

* Неблокирующие сокеты
* Библиотека, предоставляющая возможность асинхронного I/O (такие как *<aio.h>* (Linux), *libevent*, *libev* и другие)

---

### Мультиплексоры

### select
**Select** - *level-triggered* мультиплексор.

Ключевой модуль данного мультиплексора - это структура **fd_set**. Мы загружаем в fd_set все наши сокеты и вызываем функцию **select()**, передавая в качестве параметра наше
множество сокетов (структуру fd_set). На этом моменте программа блокируется, ожидая ответа (на проверку) от ядра. Далее мы определяем, содержится ли какой-либо сокет и
возвращенном множестве, и если да, значит данный сокет готов принимать/отравлять данные.

#### Особенности select

**Имплементация структуры *fd_set*:**

```c
#ifndef FD_SETSIZE
#define FD_SETSIZE  1024
#endif
#define NBBY    8       /* number of bits in a byte */
typedef long    fd_mask;
#define NFDBITS (sizeof (fd_mask) * NBBY)   /* bits per mask */
#define howmany(x,y)    (((x)+((y)-1))/(y))
typedef struct _types_fd_set {
    fd_mask fds_bits[howmany(FD_SETSIZE, NFDBITS)];
} _types_fd_set;

#define fd_set _types_fd_set
```

1) Select возвращает структуру fd_set, где хранятся только те сокеты, на которых есть результат. Однако, из-за особенности дизайна данной структуры мы не можем проходить по
структуре и проверять каждый сокет, поэтому нам придется хранить все сокеты, с которыми мы работаем, где-то еще, например в std::set или же в std::vector, для того чтобы
проверять каждый наш сокет функций **FD_ISSET**, которая позволяет нам узнавать, содержится ли сокет в возвращенном множестве fd_set.

2) Select не хранит номер файлового дескриптора напрямую. Вместо этого для определения сокета используется **поле битов** размера FD_SETSIZE (длина которого которого
фиксирована на 1024), которое состоит из множества *long* значений. Оно хранится в самой структуре fd_set. Именно поэтому мы не можем просто проходить по множеству, доставая
какой-либо сокет, так как требуется работа с битмаской.

Для того, чтобы понять лучше, рассмотрим работу с битмаской внутри структуры fd_set:

Допустим, что есть 3 сокета с номерами 3, 7 и 15. Пусть мы вызвали функцию *FD_SET()* и зарегистрировали все сокеты в нашу структуру. При вызове *FD_SET()* происходит назначение
бита с индексом значения сокета на **1**.

Тогда после проделывания всех вызовов бит поле будет выглядеть так:

```   1 long            2 long            3 long            4 long
00010001.00000001:00000000.00000000:00000000.00000000:00000000.00000000:....
   ^   ^        ^
   0   0        1
   3   7        5
```

3) Select полностью перезаписывает множество fd_set, не позволяя переиспользовать его на следующей итерации, то есть на каждой итерации или придется добавлять все сокеты снова
в множество, или же восстанавливать предварительно самостоятельно сохранненую копию множества функцией *FD_COPY(&fdset_orig, &fdset_copy)*.

4) Select не умеет проверять различные флаги, а может только проверять на готовность чтения/записи или ошибки и в зависимости от результата использовать специально предназначенную
для этого структуру.
Поэтому всегда используется 3 отдельных множества для чтения, записи и исключений - *fd_set readfds*, *fd_set writefds*, *fd_set errorfds* соответственно.

Достаточно посмотреть на объвление функции *select()* и убедиться в этом:

```c++
int select(int nfds, fd_set *restrict readfds,
       fd_set *restrict writefds, fd_set *restrict errorfds,
       struct timeval *restrict timeout);
```

---

### Мультиплексор и non-blocking сокеты, Проблемы
Здесь мы рассмотрим, какая связь между мультиплесорами и non-blocking сокетами.

А также расмотрим, как решается проблема многократного опроса неблокирующих сокетов с использованием механизма мультиплексирования **select** и увидим, почему при использовании
мультиплексора лучше использовать неблокирующие сокеты, чем блокирующие.

#### [Мультиплексор] Проблема - чтение больших данных с сокета

Допустим, что мы вызвали select с множеством блокирующих сокетов. Тогда, если объем данных слишком большой, а буфер чтения маленький, то нам придется
вызвать select много раз. Ведь если сокет блокирующий, то он будет заблокирован, пока другая сторона не прочитает данные. Неблокирующий сокет решает данную проблему - он дает
возможность слать данные в сокет (в данном случае - со стороны клиента), не дожидаясь готовности *сокет буфера* к записи.

**Сравнение:**

Условие: ```Объем данных - 1024 байта. Буффер - 64 байта```
   * **Блокирующий сокет**: потребуется вызвать функцию select 64 раза и читать за каждую итерацию очередные 64 байта - всего **64 сисколла**
   * **Неблокирующй сокет**: select будет вызвана всего 1 раз и мы прочитаем данные 64 раза на стороне программы - всего **1 сисколл**

Таким образом, *non-blocking* режим сокетов решил проблему многократного вызова сисколла **select()**.

#### [Non-blocking сокеты] Проблема - многократный опрос сокетов на готовность чтения/записи или ошибки.

Мультиплексор решает данную проблему и избавляет от необходимости самостоятельно опрашивать все сокеты.

Так, такой *неэффективный* код:
```c++
    while (1) {
        for (int i = 0; i < socketsNumber; i++) {
            char readBuffer[1024];
            memset(&readBuffer, '\0', sizeof(readBuffer));
            ssize_t recvSize = recv(sockets[socketsNumber], &readBuffer, sizeof(readBuffer), MSG_NOSIGNAL);
            if (recvSize == -1 && errno == EWOULDBLOCK) {
                continue;
            }
            else {
                ...
            }
        }
    }
```

Превращается в такой код:
```c++
 select(maxFd + 1, &workSockets, nullptr, nullptr, nullptr);

        for (auto currentSock = clientSockets.begin(); currentSock != clientSockets.end(); currentSock++) {
            if (FD_ISSET(*currentSock, &workSockets)) {
                char readBuffer[1024];
                memset(&readBuffer, '\0', sizeof(readBuffer));
                int recvSize = recv(*currentSock, &readBuffer, sizeof(readBuffer) - 1, MSG_NOSIGNAL);

                ...
```

который будет *эффективнее* и *быстрее*, ведь нам не придетсятся многократно опрашивать все сокеты, а только те, о которых мы точно знаем, что какой-то сокет готов к
чтению/отправке.

-------

### poll
**Poll** - *level-triggered* мультиплексор, который завязан на событиях.

Poll работает с *ивентами* и имеет следующие основные *флаги ивентов*:

1) **POLLIN** - на сокете доступны данные для чтения
2) **POLLOUT** - буфер сокета готов для записи
3) **POLLERR** - дескриптор вернул ошибку
4) **POLLHUP** - закрыто соединение на данном сокете
5) **POLLVAL** - некорректный файловый дескриптор

А также несколько других, которые редко используются.

Как мы видим, в poll имеется больший выбор флагов произошедших ивентов, в отличие от *select*.
Также, еще одно (и весьма существенное) достоинство по сравнению с select - в poll **нет лимита на количество обслуживаемых сокетов**.

#### Особенности poll

В качестве множества сокетов используется *массив структур **pollfd***

**Имплементация структуры pollfd:**

```c++
struct pollfd {
    int fd;         // the socket descriptor
    short events;   // bitmap of events we're interested in
    short revents;  // when poll() returns, bitmap of events that occurred
};
```

1) Из-за особенности устройства структуры *pollfd* poll не изменяет исходное множество и не перезаписывает его, что избавляет от лишних расходов на повторное копирование сокетов
на стороне программы в множество на каждой итерации. Это достигается путем очищения значения поля *revents* в каждой структуре исходного массива.

2) poll, в отличие от select, использует для каждого сокета свою структуру и не хранит бит поля для определения номера сокета.

Хорошо ли это? Чаще при разработке приложения, которое будет обслуживать *менее 1024* сокетов, стоит выбрать мультиплексор **select**.

Допустим, что у нас есть *<1024* (ограничение FD_SETSIZE select'а) сокетов - 128, произошло около 1000 итераций.
Давайте произведем расчеты:

**select:** произошло копирование в ядро 1024(FD_SETSIZE) * 3(fd_set) * 1000 бит = *3072000 бит*

**poll:** произошло копирование в ядро (32int + 16short + 16short) * 128 * 1000 бит = *8192000 бит*

Очевидно, что poll будет проигрывать в затратах на копирование, однако, не стоит забывать, что в случае с select существует также затраты и на повторное копирование на каждой
итерации, в отличие от poll.

---

### epoll
**Epoll** - наиболее эффективный и более новый, однако также и более сложный в обращении мультиплексор, доступный только в Linux. Также это единственный event-based мультиплексор
в Linux.

Epoll способен работать в двух режимах:

1) level-triggered
2) edge-triggered <- *именно из-за поддержки данного режима epoll является event-based мультиплексором*

#### level-triggered
При работе в таком режиме на вызове epoll будут возвращаться те файловые дескрипторы, в которых **интересующие клиента ивенты, даже если они были еще на прошлом вызове**.
То есть, если на прошлом вызове все данные с сокета не были прочитаны, тогда при следующей итерации на вызове epoll будут возвращены снова данные сокеты.

#### edge-triggered
При работе в таком режиме будут возвращаться те файловые дескрипторы, где произошли **только новые события**. Если даже программа прочитала не все данные на каком-либо из
сокетов, то она все равно будет заблокирована до наступления нового события.

#### Особенности epoll
1) Во-первых, epoll эффективнее из-за другого, отличающегося от select и poll, механизма отслеживания активных сокетов и тредов.
Ядро строит список интересующих программу событий для каждого сокета. Так как ядро знает все отслеживаемые дескрипторы, оно может регистрировать произошедшее событие на них,
**даже если не было вызова *epoll_wait()***. Именно данная особенность позволяет использовать *edge-triggering* режим работы мультиплесора. В результате этого, нет необходимости
возвращать все множество, а только такое, где на сокетах произошли какие-либо события (event'ы). Соответственно, на стороне программы не придется проходить по всем сокетам,
а только по тем, где есть события.

2) epoll плохо подходит для работы с большим количеством коротко живущих соединений. Каждое принятое соединение в *epoll* требует вызова двух системных вызовов вместо одного
системного вызова в *poll*:
* **epoll:** accept() + epoll_ctl()
* **poll:** accept()

---

**Именно в *edge-triggering* режиме epoll становится особенно эфективным**

Рассмотрим поведение ядра на вызове *epoll_wait()* при режиме **edge-triggered** и сравним с поведением при **level-triggered** режиме

- **level-triggered**: вызов epoll_wait будет проходить по списку дескрипторов и проверять, есть ли на каком либо из сокетов условие, которое позволит выйти из
epoll_wait без блокировки (если остались непрочитанные данные, например, и соответственно, доступно чтение)
- **edge-triggered**: такая проверка не производится, то есть нет необходимости проходить по всем сокетам, а будут отслеживаться только новые события, за счего чего
вызов становится намного эффективнее и сложность вызова epoll_wait() становится **O(1)** за счет того, что все события уже регистрировались даже тогда, когда epoll_wait() еще
не был вызван, ведь epoll имеет список всех интересующих программу событий для каждого сокета (см. *Особенности epoll*).

---

### Select vs poll vs epoll
**Когда выбрать какой механизм?**

**Функциональность (select, poll):**

Select и poll предоставляют в основном одинаковую функциональность, однако есть и отличия:
1) *Select()* копирует множество, принимая его и возвращает обратно копию множества с выставленными флагами. Poll же, может работать с тем же множеством сокетов, который был
передан как аргумент в вызов *poll()*, избавляя от лишних расходов на копирование.
2) Poll может работать с большим количеством файлов (файловых дескрипторов), в то время как select имеет фиксированный размер множества - 1024, и не может быть изменен (на Linux).
3) poll предоставляет больший выбор ивентов по сравнению с select()
4) poll предоставляет возможность работы больше чем с 1024 сокетами
5) Оба, select и poll, **не могут** работать с сокетами, которые находятся под вызовом select() или poll().

**Скорость:**
1) Оба, select и poll, работают *с файловыми дескрипторами* при вызове *select()* и *poll()* за **O(n)**.
2) Затраты на копирование - чаще при маленьких кол-вах сокетов выиграет **select**: в select на каждый сокет тратится по *3 бита* (но остается фиксированным размер массива - 1024),
в отличие от *64 бит* в случае с poll

---

**Когда выбирать select:**
1) Портативность приложения. Select - самый старый механизм мультиплексирования из доступных, поэтому можно быть полностью уверенным, что на каждой машине, где будет
запускаться программа, данный механизм мультиплексирования будет поддерживаться.
2) Трбеуется возможность работы с таймаутом в виде **наносекунд**, что невозможно в poll и epoll, где работа с таймаутом происходит с использованием миллисекунд.

**Когда выбирать poll:**
1) Требуется поддержка платформ кроме Linux, а значит, невозможна работа с epoll
2) Принимаемые соединения живут *недолго*

**Когда выбирать epoll:**
1) Приложение использует *>1 потоков*, именно в таком приложении epoll раскрывает себя полностью. В сингл-тредном приложении использование epoll почти что не будет быстрее,
чем использование poll
2) Принимаемые соединения живут *долго*
3) Когда требуется возможность удалять/добавлять сокеты в множество даже тогда, когда поток заблокирован на ожидании ивентов. Epoll поддерживает данную возможность и она
официально задокументирована.

---

### Как связан epoll с blocking и non-blocking I/O
В epoll в **edge-triggered** режиме нельзя использовать блокирующие сокеты потому, что **требуется прочитать абсолютно все имеющие на момент срабатывания ивента данные** и
**нельзя оставлять непрочитанные** на следующую итерацию и вызов epoll_wait(). А при использовании блокирующих сокетов, epoll не может проверить сокет на новые данные, так как
требуется прочесть уже передающиеся.

Поэтому при работе в режиме *edge-triggered* требуется использование non-blocking сокетов.

---

### Что такое event loop
Event loop относится к такой парадигме программирования, как *программирование с управлением по событиям* (event-driven programming).
Это означает то, что в программа будет реагировать не на готовности чтения, записи (как мы делали в poll, select сервере), а *события* на них (например, щелчки мышкой,
нажатия клавиш). Все это регистрируется как ивент и в зависимости от ивента запускается принадлежащая ивенту *callback функция*.

В основе СОП (событийно ориентированного программирования) лежит как раз так и *event loop*, который ожидает ивентов и запускает в зависимости от ивента необходимую callback
функцию.

Так выглядит event loop в псевдокоде:
```
function main
    initialize()
    while message != quit
        message := get_next_message()
        process_message(message)
    end while
end function
```

---

### Как связан event loop и асинхронность
В event loop все события обрабатываются вызываемой *callback функцией* в отдельном потке. Так как каждый ивент обрабатывается отдельно от основного потока, основной
поток event лупа может работать дальше, не блокируясь, принимая новые ивенты и обслуживая их, не блокируясь.

---

### Как связан epoll с event loop
Так как epoll обладает режимом работы edge-triggering, а значит epoll тоже относится к парадигме СОП, так как в edge-triggering epoll все завязано на событиях.

---

### Как связан epoll с асинхронностью
Epoll связан с асинхронностью, так как epoll работает с ивентами через режим edge-triggered. На каждый нужный ивент делаются *callback функции*, и когда наступает данный ивент,
cb функция сработает и обработает данный ивент, в то время как программа может делать свою полезную работу. То есть, программа не блокируется и не ждем прихождения результата,
а работает дальше. Поэтому epoll связан с асинхронностью - ведь его условие - это выполнение действий в фоне, не блокируя поток, из которого было вызвано API.

---

### Какая связь между epoll и многопоточностью
С помощью многопоточности и с использованием Epoll можно распределить обработку клиентов на несколько тредов с собственными Epoll сокетами. Тогда основной поток сервера должен
принять клиента и прикрепить его к одному из Epoll сокетов.

---

### Какая связь между epoll и concurrency
Важно при разработке веб сервера учитывать возможности процессора, то есть при использовании множества тредов (а значит, и множества Epoll сокетов) важно обеспечить
параллельную работу тредов, чтобы остальные треды **не простаивали в ожидании**, иначе возникнет *concurrency*.

---

### Как работает Node.js сервер
В Node.js сервере используется **Single-Threaded Event Loop Non-blocking I/O модель**. Это означает то, что в node.js запросы обрабатываются **в единственном треде**,
**не блокируя поток** при обработке каждого запроса, а с запросами node.js работает как с **ивентами**, которые мы рассматривали ранее. Все это вкупе делает его особенно
интересным, ведь во многих веб серверах используется многопоточная модель.

Но перед тем, как переходить к модели, использующейся в Node.js, давайте посмотрим, как устроена стандартная многопоточная модель обработки запросов:

#### Многопоточная модель и её недостатки

Веб сервер устроен так, что он имеет *фиксированный* пул тредов, а в самом сервере запущен бесконечный цикл, которые принимает соединения и запросы от них.

Тогда, последовательно отправки и принятия запросов будет такая:
1) Клиент шлет запрос серверу
2) Веб сервер принимает запросы
    1) Веб сервер выбирает первый из пришедших запросов
    2) Веб сервер выбирает из тредов свободный тред, если такой есть
    3) Веб сервер назначет обработку запроса данному треду
    4) Данный тред прочитает, обработает запрос, построит его, а также выполнит блокирующие операции (если таковые есть. Например: обращение к файловой системе,
    базе данных и др.). Это одна из причин, почему требуется тред для обработки запроса - чтобы не блокировать осовной поток
    5) Тред вернет построенный ответ обратно основному потоку
    6) Веб сервер отправляет данный ответ клиенту

**Недостатки такой архитектуры веб-сервера:**
* Обработка concurrent запросов сложна
* Когда количество concurrent запросов растет, сервер будет использовать все больше тредов
* Если все треды заняты, клиенту придется ждать неопределенное время
* Трата времени на блокирующие I/O операции (в тредах), из-за этого какой-либо тред может обслуживать клиента дольше обычного

---

**Перед тем, как переходить к модели I/O, использующейся в Node.js, рассмотрим, как устроен event loop в Javascript подробнее:**

##### Event table
Javascript использует такую струтуру данных, как **event table**. Данная структура знает, какая (callback) функция должна быть вызвана после того, как сработал какой-то ивент.

**Предназначение *event table*:** сопоставлять ивенты и относящиеся к ним функции. Event table **не вызывает функции самостоятельно**.

##### Event queue
Данная структура данных предназначена для **хранения** callback функций, которые должны быть вызваны. *Event queue* получает функции от **event table**.
*Event queue* очень похожа на *стэк* - функции добавляются в конец структуры, а берутся с начала структуры.

**Предназначение event *event queue*:** Данная структура определяет порядок выполнения функций.

Но, ведь функции должны быть отправлены в *call stack*? Здесь и приходит на помощь **event loop**.

##### Event loop
Эта конструкция проверяет, пуст ли call stack. Если он пуст, то event loop смотрит в *event queue*. Если в event queue имеются оидающие функции, то event loop помещает их в call
stack.

Чтобы еще лучше понять все эти вещи, давайте рассмотрим работу данных конструкций на примере:

Пусть, у нас есть такой код:
```js
setTimeout(() => console.log('first'), 0)
console.log('second')
```

Тогда последовательность работы будет такая:
1) Первая строчка - Javascript видит ивент (таймаут) и добавляет его в event table - сопоставляя данный ивент с функцией ```console.log('first')```, и программа продолжит
работать дальше
2) Вторая строчка - в call stack будет добавлена функция ```console.log('second')``` и исполнена, соответственно, "second" будет выведено на экран первым
3) Event loop все это время смотрел в call stack, однако там была наша функция, декларированная на второй строчке. Теперь, когда call stack пуст, event loop смотрит в
*event queue* и видит там нашу функцию, которая была добавлена раннее. Он перемещает её в call stack.
4) Функция в call stack'е вызывается и исполняется

Соответственно, вывод на экран будет таким:
```
second
first
```

Так работает Javascript - с использованием *event table*, *event queue* и *event loop*.

---

#### Однопоточная event loop модель Node.js
Теперь мы переходим непосредственно к **Single-Threaded Event Loop модели**, использующейся в Node.js сервере. Следует отметить, что node.js построен на основе Javascript
event loop, которые мы разобрали ранее.

Ключевой модуль Node.js - **Event loop**.

Node.js Веб сервер устроен так, что он также имеет **фиксированный** пул тредов, а ивенты обрабатываются в **Event loop**, который является основным потоком. Все запросы
принимаются в Event Loop и ответы отправляются оттуда же.

Рассмотрим последовательность обработки клиентов в **Node.js веб сервере**:
1) Клиент отправляет запрос серверу
2) Основной поток сервера (Event Loop) принимает запрос и помещает его в *Event Queue*.
3) Event loop проверяет, есть ли ожидающие запросы в Event Queue и пуст ли *call stack*
4) Если запросов в *Event queue* нет, то event loop продолжает ожидать новых запросов
4) Если запрос в *Event queue* есть и *call stack* пуст, то Event loop достает запрос с начала очереди (в начале очереди будут лежать те запросы, которые пришли раньше,
а в конце - которые пришли позднее остальных) и анализирует, требует ли данный запрос каких-либо блокирующих I/O операций.
    * Если данный запрос **не требует** выполнения блокирующих I/O операций, то
     1) Event loop помещает соответствующую cb функцию в stack, где она будет вызвана и выполнена
     2) основной поток оправляет построенный ответ клиенту
     Мы можем обработать запрос сразу в основном потоке потому, что в запросе нет блокирующих операций, которые могли бы заблокировать поток и работу event loop.
    * Если данный запрос **требует** выполнения блокирующих I/O операций, то:
     1) Основной поток проверяет, есть ли доступные треды
     2) Выбирает тред и назначет обработку запроса данному треду
     3) Данный тред прочитает, обработает запрос, построит его, а также выполнит блокирующие операции
     4) Основной поток оправляет построенный ответ клиенту

**Достоинства данной модели:**
* Обработка concurrent запросов легка из-за наличия *Event queue*
* Если кол-во concurrent запросов будет расти, то нам не придется создавать большое количество тредов
* Данная модель использует меньше тредов, чем многопоточная

**Вывод:**

1) Из-за наличия **event table** и **event loop** мы способны обрабатывать некоторые запросы сразу в единственном потоке программы, где выполняется event loop
2) Из-за наличия **event queue** мы способны более легко обрабатывать concurrent запросы.

### Как работает Netty сервер
Netty сервер использует **асинхронную non-blocking event-driven** модель.

Давайте рассмотрим, как работает сервер, использующий данный фреймворк.

Netty построен на основе Java **NIO**, который, в свою очередь, работает с мультиплексорами select, poll, epoll (на Linux).

1) Клиент отправляет запрос веб серверу
2) Event loop принимает данный запрос
3) Соединение передается в другой event loop, который будет слушать события на данном соединении
4) Данный event loop будет получать ивенты с polling фазы (то есть вызова сисколла *select()*, *poll()*, *epoll_wait()*).
5) Event pool получает запрос и анализирует, потребуется ли выполнение blocking I/O операций
   * Если данный запрос **не требует** выполнения блокирующих I/O операций, то:
   1) Event loop обрабатывает данный запрос с помощью callback функции
   2) Отправляет данный ответ клиенту
   * Если данный запрос **требует** выполнения блокирующих I/O операций, то:
   1) Основной поток, есть ли доступные треды
   2) Выбирает тред и назначет обработку запроса данному треду
   3) Данный тред прочитает, обработает запрос, построит его, а также выполнит блокирующие операции
   4) основной поток отправляет построенный ответ клиенту

*Асинхронная неблокирующая* модель была достигнута за счет того, что вся работа по обработке клиентов была передана в другой event loop, а основной, который отвечает за
принятие соединений, благодаря этому, никогда не будет блокироваться.