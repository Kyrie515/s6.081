## Lab multithreading

### Uthread: switching between threads

1. The difference between  *caller-saved* registers and *callee-saved* registers.

   https://stackoverflow.com/questions/9268586/what-are-callee-and-caller-saved-registers

   > **Caller-saved registers** (AKA **volatile** registers, or **call-clobbered**) are used to hold temporary quantities that need not be preserved across calls.

   For that reason, it is the caller's responsibility to push these registers onto the stack or copy them somewhere else *if* it wants to restore this value after a procedure call.

   It's normal to let a `call` destroy temporary values in these registers, though.

   > **Callee-saved registers** (AKA **non-volatile** registers, or **call-preserved**) are used to hold long-lived values that should be preserved across calls.

   When the caller makes a procedure call, it can expect that those registers will hold the same value after the callee returns, making it the responsibility of the callee to save them and restore them before returning to the caller. Or to not touch them.
   
1. Understand how the thread know where to execute the function we pass to it

   >*What I cannot create, I do not understand*
   >
   >​											--Richard Feynman

   When I am doing this section, there is a question in my mind: if I cannot modify the `pc` register directly, how can I change the execution sequence of the function? That is, when you use the function `thread_create(void* (func)())`, **after you executed the `thread_schedule()` function in `main()`, you will have the chosen thread to execute the creation function. **

   From above, we can find that in `thread_schedule()`, we need to help the thread to find the entry of the function that needed to be executed.

   ```C
   void 
   thread_schedule(void)
   {
     struct thread *t, *next_thread;
   
     /* Find another runnable thread. */
     next_thread = 0;
     t = current_thread + 1;
     for(int i = 0; i < MAX_THREAD; i++){
       if(t >= all_thread + MAX_THREAD)
         t = all_thread;
       if(t->state == RUNNABLE) {
         next_thread = t;
         break;
       }
       t = t + 1;
     }
   
     if (next_thread == 0) {
       printf("thread_schedule: no runnable threads\n");
       exit(-1);
     }
   
     if (current_thread != next_thread) {         /* switch threads?  */
       next_thread->state = RUNNING;
       t = current_thread;
       current_thread = next_thread;
       /* YOUR CODE HERE
        * Invoke thread_switch to switch from t to next_thread:
        * thread_switch(??, ??);
        */
       thread_switch((uint64)&t->context, (uint64)&next_thread->context);
     } else
       next_thread = 0;
   }
   ```
   
   From the function we can see that we keep the context of current thread and switch the CPU to next thread, then function returns, so after function returned, we can get the thread to run the execution function, that means, we can change the `ra` register into the creation function address to help the thread find the entry of the function. So here is the `thread_create()` implementation:
   
   ```C
   void 
   thread_create(void (*func)())
   {
     struct thread *t;
   
     for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
       if (t->state == FREE) break;
     }
     t->state = RUNNABLE;
     // YOUR CODE HERE
     /*
      * initialize the stack point at the top of the stack
      */
     uint64 sp = (uint64)&t->stack[STACK_SIZE];
     sp -= sp % 16; //Align the sp
     /*
      * wrong way to set ra:
      * *(uint64*)ra = (uint64) func
      * because this way you set the value in address ra to func address
      * but you should set the ra directly to the func address because it is called Return Address
      */
   
     t->context.ra = (uint64)func;
     t->context.sp = sp;
   
   }
   ```

### Using threads

NOTE: Cannot initialize the lock inside the function because this would make the lock local, then the thread will not compete for the lock. From this we can know that locks should also be shared between thread.

Using a bucket lock for each bucket could speed threads up.

```C
pthread_mutex_t bucket_lock[NBUCKET];
```

Then according to the index computed by the hash function (`key % NBUCKET`) to acquire and release the lock

```C
static 
void put(int key, int value)
{
  int i = key % NBUCKET;
  // is the key already present?
  struct entry *e = 0;
//  pthread_mutex_lock(&lock);
  pthread_mutex_lock(&bucket_lock[i]);
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
//  pthread_mutex_unlock(&lock);
  pthread_mutex_unlock(&bucket_lock[i]);
}
```

### Barrier

Barrier is a mechanism which threads will wait here until all the threads reach the barrier.

```C
static void 
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread += 1;
  if (bstate.nthread == nthread) {
    bstate.round += 1;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  } else {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

