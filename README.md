# **Starve-Free Readers-Writers Problem**

The readers-writers problems are a set of problems is computer science that deal with synchronisation of processes of two kinds - readers and writers - on the same shared data. It is easy to observe that reading while another process is writing, or writing while another process is writing may cause errors. On the other hand, multiple readers can read at a time without any error.

The first problem provides a solution which gives readers a priority, the second problem provides a solution which gives writers a priority. This may lead to starvation of writers and readers in the first and second problem respectively.

The third problem known as **Starve-Free Readers-Writers Problem** requires a solution which guarantees that no process may starve. A solution to this problem is described in detail below.

## **The Idea:**

The idea is to use semaphores that behave in FIFO manner, i.e. the process which started its wait first, will get released from the semaphore first. Also, we will ensure that once a writer is waiting, no newly arrived reader gets to access the resource, this ensures no starvation for writers. While the FIFO nature ensures no starvation for readers, thereby making this a feasible solution for the **Starve-Free Readers-Writers Problem**.

Below is the C++ style code for a queue, and a Semaphore based on it:

```C++
//node for queue
struct Node
{
    int data;           //will be process id in our case
    Node *next;         //pointer to next node in queue or NULL
    Node(int x)         //constructor
    {
        data = x;
        next = NULL;
    }
};

struct Queue
{
    Node *head = NULL , *tail = NULL;       //nodes enter through tail 
                                            //and leave through head
    void push(int x)
    {
        Node *temp = new Node(x);
        if(head == NULL)                    //if current node is first
        {
            head = tail = NULL;
            return;
        }
        else
        {
            tail->next = temp;
            tail = temp;
        }
    }
    int pop()
    {
        if(head == NULL)
        {
            return -1; 
        }
        else
        {
            int answer = head->data;
            head = head->next;
            if(head == NULL)            //if current node was the last
            {
                tail = NULL;
            }
            return answer;
        }
    }
};

struct Semaphore
{
    int available = 1;      
    Queue Q;

    void wait(int pid)
    {
        available--;
        if(available < 0)
        {
            Q.push(pid);
            // A function/system-call that can block a process given 
            // its process id
            block(pid);
        }
    }
    void signal()
    {
        available++;
        if(available < 1)
        {
            int which = Q.pop();
            // A function/system-call that can wake up a blocked process
            // given its process id
            wakeup(which);
        }
    }
};

```

Now, coming back to the readers-writers problem. We maintain three global variables:

1. ``readersIn``: integer variable, number of readers that have started reading.
2. ``readersOut``: integer variable, number of readers that have completed reading.
3. ``waiting``: boolean variable, whether a writer is waiting or not.

The first two variables are maintained to not allow writers to access data unless all readers have finished their reading. The last variable is maintained to not allow more readers to start reading while a writer (which came earlier than them) is waiting.

We use three semaphores:

1. ``inSemaphore``: manages the queue of all incoming processes, also allows manipulation/use of ``readersIn``.
2. ``outSemaphore``: allows manipulation/use of ``readersOut``
3. ``writerSemaphore``: contains at most one process at any time, a writer which is waiting for readers to finish their work.

Note that an exception is that we initialise ``writerSemaphore->available`` to 0 instead of 1. This is done because we do not want it to allow the process to continue to critical section EVEN IF no other process is waiting on it. ``writerSemaphore->signal()`` is triggered only by the last reader to finish reading, who entered before the waiting writer. 

Below are the explanations for implementation of reader and writer processes:

### **Readers:**

A reader process first waits on the ``inSemaphore``, then once woken up it updates ``readersIn``, releases ``inSemaphore`` and enters the critical section, i.e. starts reading. 


Once it has finished, it then waits on the ``outSemaphore`` and when woken up it updates ``readersOut``. At this point, it checks if ``readersIn`` and  ``readersOut`` are equal AND ``waiting`` is true. If both the conditions are met, it means all readers have finished reading and now the waiting writer may enter, so it triggers ``writerSemaphore->signal()``.

Now, it releases the ``outSemaphore``.

### **Writers:**

A writer process first waits on the ``inSemaphore``, followed by waiting on the ``outSemaphore``.

Now it checks whether all readers are finished doing their work and if yes, it simply releases the ``outSemaphore`` and enters its critical section.

However, if not all readers are done working, the writer releases ``outSemaphore`` but starts waiting on ``writerSemaphore`` and also sets ``waiting`` to ``true``. Once ``writerSemaphore->signal()`` is triggered, it then sets ``waiting`` to false and goes on to execute its critical section. 

Note that at one point, at most one writer is queued up on the ``writerSemaphore`` because the writer has not released ``inSemaphore`` yet, so this allows us to set ``waiting`` to ``false`` without worrying about further writers still being in queue as they will set it to ``true`` when they need it.

Note that writer needs to wait on the ``outSemaphore`` so that no reader process might change the value of ``readersIn`` or ``readersOut`` while the writer checks the above condition.

At the end, it releases the ``inSemaphore`` allowing further processes to proceed. It does not release it earlier because that would allow further writers to queue up on ``writerSemaphore`` thus destorying the validity of ``waiting`` (as explained above) and also it would allow further readers to read while it is writing.

Below is the C++ style pseudo-code for the above logic:

``` C++
Semaphore *inSemaphore = new Semaphore();
Semaphore *outSemaphore = new Semaphore();
Semaphore *writerSemaphore = new Semaphore();


//number of readers who have started reading and finished reading
//respectively
int readersIn = 0, readersOut = 0; 

//is a writer process waiting?
bool waiting = false;

void reader(int pid)
{
    //assuming we have access to process id of the reader in the passed
    //variable pid
    inSemaphore->wait(pid); //wait on inSemaphore
    readersIn++;
    inSemaphore->signal();  //release inSemaphore

    //
    // CRITICAL SECTION : READ DATA
    //

    outSemaphore->wait(pid);    //wait on outSemaphore
    readersOut++;

    //check if all readers are done
    if(readersIn == readersOut && waiting == true)
    {
        //this means the current reader was last to finish reading
        //now the waiting writer can proceed with writing
        writerSemaphore->signal();  
    }
    outSemaphore->signal();     //release outSemaphore
}

void writer(int pid)
{
    //assuming we have access to process id of the reader in the passed
    //variable pid
    inSemaphore->wait(pid);     //wait on inSemaphore
    outSemaphore->wait(pid);    //wait on outSemaphore
    if(readersIn == readersOut)
    {
        //no reader is currently working, proceed to critical section
        outSemaphore->signal();     //release outSemaphore
    }
    else
    {
        waiting = true;             //writer is now waiting
        outSemaphore->signal();     //release outSemaphore
        writerSemaphore->wait(pid); //wait on writerSemaphore
        waiting = false;            //no more writers waiting
    }

    //
    // CRITICAL SECTION : WRITE DATA
    //

    inSemaphore->signal();          //release inSemaphore
}

```


## References

- https://arxiv.org/abs/1309.4507




