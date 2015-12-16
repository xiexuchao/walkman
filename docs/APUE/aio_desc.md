AIO详解
------

  Linux中最常用的输入/输出(I/O)模型是同步I/O. 在这个模型中，当请求发出之后，应用程序就会阻塞，直到请求满足为止。这是一个很好的解决方案，因为调用应用程序在等待I/O请求完成时不需要使用任何CPU. 但是在某些情况下，I/O请求可能需要与其他进程产生交叠。可移植操作系统接口(POSIX)异步I/O(AIO)应用程序接口就提供了这种功能。在本文中，我们将对这个API概要进行介绍，并来了解一下如何使用它。
  
### AIO简介
  Linux异步I/O是Linux内核中提供的一个相当新的增强。它是2.6版本内核的一个标准特性，但是我们在2.4版本内核的补丁中也可以找到它。AIO背后的基本思想是允许进程发起很多I/O操作，而不用阻塞或等待任何操作完成。稍后或在接收到I/O操作完成的通知时，进程就可以检索I/O操作的结果。
  
### I/O模型
  在深入介绍AIO API之前，让我们先来探索一下Linux上可以使用的不同I/O模型。这并不是一个详尽的介绍，但是我们将试图介绍最常用的一些模型来解释它们与异步I/O之间的区别。下图给出了同步和异步模型，以及阻塞和非阻塞模型。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/io_models.png)
  
  每个I/O模型都有自己的使用模式，它们对于特定的应用程序都有自己的优点。本节将要简要对其一一进行介绍。
  
### 同步阻塞I/O
  最常用的一个模型是同步阻塞I/O模型。在这个模型中，用户空间的应用程序执行一个系统调用，这会导致应用程序阻塞。这意味着应用程序会一直阻塞，直到系统调用完成为止(数据传输完成或发生错误)。调用应用程序处于一种不再消费CPU而只是简单等待响应的状态，因此从处理的角度来看，这是非常有效的。
  
  图2给出了传统的阻塞I/O模型，这也是目前应用程序中最为常用的一种模型。其行为非常容易理解，其用法对于典型的应用程序来说都非常有效。在调用read系统调用后，应用程序会阻塞并对内核进行上下文切换。然后会触发读操作，当响应返回时(从我们正在从中读取的设备中返回), 数据就被移动到用户空间的缓冲区中。然后应用程序就会解除阻塞(read调用返回).
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/sync_io_model.png)
  
  从应用程序的角度来说，read调用会延续很长时间。实际上，在内核执行读操作和其他工作时，应用程序的确会被阻塞。
  
### 同步非阻塞I/O
  同步阻塞I/O的一种效率稍低的变种是同步非阻塞I/O。在这种模型中，设备是以无阻塞的形式打开的。这意味着I/O操作不会立即完成，read操作可能会返回一个错误代码，说明这个命令不能立即满足(EAGAIN或EWOULDBLOCK), 如图3:
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/sync_nonblock_io_model.png)
  
  非阻塞的实现是I/O命令可能并不会立即满足，需要应用程序调用许多次来等待操作完成。这可能效率不高，因为在很多情况下，当内核执行这个命令时，应用程序必须要进行忙碌等待，直到数据可用为止，或者试图执行其他工作。正如图3所示的一样，这个方法可以引入I/O操作的延时，因为数据在内核中变为可用到用户调用read返回数据之间存在一定的间隔，这回导致整体数据吞吐量的降低。
  
### 异步阻塞I/O
  另外一个阻塞解决方案是带有阻塞通知的非阻塞I/O. 在这种模型中，配置的是非阻塞I/O,然后使用阻塞select系统调用来确定一个I/O描述符何时有操作。使select调用非常有趣的是它可以用来为多个描述符提供通知，而不仅仅为一个描述符提供通知。对于每隔提示符来说，我们可以请求这个描述符可以写数据、有读数据可用以及是否发生错误的通知。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/async_block_io_model_select.png)
  
  select调用的主要问题是它的效率不是很高。尽管这是异步通知使用的一种方便模型，但是对于高性能的I/O操作来说不建议使用。
  
### 异步非阻塞I/O(AIO)
  最后，异步非阻塞I/O模型是一种处理与I/O重叠进行的模型。读请求会立即返回，说明read请求已经成功发起。在后台完成读操作时，应用程序然后会执行其他处理操作。当read的响应到达时，就会产生一个信号或执行一个基于线程的回调函数来完成这次I/O处理过程。
  ![](https://github.com/walkerqiao/walkman/blob/master/images/APUE/async_nonblock_io_model_aio.png)
  
  在一个进程中为了执行多个I/O请求而对计算操作和I/O处理进行重叠处理的能力利用了处理速度与I/O速度之间的差异。当一个或多个I/O请求挂起时，CPU可以执行其他任务；或者更为常见的是，在发起其他I/O的同时对已经完成的I/O进行操作。
  
  下一节将深入介绍这种模型，探索这种模型使用的API，然后展示几个命令。
  
### 异步I/O动机
  从前面I/O模型的分类中，我们可以看出AIO的动机。
  * 阻塞模型需要在I/O操作开始时阻塞应用程序。这意味着不可能同时重叠进行处理和I/O操作。 
  * 同步非阻塞模型允许处理和I/O重叠，但是这需要应用程序根据重现的规则来检查I/O操作的状态。
  * 剩下的就是异步非阻塞I/O了，它允许处理和I/O操作重叠进行，包括I/O操作完成的通知。
  
  总结: 阻塞就不能同时进行处理和I/O. 同步非阻塞可以同时进行处理和I/O, 但需要应用程序自己去检查I/O操作的状态。
  异步非阻塞能同时进行处理和I/O, 同时也会在I/O就绪时通知应用程序。

  除了需要阻塞之外，select函数所提供的功能(异步阻塞I/O)与AIO类似。不过，它是通过对通知事件进行阻塞，而不是对I/O调用进行阻塞。
  
  还有考虑同时进行处理和I/O操作，是基于CPU处理能力和磁盘的读写能力不在一个量级上。CPU处理能力远远高于磁盘读写能力。
  
### Linux上的AIO简介
  本节将探索Linux的异步I/O模型，从而帮助我们理解如何在应用程序中使用这种技术。
  
  在传统的I/O模型中，有一个使用唯一句柄标识的I/O通道。在Unix中，这些句柄是文件描述符(这对等同于文件、管道、套接字等等)。在阻塞I/O中，我们发起了一次传输操作，当传输操作完成或发生错误时，系统调用会返回。
  
  在异步非阻塞I/O中，我们可以同时发起多个传输操作。这需要每隔传输操作都有唯一的上下文，这样我们才能在它们完成时区分到底是哪个传输操作完成了。在AIO中，这是一个aiocb(AIO I/O Control Block)结构。这个结构包含了有关传输的所有信息，包括为数据准备的用户缓冲区。在产生I/O(称为完成)通知时，aiocb结构就被用来唯一标识所完成的I/O操作。这个API展示显示了如何使用它。
  
### AIO API
  AIO接口的API非常简单，但是它为数据传输提供了必须的功能，并给出了两个不同的通知模型。 表1给出了AIO的接口函数，本节稍后会更详细进行介绍。
  
### AIO API list
  * aio_read: 请求异步读操作
  * aio_error: 检查异步请求的状态
  * aio_return: 获得完成的异步请求的返回状态
  * aio_write: 请求异步写操作
  * aio_suspend: 挂起调用进程，直到一个或多个异步请求已经完成(或失败)
  * aio_cancel: 取消异步I/O请求
  * lio_listio: 发起一系列I/O操作
  
  每个API函数都使用aiocb结构开始或检查。这个结构有很多元素，但是清单1仅仅给出了需要(或可以)使用的元素。
```
/** aiocb structure **/
struct aiocb{
  int aio_fildes;
  int aio_lio_opcode;
  volatile void *aio_buf;
  size_t aio_nbytes;
  struct sigevent aio_sigevent;
  
  /** Internal fields **/
}
```
  sigevent结构告诉AIO在I/O操作完成时应该执行什么操作。我们将在AIO的展示中对这个结构进行探索。现在我们将展示各个AIO的API函数是如何工作的，以及我们应该如何使用它们。

#### aio_read
  aio_read函数请求对一个有效的文件描述符进行异步读操作。这个文件描述符可以表示一个文件、套接字甚至管道。aio_read函数的原型如下: `int aio_read(struct aiocb *aiocb)`;
  aio_read函数在请求进行排队之后立即返回。如果执行成功，返回值就为0；如果出现错误，返回值就为-1, 并设置errno的值。
  要执行读操作，应用程序必须对aiocb结构进行初始化。下面这个简短的例子就展示了如何填充aiocb请求结构，并使用aio_read来执行异步读请求(现在暂时忽略通知)操作。它还展示了aio_error的用法，不过我们将稍后再做解释。
  
```
#include <aio.h>

...

int fd, ret;
struct aiocb my_aiocb;
fd = open("file.txt", O_RDONLY);
if(fd < 0) perror("open");

/** Zero out the aiocb structure (recommended) **/
bzero((char *)&my_aiocb, sizeof(struct aiocb));

/** Allocate ad data buffer for the aiocb request **/
my_aiocb.aio_buf = malloc(BUFSIZE + 1);

if(!my_aiocb.aio_buf) perror("malloc");

/** Initialize the neccessary fields in the aiocb **/
my_aiocb.aio_fildes = fd;
my_aiocb.aio_nbytes = BUFSIZE;
my_aiocb.aio_offset = 0;

ret = aio_read(&my_aiocb);
if(ret < 0) perror("aio_read");

while(aio_error(&my_aiocb) == EINPROGRESS);

if((ret = aio_return(&my_aiocb)) > 0) {
  /** got ret bytes on the read **/
} else {
  /** read failed, consult errno **/
}
```
  在上面代码中，在打开要从中读取数据的文件之后，我们就清空了aiocb结构，然后分配一个数据缓冲区。并将对这个数据缓冲区的引用放到aio_buf中，然后，我们将aio_nbytes初始化成缓冲区的大小。并将aio_offset设置成0(该文件中的第一个偏移量)。我们将aio_fildes设置为从中读取数据的文件描述符。在设置这些域之后，就调用aio_read请求进行读操作。然后我们可以调用aio_error来确定aio_read的状态。只要状态是EINPROGRESS, 就一直忙碌等待，直到状态发生变化为止。现在，请求可能成功，可能失败。
  
  注意使用这个API与标准的函数从文件读取内容是非常相似的。除了aio_read的一些异步特性之外，另外一个区别是读操作偏移量的设置。在传统的read调用中，偏移量是在文件描述符上下文件中进行维护的。对于每隔读操作来说，偏移量都需要进行更新，这样后续的操作此阿能对下一块数据进行殉职。对于异步I/O操作来说这是不可能的，因为我们可以同时执行很多读请求，因此必须为每个特定的读请求都指定偏移量。
  
#### aio_error
  aio_error函数被用来确定请求的状态。其原型如下:`int aio_error(struct aiocb *aiocbp)`;
  这个函数可以返回以下内容:
  * EINPROGRESS: 说明请求尚未完成
  * ECANCELLED: 说明请求被应用程序取消了
  * -1: 说明发生了错误，具体错误原因可查阅errno
  
#### aio_return
  异步I/O和标准块I/O之间的另外一个区别是我们不能立即访问这个函数的返回状态，因为我们没有阻塞在read调用上。在标准的read调用中，返回状态是在该函数返回时提供的。但是在异步I/O中，我们要使用aio_return函数。这个函数的原型如下:`ssize_t aio_return(struct aiocb *aiocb);`

  只有在aio_error调用确定请求已经完成(可能完成，也可能发生了错误)之后，才会调用这个函数。aio_return的返回值就等价于同步情况read和write系统调用的返回值(所传输的字节数，如果发生错误，返回值就为-1).
  
#### aio_write
  aio_write函数用来请求一个异步写操作。其函数原型如下:`int aio_write(struct aiocb *aiocb);`
  aio_write函数会立即返回，说明请求已经进行排队(成功返回值为0，失败时返回值为-1, 并相应的设置errno).
  
  这与read系统调用类似，但是有一点不一样的行为需要注意。回想一下对于read调用来说，要使用的偏移量时非常重要的。然而，对于write来说，这个偏移量只有在没有设置O_APPEND选项的文件上下文中才会非常重要。如果设置了O_APPEND, 那么这个偏移量就会被忽略，数据都会被附加到文件的末尾。否则，aio_offset域就确定了数据要写入的文件中的偏移量。
  
#### aio_suspend
  我们可以执行aio_suspend函数来挂起(或阻塞)调用进程，直到异步请求完成为止，此时会产生一个信号，或者发生其他超时操作。调用者提供了一个aiocb引用列表，其中任何一个完成都会导致aio_suspend返回。aio_suspend的函数原型如下:`int aio_suspend(const struct aiocb *const cblist[], int n, const struct timespec *timeout)`
  aio_suspend的使用非常简单。我们要提供一个aiocb引用列表。如果任何一个完成了，这个调用就会返回0。否则就会返回-1, 说明发生了错误。
```
struct aiocb *cblist[MAX_LIST]

/** clear the list. **/
bzero((char *)cblist, sizeof(cblist));

/** load one or more references into the list **/
cblist[0] = &my_aiocb;

ret = aio_read(&my_aiocb);

ret = aio_suspend(cblist, MAX_LIST, NULL);
```
  注意，aio_suspend的第二个参数是cblist中元素的个数，而不是aiocb引用的个数。cblist中任何NULL元素都会被aio_suspend忽略。
  
  如果为aio_suspend提供了超时，而超时情况的确发生了，那么就会返回-1,errno中会包含EAGAIN.
  
#### aio_cancel
  aio_cancel函数允许我们取消对某个文件描述符执行的一个或所有I/O请求。其原型为`int aio_cancel(int fd, struct aiocb *aiocb);`
  
  要取消一个请求，我们需要提供文件描述符和aiocb引用。如果这个请求被成功取消了，那么这个函数就会返回AIO_CANCELED. 如果请求完成了，这个函数就会返回AIO_NOTCANCELED.
  
  要取消对某个给定文件描述符的所有请求，我们需要提供这个文件的描述符，以及一个对aiocb的NULL引用。如果所有的请求都取消了，这个函数就会返回AIO_CANCELED; 至少一个请求没有被取消，那么这个函数就会返回AIO_NOT_CANCELED; 如果没有一个请求可以被取消，那么这个函数就会返回AIO_ALLDONE. 我们然后可以用aio_error来验证每个aio请求。如果这个请求已经被取消了，那么aio_error就会返回-1,并且errno会被设置为ECANCELED.
  
#### lio_listio
  最后，AIO提供了一种方法使用lio_listio API函数同时发起多个传输。这个函数非常重要，因为这意味着我们可以在一个系统调用(一次内核上下文切换)中启动大量的I/O操作。从性能的角度来看，这非常重要，因此值得我们花点时间探索一下。lio_listio API函数的原型为:`int lio_listio(int mode, struct aiocb *list[], int nent, struct sigevent *sig);`
  mode参数可以是LIO_WAIT或LIO_NOWAIT. LIO_WAIT会阻塞这个调用，直到所有I/O都完成为止。在操作进行排队之后， LIO_NOWAIT就会返回。list是一个aiocb引用的列表，最大元素个数是由nent定义的。注意list的元素可以为NULL, lio_listio会将其忽略。sigevent引用定义了在所有I/O操作都完成时产生信号的方法。
  
  对于lio_listio的请求与传统的read或write请求在必须指定的操作方面稍有不同，如:
```
struct aiocb aiocb1, aiocb2;
struct aiocb *list[MAX_LIST];

...

/** Prepare the first aiocb **/
aiocb1.aio_fildes = fd;
aiocb1.aio_buf = malloc(BUFSIZE + 1);
aiocb1.aio_nbytes = BUFSIZE;
aiocb1.aio_lio_opcode = LIO_READ;

...

bzero((char *)list, sizeof(list));
list[0] = &aiocb1;
list[1] = &aiocb2;

ret = lio_listio(LIO_WAIT, list, MAX_LIST, NULL);
```
  对于读来说，aio_lio_opcode域的值为LIO_READ. 对于写操作来说，我们要使用LIO_WRITE, 不过LIO_NOP对于不执行操作来说也是有效的。
  
### AIO通知
  现在我们已经看过了可用的AIO函数，本节将深入介绍对异步通知可以使用的方法。我们将通过信号和函数回调来探索异步函数的通知机制。
  
#### 使用信号进行异步通知
  使用信号进行进程间通信(IPC)是UNIX中的一种传统机制，AIO也可以支持这种机制。在这种范例中，应用程序需要定义信号处理程序，在产生指定的信号时就会调用这个处理程序。应用程序然后配置一个异步请求将在请求完成时产生一个信号。作为信号上下文的一部分，特定的aiocb请求被提供用来记录多个可能会出现的请求。下面展示了这种通知方法:
```
void setup_io(...)
{
  int fd;
  struct sigaction sig_act;
  struct aiocb my_aiocb;
  
  ...
  
  /** setup the signal handler **/
  sigemptyset(&sig_act.sa_mask);
  sig_act.sa_flags = SA_SIGINFO;
  sig_act.sa_sigaction = aio_completion_handler;
  
  /** setup the AIO request **/
  bzero((char *)&my_aiocb, sizeof(struct aiocb));
  my_aiocb.aio_fildes = fd;
  my_aiocb.aio_buf = malloc(BUFSIZE + 1);
  my_aiocb.aio_nbytes = BUFSIZE;
  my_aiocb.aio_offset = next_offset;
  
  /** Link the AIO request with the Signal handler **/
  my_aiocb.aio_sigevent.sigev_notify = SIGEV_SIGNAL;
  my_aiocb.aio_sigevent.sigev_signo = SIGIO;
  my_aiocb.aio_sigevent.sigev_value.sival_ptr = &my_aiocb;
  
  /** Map the Signal to the Signal Handler **/
  ret = sigaction(SIGIO, &sig_act, NULL);
  
  ...
  
  ret = aio_read(&my_aiocb);
}

viod aio_completion_handler(int signo, siginfo_t *info, void *context)
{
  struct aiocb *req;
  
  /** Ensure it's our signal **/
  if(info->si_signo == SIGIO) {
    req = (struct aiocb *)info->si_value.sival_prt;
    
    /** Did the request complete ? **/
    if(aio_error(req) == 0) {
      /** Request completed successfully, get the return status **/
      ret = aio_return(req);
    }
  }
  
  return;
}
```
  上面代码中，我们在aio_completion_handler函数中设置信号处理程序来捕获SIGIO信号。然后初始化aio_sigevent结构产生SIGIO信号来进行通知(这是通过sigev_notify中的SIGEV_SIGNAL定义来指定的)。当读操作完成时，信号处理程序就从该信号的si_value结构中提取处aiocb,并检查错误状态和返回状态来确定I/O操作是否完成。
  
  对于性能来说，这个处理程序也是通过请求下一次异步传输而继续进行i/O操作的理想地方。采用这种方式，在一次数据传输完成时，我们就可以立即开始下一次数据传输操作。
  
#### 使用回调函数进行异步通知
  另一种通知方式是系统回调函数。这种机制不会为通知而产生一个信号，而是会调用用户空间的一个函数来实现通知功能。我们在sigevent结构中设置了对aiocb的引用，从而可以唯一标识正在完成的特定请求。
```
void setup_io(...)
{
  int fd;
  struct aiocb my_aiocb;
  
  ...
  
  /** setup the signal handler **/
  bzero((char *)&my_aiocb, sizeof(struct aiocb));
  my_aiocb.aio_fildes = fd;
  my_aiocb.aio_buf = malloc(BUFSIZE + 1);
  my_aiocb.aio_nbytes = BUFSIZE;
  my_aiocb.aio_offset = next_offset;
  
  /** Link the AIO request with a thread callback **/
  my_aiocb.aio_sigevent.sigev_notify = SIGEV_THREAD;
  my_aiocb.aio_sigevent.notify_function = aio_completion_handler;
  my_aiocb.aio_sigevent.notify_attributes = NULL;
  my_aiocb.aio_sigevent.sigev_value.sival_ptr = &my_aiocb;
  
  /** Map the Signal to the Signal Handler **/
  ret = sigaction(SIGIO, &sig_act, NULL);
  
  ...
  
  ret = aio_read(&my_aiocb);
}

viod aio_completion_handler(sigval_t sigval)
{
  struct aiocb *req;
  
  req = (struct aiocb *)sigval.sival_ptr;
  
  /** Did the request complete ? **/
  if(aio_error(req) == 0) {
    /** Request completed successfully, get the return status **/
    ret = aio_return(req);
  }
  
  return;
}
```
  上面代码中，在创建自己的aiocb请求之后，我们使用SIGEV_THREAD请求了一个线程回调函数来作为通知方法。然后我们将指定特定的通知程序，并将要传输的上下文家在到处理程序中(在这种情况中，是个对aiocb请求自己的引用)。 在这个处理程序中，我们简单地引用到达的sigval指针并使用AIO函数来验证请求已经完成。
  
### 对 AIO 进行系统优化
  proc 文件系统包含了两个虚拟文件，它们可以用来对异步 I/O 的性能进行优化：
  * /proc/sys/fs/aio-nr 文件提供了系统范围异步 I/O 请求现在的数目。
  * /proc/sys/fs/aio-max-nr 文件是所允许的并发请求的最大个数。最大个数通常是 64KB，这对于大部分应用程序来说都已经足够了。

### 结束语
  使用异步 I/O 可以帮助我们构建 I/O 速度更快、效率更高的应用程序。如果我们的应用程序可以对处理和 I/O 操作重叠进行，那么AIO就可以帮助我们构建可以更高效地使用可用 CPU 资源的应用程序。尽管这种 I/O 模型与在大部分Linux应用程序中使用的传统阻塞模式都不同，但是异步通知模型在概念上来说却非常简单，可以简化我们的设计。
  
#### 参考连接
https://www.ibm.com/developerworks/cn/linux/l-async/
  
