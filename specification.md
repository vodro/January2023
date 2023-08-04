#<div style='text-align: center'>January 2023 CSE 314  </div>
###<div style='text-align: center'>Offline Assignment 5: xv6 - Threading & Synchronization</div>

#### Introduction
In our previous offline, we have used **POSIX** threads to solve _synchronization problems_. As a smart CSE student, we won't be limited to a user only, but also design thread and synchronization primitives of our own. In this offline, we will add support of threads in **xv6**. We will implement a user level thread library and some system calls related to threads that very familiar to us. Then, we will implement **POSIX-like** synchronization primitives in **xv6**. Finally, we will use these primitives to solve some synchronization problems.

#### Background
* Revisit the difference between _process_ and _thread_. The main difference is that threads share the same address space, while processes have their own address space. 

* Thoroughly understand what threads are and how you used them on offline on **IPC**.
* Key take-away idea for threads: threads are very much _like processes_ (they can run in parallel on different physcal CPUs), but they share the same address space (the address space of the process that created them). 

* Though the threads share common address space, each thread requires its own stack. This is because each thread might execute entirely different code in the program (call different functions with different arguments; all this information has to be preserved for each thread individually) 


###Task 1: Implementing thread support in xv6
You need to write three system calls 
* `thread_create()`
* `thread_join()`
* `thread_exit()`

Our first system call `thread_create()` would have  the following signature:
```c
int thread_create(void(*fcn)(void*), void *arg, void*stack)
```
This call creates a new kernel thread which shares the address space with the calling process.  

The new thread creation will be similar to new process creation. Actually we will create a new process with the same address space as the calling process. The only difference is that we will create a new stack for the new process. In our implementation we will copy file descriptors in the same manner fork() does it. The new process uses `stack`` as its user `tack, which is passed the given argument `arg` and uses a fake return PC (`0xffffffff`). The stack should be **strictly one page** in size. The new thread starts executing at the address specified by `fcn`.


The other new system call is: 
```c 
int thread_join(void)
``` 
This call waits for a child thread that shares the address space with the calling process. It returns the PID of waited-for child or -1 if none.

The last system call is: 
```c
int thread_exit(void)
```

You also need to think about the semantics of a couple of existing system calls. For example, `int wait()` should wait for a child process that does not share the address space with this process. It should also free the address space if this is last reference to it. Finally, `exit() `should work as before but for both processes and threads; little change is required here. 

To test your implementation you may use the following program:
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

struct balance {
    char name[32];
    int amount;
};

volatile int total_balance = 0;

volatile unsigned int delay (unsigned int d) {
   unsigned int i; 
   for (i = 0; i < d; i++) {
       __asm volatile( "nop" ::: );
   }

   return i;   
}

void do_work(void *arg){
    int i; 
    int old;
   
    struct balance *b = (struct balance*) arg; 
    printf(1, "Starting do_work: s:%s\n", b->name);

    for (i = 0; i < b->amount; i++) { 
        // lock and mlock will be implemented by you.
         // thread_spin_lock(&lock);
         // thread_mutex_lock(&mlock);
         old = total_balance;
         delay(100000);
         total_balance = old + 1;
         //thread_spin_unlock(&lock);
         // thread_mutex_lock(&mlock);

    }
  
    printf(1, "Done s:%x\n", b->name);

    thread_exit();
    return;
}

int main(int argc, char *argv[]) {

  struct balance b1 = {"b1", 3200};
  struct balance b2 = {"b2", 2800};
 
  void *s1, *s2;
  int thread1, thread2, r1, r2;

  s1 = malloc(4096);
  s2 = malloc(4096);

  thread1 = thread_create(do_work, (void*)&b1, s1);
  thread2 = thread_create(do_work, (void*)&b2, s2); 

  r1 = thread_join();
  r2 = thread_join();
  
  printf(1, "Threads finished: (%d):%d, (%d):%d, shared balance:%d\n", 
      thread1, r1, thread2, r2, total_balance);

  exit();
}
```
Here we create two threads that execute the same `do_work()` function concurrently. The do_work() function in both threads updates the shared variable total_balance.


#### Hints
The `thread_create()` call should behave very much like fork, except that instead of copying the address space to a new page directory, clone initializes the new process so that the new process and cloned process use the same page directory. Thus, memory will be shared, and the two "processes" are really actually threads.

The `int thread_join(void)` system call is very similar to the already existing int wait(void) system call in xv6. Join waits for a thread child to finish, and wait waits for a process child to finish.

Finally, the `thread_exit()` system call is very similar to exit(). You should however be careful and do not deallocate the page table of the entire process when one of the threads exits.

#### Task 2: Implementing synchronization primitives in xv6
If you implemented your threads correctly and ran them a couple of times you might notice that the total balance (the final value of the total_balance does not match the expected `6000`, i.e., the sum of individual balances of each thread. This is because it might happen that both threads read an old value of the total_balance at the same time, and then update it at almost the same time as well. As a result the deposit (the increment of the balance) from one of the threads is lost.

##### Spinlock
To fix this synchronization error you have to implement a spinlock that will allow you to execute the update atomically, i.e., you will have to implement the `thread_spin_lock()` and `thread_spin_unlock()` functions and put them around your atomic section (you can uncomment existing lines above).


Specifically you should define a simple lock data structure and implement three functions that:
 1. initialize the lock to the correct initial state (`void thread_spin_init(struct thread_spinlock *lk)`)
 2. a funtion to acquire a lock (`void thread_spin_lock(struct thread_spinlock *lk)`) 
 3.  a function to release it `void thread_spin_unlock(struct thread_spinlock *lk)`

To implement spinlocks you can copy the implementation from the xv6 kernel. Just copy them into your program (`threads.c` and make sure you understand how the code works).


#### Mutexes 
Suppose, you are running on a system with a **single** physical CPU, or the system is under high load and a context switch occurs in a *critical section*  then all threads of the process start to spin endlessly, waiting for the interrupted (lock-holding) thread to be secheduled and run again the spinlocks become inefficient. If you look closely, the main culprit is that the threads are spinning in a loop, wasting CPU cycles. Mutexes to our rescue!

A higher level pseudo-code for a mutex is as follows:
```c
void thread_mutex_lock(struct thread_mutex *m)
{
  while(locked(m))
    yield();
}

void
thread_mutex_unlock(struct thread_mutex *m)
{
  unlock(m);
}
```

Based on the high-level description of the mutex above, implement a mutex that will allow you to execute the update atomically similar to spinlock, but instead of spinning will release the CPU to another thread. Test your implementation by replacing spinlocks in your example above with mutexes.


Specifically you should define a simple mutex data structure and implement three functions that: 
1. initialize the mutex to the correct initial state (`void thread_mutex_init(struct thread_mutex *m)`)
2. a funtion to acquire a mutex (`void thread_mutex_lock(struct thread_mutex *m)`)
3. a function to release it `void thread_mutex_unlock(struct thread_mutex *m)`.

Mutexes can be implemented very similarly to spinlocks (the implementation you already have). Since xv6 doesn't have an explicit `yield(0)` system call, you can use `sleep(1)` instead from `thread_mutex_lock` function.



### Conditional Variables
While spinlock and mutex synchronization work well, sometimes we need a synchronization pattern similar to the producer-consumer queue we've discussed in class, i.e., instead of spinning on a spinlock or yielding the CPU in a mutex, we would like the thread to sleep until certain contidion is met.

POSIX provides support for such scheme with conditional variables. A condition variable is used to allow a thread to sleep until a condition is true. Note that conditional variables are always used along with the mutex.

You have to implement conditional variables similar to the once provided by POSIX. The function primarily used for this is pthread_cond_wait(). It takes two arguments; the first is a pointer to a condition variable, and the second is a locked mutex. When invoked, pthread_cond_wait() unlocks the mutex and then pauses execution of its thread. It will now remain paused until such time as some other thread wakes it up. These operations are "atomic;" they always happen together, without any other threads executing in between them.

```c
struct q {
   struct thread_cond cv;
   struct thread_mutex m;
 
   void *ptr;
};

// Initialize
thread_cond_init(&q->cv);
thread_mutex_init(&q->m);

// Thread 1 (sender)
void*
send(struct q *q, void *p)
{
   thread_mutex_lock(&q->m);
   while(q->ptr != 0)
      ;
   q->ptr = p;
   thread_cond_signal(&q->cv);
   thread_mutex_unlock(&q->m);
}

// Thread 2 (receiver)

void*
recv(struct q *q)
{
  void *p;

  thread_mutex_lock(&q->m);

  while((p = q->ptr) == 0)
    pthread_cond_wait(&q->cv, &q->m);
  q->ptr = 0;

  thread_mutex_unlock(&q->m);
  return p;
}
```
Develop a problem for testing. 

#### Semaphores
Conditional variables can be used to implement semaphores. 
Follow [this](https://github.com/angrave/SystemProgramming/wiki/Synchronization,-Part-5:-Condition-Variables) to implement semaphores using conditional variables.



A sample program to test your semaphore implementation is given below:
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

struct queue{
	int arr[16];
	int front;
	int rear;
	int size;
	queue()
	{
		front = 0;
		rear = 0;
		size = 0;
	}
	void push(int x)
	{
		arr[rear] = x;
		rear = (rear+1)%16;
		size++;
	}
	int front()
	{
		if(size==0)
			return -1;
		return arr[front];
	}
	void pop()
	{
		front = (front+1)%16;
		size--;
	}
};
struct queue q;
// a mutex object lock 
// a semaphore object empty
// a semaphore object full

void init_semaphore()
{
	// initialize mutex lock
	// initialize semaphore empty with 5
	// initialize semaphore full with 0

}

void * ProducerFunc(void * arg)
{	
	printf("%s\n",(char*)arg);
	int i;
	for(i=1;i<=10;i++)
	{
		// wait for semphore empty

		// wait for mutex lock
		
		sleep(1);	
		q.push(i);
		printf("producer produced item %d\n",i);
		
		// unlock mutex lock	
		// post semaphore full
	}
}

void * ConsumerFunc(void * arg)
{
	printf("%s\n",(char*)arg);
	int i;
	for(i=1;i<=10;i++)
	{	
		// wait for semphore full
		// wait for mutex lock
 		
			
		sleep(1);
		int item = q.front();
		q.pop();
		printf("consumer consumed item %d\n",item);	


		// unlock mutex lock
		// post semaphore empty		
	}
}

int main(void)
{	
	
	init_semaphore();
	
	char * message1 = "i am producer";
	char * message2 = "i am consumer";


	void *s1, *s2;
  	int thread1, thread2, r1, r2;

  	s1 = malloc(4096);
  	s2 = malloc(4096);

  	thread1 = thread_create(ProducerFunc, (void*)message1, s1);
  	thread2 = thread_create(ConsumerFunc, (void*)message2, s2); 

  	r1 = thread_join();
  	r2 = thread_join();	
	
	exit();
}

```
Make necessary changes to make it running. 


<!-- [malloc vs sbrk](https://stackoverflow.com/questions/19676688/how-malloc-and-sbrk-works-in-unix) -->