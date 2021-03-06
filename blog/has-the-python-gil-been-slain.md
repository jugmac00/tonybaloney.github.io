blog_heading: Has the Python GIL been slain?
blog_subheading: Exploring new features in upcoming Python 3.8 and how they address the GIL.
blog_header_image: posts/gil_header.jpeg
blog_author: Anthony Shaw
blog_publish_date: May 15, 2019
---

In early 2003, Intel launched the new Pentium 4 “HT” processor. This processor was clocked at 3 GHz and had “Hyper-Threading” Technology.

<iframe width="560" height="315" src="https://www.youtube.com/embed/AmwzUrL3vMc?controls=0" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Over the following years, Intel and AMD battled to achieve the best desktop computer performance by increasing bus-speed, L2 cache size and reducing die size to minimize latency. The 3Ghz HT was superseded in 2004 by the “Prescott” model 580, which clocked up to 4 GHz.

It seemed like the path forward for better performance was higher clock speed, but CPUs were plagued by high power consumption and earth-warming heat output.

Do you have a 4Ghz CPU in your desktop? Unlikely, because the way forward for performance was higher-bus speed and multiple cores. The Intel Core 2 superseded the Pentium 4 in 2006, with clock speeds far lower.

Aside from the release of consumer multicore CPUs, something else happened in 2006, Python 2.5 was released! Python 2.5 came bundled with a beta version of the with statement that you know and love.
Python 2.5 had one major limitation when it came to utilizing Intel’s Core 2 or AMD’s Athlon X2.

-- The GIL.

## What is the GIL?

The GIL, or __Global Interpreter Lock__, is a boolean value in the Python interpreter, protected by a mutex. The lock is used by the core bytecode evaluation loop in CPython to set which thread is currently executing statements.

CPython supports multiple threads within a single interpreter, but threads must request access to the GIL in order to execute Opcodes (low-level operations). This, in turn, means that Python developers can utilize async code, multi-threaded code and never have to worry about acquiring locks on any variables or having processes crash from deadlocks.
The GIL makes multithreaded programming in Python simple.

![](/img/posts/breakdance.gif){: .img-responsive .center-block}

The GIL also means that whilst CPython can be multi-threaded, only 1 thread can be executing at any given time. This means that your quad-core CPU is doing this — (minus the bluescreen, hopefully)

The current version of the GIL was written in 2009, to support async features and has survived relatively untouched even after many attempts to remove it or reduce the requirement for it.

The requirement for any proposal to remove the GIL is that it should not degrade the performance of any single-threaded code. Anyone who ever enabled Hyper-Threading back in 2003 will appreciate why that is important.

## Avoiding the GIL in CPython

If you want truly concurrent code in CPython, you have to use multiple processes.
In CPython 2.6 the multiprocessing module was added to the standard library. Multiprocessing was a wrapper around the spawning of CPython processes (each with its own GIL) —

```python
from multiprocessing import Process

def f(name):
    print 'hello', name

if __name__ == '__main__':
    p = Process(target=f, args=('bob',))
    p.start()
    p.join()
```

Processes can be spawned, sent commands via compiled Python modules or functions and then rejoined into the master process.

Multiprocessing also supports sharing of variables via a Queue or a Pipe. It also has a Lock object, for locking objects in the master process for writing from other processes.
The multiprocessing has 1 major flaw. It has significant overhead, both in time and in memory usage. CPython startup times, even without no-site, are 100–200ms (see https://hackernoon.com/which-is-the-fastest-version-of-python-2ae7c61a6b2b).
So you can have concurrent code in CPython, but you have to carefully plan it’s application for long-running processes that have little sharing of objects between them.
Another alternative is a third party package like Twisted.

## PEP554 and the death of the GIL?

So to recap, multithreading in CPython is easy, but it’s not truly concurrent, and multiprocessing is concurrent but has a significant overhead.
What if there was a better way?
The clue in bypassing the GIL is in the name, the global interpreter lock is part of the global interpreter state. CPython processes can have multiple interpreters, and hence multiple locks, however, this feature is rarely used because it is only exposed via the C-API.
One of the features proposed for CPython 3.8 is PEP554, the implementation of sub-interpreters and an API with a new interpreters module in the standard library.
This enables creating multiple interpreters, from Python within a single process. Another change for Python 3.8 is that interpreters will all have individual GILs —

![](/img/posts/multiprocessing.png){: .img-responsive .center-block}

Because Interpreter state contains the memory allocation arena, a collection of all pointers to Python objects (local and global), sub-interpreters in PEP 554 cannot access the global variables of other interpreters.
Similar to multiprocessing, the way to share objects between interpreters would be to serialize them and use a form of IPC (network, disk or shared memory). There are many ways to serialize objects in Python, there’s the marshal module, the pickle module and more standardized methods like json and simplexml. Each of these has pro’s and con’s, all of them have an overhead.
First prize would be to have a shared memory space that is mutable and controlled by the owning process. That way, objects could be sent from a master-interpreter and received by other interpreters. This would be a lookup managed-memory space of PyObject pointers that could be accessed by each interpreter, with the main process controlling the locks.

![](/img/posts/multiinterpreters.png){: .img-responsive .center-block}

The API for this is still being worked out, but it will probably look like this:

```python
import _xxsubinterpreters as interpreters
import threading
import textwrap as tw
import marshal

# Create a sub-interpreter
interpid = interpreters.create()

# If you had a function that generated some data
arry = list(range(0,100))

# Create a channel
channel_id = interpreters.channel_create()

# Pre-populate the interpreter with a module
interpreters.run_string(interpid, "import marshal; import _xxsubinterpreters as interpreters")

# Define a
def run(interpid, channel_id):
    interpreters.run_string(interpid,
                            tw.dedent("""
        arry_raw = interpreters.channel_recv(channel_id)
        arry = marshal.loads(arry_raw)
        result = [1,2,3,4,5] # where you would do some calculating
        result_raw = marshal.dumps(result)
        interpreters.channel_send(channel_id, result_raw)
        """),
               shared=dict(
                   channel_id=channel_id
               ),
               )

inp = marshal.dumps(arry)
interpreters.channel_send(channel_id, inp)

# Run inside a thread
t = threading.Thread(target=run, args=(interpid, channel_id))
t.start()

# Sub interpreter will process. Feel free to do anything else now.
output = interpreters.channel_recv(channel_id)
interpreters.channel_release(channel_id)
output_arry = marshal.loads(output)

print(output_arry)
```

This example uses numpy and sends a numpy array over a channel by serializing it with the marshal module, the sub-interpreter then processes the data (on a separate GIL) so this could be a CPU-bound concurrency problem perfect for sub-interpreters.

### That looks inefficient

The marshal module is fairly fast, but not as fast as sharing objects directly from memory.
PEP 574 proposes a new pickle protocol (v5) which has support for allowing memory buffers to be handled separately from the rest of the pickle stream. For large data objects, serializing them all in one go and deserializing from the sub-interpreter would add a lot of overhead.
The new API could be interfaced (hypothetically, neither have been merged yet) like this —

```python
import _xxsubinterpreters as interpreters
import threading
import textwrap as tw
import pickle

# Create a sub-interpreter
interpid = interpreters.create()

# If you had a function that generated a numpy array
arry = [5,4,3,2,1]

# Create a channel
channel_id = interpreters.channel_create()

# Pre-populate the interpreter with a module
interpreters.run_string(interpid, "import pickle; import _xxsubinterpreters as interpreters")

buffers=[]

# Define a
def run(interpid, channel_id):
    interpreters.run_string(interpid,
                            tw.dedent("""
        arry_raw = interpreters.channel_recv(channel_id)
        arry = pickle.loads(arry_raw)
        print(f"Got: {arry}")
        result = arry[::-1]
        result_raw = pickle.dumps(result, protocol=5)
        interpreters.channel_send(channel_id, result_raw)
        """),
                            shared=dict(
                                channel_id=channel_id,
                            ),
                            )

input = pickle.dumps(arry, protocol=5, buffer_callback=buffers.append)
interpreters.channel_send(channel_id, input)

# Run inside a thread
t = threading.Thread(target=run, args=(interpid, channel_id))
t.start()

# Sub interpreter will process. Feel free to do anything else now.
output = interpreters.channel_recv(channel_id)
interpreters.channel_release(channel_id)
output_arry = pickle.loads(output)

print(f"Got back: {output_arry}")
```

## That sure looks like a lot of boilerplate

Ok, so this example is using the low-level sub-interpreters API. If you’ve used the multiprocessing library you’ll recognize some of the problems. It’s not as simple as threading , you can’t just say run this function with this list of inputs in separate interpreters (yet).
Once this PEP is merged, I expect we’ll see some of the other APIs in PyPi adopt them.

## How much overhead does a sub-interpreter have?

Short answer: More than a thread, less than a process.

Long answer: The interpreter has its own state, so whilst PEP554 will make it easy to create sub-interpreters, it will need to clone and initialize the following:

* modules in the __main__ namespace and importlib
* the sys dictionary containing
* builtin functions ( print() , assert etc)
* threads
* core configuration

The core configuration can be cloned easily from memory, but the imported modules are not so simple. Importing modules in Python is slow, so if creating a sub-interpreter means importing modules into another namespace each time, the benefits are diminished.

## What about asyncio?

The existing implementation of the asyncio event loop in the standard library creates frames to be evaluated but shares state within the main interpreter (and therefore shares the GIL).

After PEP554 has been merged, and likely in Python 3.9, an alternate event loop implementation could be implemented (although nobody has done so yet) that runs async methods within sub interpreters, and hence, concurrently.

## Sounds great, ship it!

Well, not quite.
Because CPython has been implemented with a single interpreter for so long, many parts of the code base use the “Runtime State” instead of the “Interpreter State”, so if PEP554 were to be merged in it’s current form there would still be many issues.
For example, the Garbage Collector (in 3.7<) state belongs to the runtime.
During the PyCon sprints changes have started to move the garbage collector state to the interpreter, so that each sub interpreter will have it’s own GC (as it should).
Another issue is that there are some “global”, variables lingering around in the CPython codebase and many C extensions. So when people suddenly started writing properly concurrent code, we might start to see some problems.
Another issue is that file handles belong to the process, so if you have a file open for writing in one interpreter, the sub interpreter won’t be able to access the file (without further changes to CPython).
In short, there are many other things still to be worked out.

## Conclusion: Is the GIL dead?

The GIL will still exist for single-threaded applications. So even when PEP554 is merged, if you have single-threaded code, it won’t suddenly be concurrent.
If you want concurrent code in Python 3.8, you have CPU-bound concurrency problems then this could be the ticket!

### When?

Pickle v5 and shared memory for multiprocessing will likely be Python 3.8 (October 2019) and sub-interpreters will be between 3.8 and 3.9.
If you want to play with my examples now, I’ve built a custom branch with all of the code required https://github.com/tonybaloney/cpython/tree/subinterpreters