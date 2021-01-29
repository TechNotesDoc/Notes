

参见`libuv/src/threadpool.c`文件

```c
struct uv__work {
  void (*work)(struct uv__work *w);
  void (*done)(struct uv__work *w, int status);
  struct uv_loop_s* loop;
  void* wq[2];
};
```

   `uv__work`就代表一个task，可以看到里面有两个函数指针（work代表任务实际操作，done用于对任务进行状态确认）。`wq`成员就是一个QUEUE的节点， `uv__work`就是通过`wq`与其他 `uv__work`连接成一个队列。

   下面来看一下`threadpool`的初始化，代码如下：

```c
#include "uv-common.h"
#include <stdlib.h>
#define MAX_THREADPOOL_SIZE 1024

static uv_once_t once = UV_ONCE_INIT;
static uv_cond_t cond;
static uv_mutex_t mutex;
static unsigned int idle_threads;
static unsigned int slow_io_work_running;
static unsigned int nthreads;
static uv_thread_t* threads;
static uv_thread_t default_threads[4];
static QUEUE exit_message;
static QUEUE wq;
static QUEUE run_slow_work_message;
static QUEUE slow_io_pending_wq;


static void init_threads(void) {
  unsigned int i;
  const char* val;
  uv_sem_t sem;
  
  /*
   * 1.线程池中的线程数，默认值为4
   */
  nthreads = ARRAY_SIZE(default_threads);
   /*
   * 2.从环境变量中得到UV_THREADPOOL_SIZE值,如果有的话，就更新nthreads值为环境变量的值
   */
  val = getenv("UV_THREADPOOL_SIZE");
  if (val != NULL)
    nthreads = atoi(val);
   
  /*
  *  3.线程数量nthreads范围检测 (1-MAX_THREADPOOL_SIZE)
  */
  if (nthreads == 0)
    nthreads = 1;
  if (nthreads > MAX_THREADPOOL_SIZE)
    nthreads = MAX_THREADPOOL_SIZE;

 /*
  * 4. 更新线程指针threads，如果nthreads大于默认的4个线程，就重新申请nthreads个内存
  */
  threads = default_threads;
  if (nthreads > ARRAY_SIZE(default_threads)) {
    threads = uv__malloc(nthreads * sizeof(threads[0]));
    if (threads == NULL) {
      nthreads = ARRAY_SIZE(default_threads);
      threads = default_threads;
    }
  }
  /*
  * 5. 初始化条件变量
  */
  if (uv_cond_init(&cond))
    abort();

  /*
  * 6. 初始互斥锁
  */
  if (uv_mutex_init(&mutex))
    abort();

    
  /*
  * 7. 初始化线程池队列头wq 这是个全局变量，还有slow_io_pending_wq，run_slow_work_message
  */
  QUEUE_INIT(&wq);
  QUEUE_INIT(&slow_io_pending_wq);
  QUEUE_INIT(&run_slow_work_message);

 /*
  * 6. 初始话一个局部的信号量sem
  */
  if (uv_sem_init(&sem, 0))
    abort();

  /*
  * 6. 创建nthreads个线程，且回调函数都是worker
  */
  for (i = 0; i < nthreads; i++)
    if (uv_thread_create(threads + i, worker, &sem))
      abort();

  /*
  * 7.等待信号量释放，如果释放掉说明，上面uv_thread_create成功
  */
  for (i = 0; i < nthreads; i++)
    uv_sem_wait(&sem);

  uv_sem_destroy(&sem);
}
```

​    上面的代码中，一共创建了`nthreads`个线程，那么每个线程的执行代码是什么呢？由线程创建代码：`uv_thread_create(threads + i, worker, &sem)`，可以看到，每一个线程都是执行worker函数，下面看看worker函数都在做什么：

```c
static void worker(void* arg) {
  struct uv__work* w;
  QUEUE* q;
  int is_slow_work;

  uv_sem_post((uv_sem_t*) arg);
  arg = NULL;
  /*
   *  因为是多线程，所以需要互斥锁 ，这里加锁
   */
  uv_mutex_lock(&mutex);
  for (;;) {
    /* `mutex` should always be locked at this point. */

    /* Keep waiting while either no work is present or only slow I/O
     *   and we're at the threshold for that. 
     *  如果任务队列是空的，或者  （slow I/O 且在临界点）
     *
     */
    while (QUEUE_EMPTY(&wq) ||
           (QUEUE_HEAD(&wq) == &run_slow_work_message &&
            QUEUE_NEXT(&run_slow_work_message) == &wq &&
            slow_io_work_running >= slow_work_thread_threshold())) {
      idle_threads += 1; 			// 空闲线程数加1
      uv_cond_wait(&cond, &mutex);	// 一直在等待条件变量
      idle_threads -= 1;			// 被唤醒之后，说明有任务被post到队列，因此空闲线程数需要减1
    }
    /*
     *  取出第一个task
     */
    q = QUEUE_HEAD(&wq);
    if (q == &exit_message) {
      uv_cond_signal(&cond);
      uv_mutex_unlock(&mutex);
      break;
    }

     // 从队列中移除这个task
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);  /* Signal uv_cancel() that the work req is executing. */

    is_slow_work = 0;
    if (q == &run_slow_work_message) {
      /* If we're at the slow I/O threshold, re-schedule until after all
         other work in the queue is done. */
      if (slow_io_work_running >= slow_work_thread_threshold()) {
        QUEUE_INSERT_TAIL(&wq, q);
        continue;
      }

      /* If we encountered a request to run slow I/O work but there is none
         to run, that means it's cancelled => Start over. */
      if (QUEUE_EMPTY(&slow_io_pending_wq))
        continue;

      is_slow_work = 1;
      slow_io_work_running++;

      q = QUEUE_HEAD(&slow_io_pending_wq);
      QUEUE_REMOVE(q);
      QUEUE_INIT(q);

      /* If there is more slow I/O work, schedule it to be run as well. */
      if (!QUEUE_EMPTY(&slow_io_pending_wq)) {
        QUEUE_INSERT_TAIL(&wq, &run_slow_work_message);
        if (idle_threads > 0)
          uv_cond_signal(&cond);
      }
    }
   /*
    *  解锁
    */
    uv_mutex_unlock(&mutex);

   /*
    *  根据节点q取出 uv__work 然后调用自己的回调函数
    */
    w = QUEUE_DATA(q, struct uv__work, wq);
    w->work(w);

    uv_mutex_lock(&w->loop->wq_mutex);
    w->work = NULL;  /* Signal uv_cancel() that the work req is done
                        executing. */
    QUEUE_INSERT_TAIL(&w->loop->wq, &w->wq);
    uv_async_send(&w->loop->wq_async);
    uv_mutex_unlock(&w->loop->wq_mutex);

    /* Lock `mutex` since that is expected at the start of the next
     * iteration. */
    uv_mutex_lock(&mutex);
    if (is_slow_work) {
      /* `slow_io_work_running` is protected by `mutex`. */
      slow_io_work_running--;
    }
  }
}
```

   

可以看到，多个线程都会在worker方法中等待在conn条件变量上，一旦有任务加入队列，线程就会被唤醒，然后只有一个线程会得到任务的执行权，其他的线程只能继续等待。

   那么如何向队列提交一个task呢？看以下代码：

```c
 1 void uv__work_submit(uv_loop_t* loop,
 2                      struct uv__work* w,
 3                      void (*work)(struct uv__work* w),
 4                      void (*done)(struct uv__work* w, int status)) {
 5   uv_once(&once, init_once);
 6   // 构造一个task
 7   w->loop = loop;
 8   w->work = work;
 9   w->done = done;
10   // 将其插入任务队列
11   post(&w->wq);
12 }
```

   接着看post做了什么：

```c
static void post(QUEUE* q, enum uv__work_kind kind) {
  // 同步队列操作
  uv_mutex_lock(&mutex);
    
  if (kind == UV__WORK_SLOW_IO) {
    /* Insert into a separate queue. */
    QUEUE_INSERT_TAIL(&slow_io_pending_wq, q);
    if (!QUEUE_EMPTY(&run_slow_work_message)) {
      /* Running slow I/O tasks is already scheduled => Nothing to do here.
         The worker that runs said other task will schedule this one as well. */
      uv_mutex_unlock(&mutex);
      return;
    }
    q = &run_slow_work_message;
  }
 // 将task插入队列尾部
  QUEUE_INSERT_TAIL(&wq, q);
  // 如果当前有空闲线程，就向条件变量发送信号
  if (idle_threads > 0)
    uv_cond_signal(&cond);
  uv_mutex_unlock(&mutex);
}

```

   有提交任务，就肯定会有取消一个任务的操作，是的，他就是uv__work_cancel，代码如下：



```c

static int uv__work_cancel(uv_loop_t* loop, uv_req_t* req, struct uv__work* w) {
  int cancelled;

  uv_mutex_lock(&mutex);
  uv_mutex_lock(&w->loop->wq_mutex);
// 只有当前队列不为空并且要取消的uv__work有效时才会继续执行
  cancelled = !QUEUE_EMPTY(&w->wq) && w->work != NULL;
  if (cancelled)
    QUEUE_REMOVE(&w->wq);;// 从队列中移除task

  uv_mutex_unlock(&w->loop->wq_mutex);
  uv_mutex_unlock(&mutex);

  if (!cancelled)
    return UV_EBUSY;
// 更新这个task的状态
  w->work = uv__cancelled;
  uv_mutex_lock(&loop->wq_mutex);
  QUEUE_INSERT_TAIL(&loop->wq, &w->wq);
  uv_async_send(&loop->wq_async);
  uv_mutex_unlock(&loop->wq_mutex);

  return 0;
}

```

   至此，一个线程池的组成以及实现原理都说完了，可以看到，`libuv`几乎是用了最少的代码完成了高效的线程池。