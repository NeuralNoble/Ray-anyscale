# Ray Anyscale

## 1. Introduction to Ray
### What is Ray?
Ray is an open-source framework designed to scale Python applications from a single machine to large clusters. It provides a unified interface for distributed computing, making it easier to build and run distributed applications.

### Why use Ray?
Ray addresses several challenges in distributed computing:

- Simplifies the transition from local to distributed execution
- Provides efficient resource management across a cluster
- Enables seamless scaling of applications
- Offers a rich ecosystem of libraries for various tasks

## Key Features

1. Distributed Task Execution: Ray allows you to run functions (tasks) across multiple machines with minimal changes to your code.
2. Actor Model: Ray implements the actor model, allowing you to create and manage stateful workers.
3. Flexible Scaling: Ray can scale from a single machine to thousands of nodes in a cluster.
4. Rich Ecosystem: Ray comes with libraries for machine learning (Ray Tune, RLlib), serving (Ray Serve), and data processing (Ray Data).
5. Fault Tolerance: Ray provides mechanisms to handle failures and recover from them.

Before we go on to understand what are tasks we first need to understand what are remote functions.

### Remote Functions

Remote functions in Ray are just regular Python functions with a special power: they can run on any computer in your Ray setup, not just the one you're typing on.

### Key Characteristics

1. **Asynchronous Execution** : When called, remote functions return immediately with an ObjectRef (future), allowing for non-blocking operations.

When we say "remote functions return immediately with an ObjectRef (future), allowing for non-blocking operations," we're talking about how remote functions behave differently from regular functions. Let's break it down:
- "Return immediately":
When you call a regular function, your program waits for it to finish before moving on. But when you call a remote function, it starts the work and immediately gives you back a kind of "receipt" (the ObjectRef). Your program can then continue right away without waiting.
-  "ObjectRef (future)":
This "receipt" is called an ObjectRef. It's like a promise that you'll get the actual result later. We sometimes call it a "future" because it represents a value you'll get in the future.
- "Non-blocking operations":
This means your program doesn't get stuck waiting. It can do other things while the remote function is working.
- in code it looks like this

```python
import ray
import time

ray.init()

@ray.remote
def slow_function():
    time.sleep(5)  # Pretend this is doing some heavy work
    return "Done!"
# Start the remote function
future = slow_function.remote()

# This prints immediately, we don't wait for slow_function to finish
print("I can do other things while waiting!")

# When we actually need the result, we use ray.get()
result = ray.get(future)
print(result)
```

2. **Distributed Execution**: They can be executed on any available worker in the Ray cluster.
3. **Stateless**: Each invocation of a remote function is independent and doesn't maintain state between calls.
4. **Serialization**: Arguments to and return values from remote functions are automatically serialized and deserialized by Ray.

When we say remote functions are "stateless," it means they have no memory of previous calls. Each time you use a remote function, it's like starting fresh, with no information carried over from before.
Let's break it down with a simple analogy:
Imagine a vending machine (this represents our remote function):

1. Stateful (what remote functions are NOT):
If the vending machine was stateful, it might remember your previous purchases. If you bought a soda yesterday, it might say "Welcome back! Want another soda?" the next time you use it.
2. Stateless (what remote functions ARE):
A stateless vending machine treats every interaction as new. It doesn't remember if you've used it before or what you bought last time. Each time you use it, it's like it's meeting you for the first time.

Here's a code example to illustrate this:
```python
import ray

ray.init()

@ray.remote
class StatefulCounter:
    def __init__(self):
        self.count = 0

    def increment(self):
        self.count += 1
        return self.count

@ray.remote
def stateless_increment(count):
    return count + 1

# Stateful example (this is NOT how remote functions work)
counter = StatefulCounter.remote()
print(ray.get(counter.increment.remote()))  # Prints 1
print(ray.get(counter.increment.remote()))  # Prints 2

# Stateless example (this IS how remote functions work)
print(ray.get(stateless_increment.remote(0)))  # Prints 1
print(ray.get(stateless_increment.remote(0)))  # Prints 1 again
```
In the stateful example, the StatefulCounter remembers its count between calls. This is not how remote functions work.

In the stateless example, stateless_increment doesn't remember anything. Each call starts fresh, so if you always pass in 0, you'll always get 1 back.

The key points are:
- Each call to a remote function is independent.
- Remote functions don't remember anything from previous calls.
- If you need to keep track of something between calls, you need to manage that yourself (maybe by passing in updated values each time).

This stateless nature makes remote functions simpler and more predictable, especially when running them across multiple computers.


Serialization is like packing a suitcase for a trip, and deserialization is like unpacking it when you arrive. Here's how it works in Ray:

- **Serialization:**
When you call a remote function, Ray needs to send the function and its arguments to another computer (or process) to run. But computers can't directly send Python objects over a network. So, Ray "packs" (serializes) these objects into a format that can be sent easily.
- **Deserialization:**
When the packed data arrives at the destination, Ray "unpacks" (deserializes) it back into Python objects so the function can use them.


## Tasks

Tasks in Ray are one of the core building blocks for parallel and distributed computing. They represent units of work that can be executed asynchronously and in parallel across a cluster of machines.*Think of a task as a job that you want to get done, but you're okay with someone else doing it for you.*
More technically Tasks are remote functions that can be executed anywhere in a Ray cluster. They are defined using the `@ray.remote` decorator.

### How to Create and Use Tasks?

- **Creating Tasks:**
To create a task, you simply decorate a Python function with @ray.remote

```
import ray

ray.init()

@ray.remote
def add(x, y):
    return x + y

# Start the task
result_ref = add.remote(2, 3)

# Get the result
result = ray.get(result_ref)
print(result)  # Prints: 5
```

- **Executing Tasks:**
Tasks are executed by calling the remote method and using ray.get() to retrieve the result

- **Task Scheduling:**
When you call a remote task, Ray's scheduler decides where to execute the task based on available resources and any specified resource requirements.

- **Return Values:**
Tasks return Ray ObjectRefs, which are futures representing the eventual result of the computation.

- **Resource Specification:**
You can specify resource requirements for tasks

```
@ray.remote(num_cpus=2, num_gpus=1)
def resource_intensive_task():
    # This task requires 2 CPUs and 1 GPU
    pass
```

- **Error Handling:**
If a task raises an exception, it's propagated when you call ray.get()

### Parallelism in Ray

example -

```
futures = [my_task.remote(i, i) for i in range(100)]
results = ray.get(futures)
```

**1. Parallel Execution:**
- This code is setting up 100 tasks to run in parallel.
- Each `my_task.remote(i, i)` call creates a separate task.
- Ray will try to run these tasks simultaneously, depending on available resources.


**2. How it works:**

- The list comprehension [my_task.remote(i, i) for i in range(100)] quickly creates 100 task invocations.
- Each my_task.remote(i, i) returns immediately with an ObjectRef (future).
- The futures list contains 100 ObjectRefs, not actual results.

**3. Resource management:**

- Ray will distribute these tasks across available workers in your Ray cluster.
- If you have multiple CPUs or machines, many tasks can truly run at the same time.
- If you have fewer resources than tasks, Ray will automatically queue and schedule them efficiently.

**4. Getting results:**
- `ray.get(futures)` waits for all 100 tasks to complete and collects their results.
- The results are returned in the same order as the futures list.


**5. Potential speedup:**
- If `my_task` takes 1 second to run sequentially, doing 100 tasks might take 100 seconds.
- With Ray, if you have enough resources, it could potentially complete in just over 1 second.

## What are Actors?

Actors in Ray are like special workers that remember things. Think of an actor as a person with a notebook. They can do tasks and write down information to use later.

#### Key Characteristics of Actors
- Stateful: Unlike tasks, actors can remember information between calls.
- Persistent: They keep running and maintain their state until you specifically tell them to stop.
- Concurrent: Multiple actors can run at the same time, each with its own state.
- Single-threaded: By default, an actor processes one method call at a time.

I'll explain the key characteristics of actors using an analogy of a personal assistant, and then provide code examples for each characteristic.

Imagine an actor as a personal assistant named Alex. Here are the key characteristics:

1. Stateful (Memory)
2. Persistent (Always available)
3. Concurrent (Multiple assistants can work simultaneously)
4. Single-threaded (Focuses on one task at a time)

Let's break these down:

**1. Stateful (Memory):**
   Analogy: Alex keeps a notebook and remembers information from previous tasks.
   
   Code example:
   ```python
   import ray
   
   @ray.remote
   class Assistant:
       def __init__(self):
           self.tasks_completed = 0
   
       def do_task(self):
           self.tasks_completed += 1
           return f"Completed task. Total tasks done: {self.tasks_completed}"
   
   ray.init()
   alex = Assistant.remote()
   print(ray.get(alex.do_task.remote()))  # Output: Completed task. Total tasks done: 1
   print(ray.get(alex.do_task.remote()))  # Output: Completed task. Total tasks done: 2
   ```

**2. Persistent (Always available):**
   Analogy: Alex doesn't go home at the end of the day; they're always ready to work.
   
   Code example:
   ```python
   # Alex is available as long as the Ray cluster is running
   # You can keep using the same 'alex' instance
   print(ray.get(alex.do_task.remote()))  # Output: Completed task. Total tasks done: 3
   # Even after a long time...
   print(ray.get(alex.do_task.remote()))  # Output: Completed task. Total tasks done: 4
   ```

**3. Concurrent (Multiple assistants can work simultaneously):**
   Analogy: You can hire multiple assistants, each working independently.
   
   Code example:
   ```python
   alex = Assistant.remote()
   bob = Assistant.remote()
   
   # Alex and Bob can work at the same time
   alex_task = alex.do_task.remote()
   bob_task = bob.do_task.remote()
   
   print(ray.get(alex_task))  # Output: Completed task. Total tasks done: 1
   print(ray.get(bob_task))   # Output: Completed task. Total tasks done: 1
   ```
**4. Single Threaded**
Now, I'll explain what "single-threaded" means for actors using a simple analogy and then provide a code example.

Imagine an actor as a chef in a kitchen:

1. Single-threaded means the chef can only do one task at a time.
2. If you ask the chef to make a sandwich and then immediately ask them to make a salad, they'll finish the sandwich first before starting the salad.
3. The chef won't try to make both at the same time, which could lead to a mess.

Now, let's see this in action with a code example:

```python
import ray
import time

ray.init()

@ray.remote
class Chef:
    def make_sandwich(self):
        print("Starting to make a sandwich")
        time.sleep(3)  # Pretend it takes 3 seconds to make a sandwich
        print("Finished making a sandwich")
        return "Sandwich ready!"

    def make_salad(self):
        print("Starting to make a salad")
        time.sleep(2)  # Pretend it takes 2 seconds to make a salad
        print("Finished making a salad")
        return "Salad ready!"

# Hire our chef
chef = Chef.remote()

# Ask the chef to make a sandwich and a salad
sandwich_order = chef.make_sandwich.remote()
salad_order = chef.make_salad.remote()

# Get the results
print(ray.get(sandwich_order))
print(ray.get(salad_order))
```

When you run this code, you'll see something like this:

```
Starting to make a sandwich
Finished making a sandwich
Starting to make a salad
Finished making a salad
Sandwich ready!
Salad ready!
```

What's happening here:

1. We ask the chef to make a sandwich and then immediately ask for a salad.
2. However, the chef (our actor) finishes the sandwich completely before starting the salad.
3. This happens even though making a salad is quicker (2 seconds) than making a sandwich (3 seconds).

This is what "single-threaded" means - the actor (our chef) handles one request at a time, in the order they were received, and finishes each task before moving to the next.



## Creating an Actor

To create an actor, you use the `@ray.remote` decorator on a Python class:

```python
import ray

@ray.remote
class Counter:
    def __init__(self):
        self.value = 0

    def increment(self):
        self.value += 1
        return self.value

    def get_value(self):
        return self.value
```

## Using an Actor

Here's how you use an actor:

1. Create an actor instance:
   ```python
   counter = Counter.remote()
   ```

2. Call methods on the actor:
   ```python
   increment_ref = counter.increment.remote()
   value = ray.get(increment_ref)
   print(value)  # Prints: 1
   ```

## Detailed Explanation of Actor Behavior

1. **State Persistence**:
   - When you call `counter.increment.remote()` multiple times, the actor remembers and updates its internal `value`.
   ```python
   ray.get(counter.increment.remote())  # Returns 2
   ray.get(counter.increment.remote())  # Returns 3
   ```

2. **Concurrent Access**:
   - Multiple parts of your program can use the same actor concurrently.
   - Ray ensures that method calls to a single actor are processed one at a time to avoid conflicts.

3. **Multiple Actors**:
   - You can create multiple instances of an actor, each with its own state:
   ```python
   counter1 = Counter.remote()
   counter2 = Counter.remote()
   ray.get(counter1.increment.remote())  # Returns 1
   ray.get(counter2.increment.remote())  # Also returns 1
   ```

4. **Passing Actors**:
   - You can pass actor handles to tasks or other actors:

```
@ray.remote
def use_counter(counter):
    return ray.get(counter.increment.remote())

result = ray.get(use_counter.remote(counter))
```

## Advanced Actor Features

1. **Resource Specification**:
   - You can specify resource requirements for actors:
   ```python
   @ray.remote(num_cpus=2, num_gpus=1)
   class ResourceHeavyActor:
       # This actor requires 2 CPUs and 1 GPU
       pass
   ```

2. **Actor Options**:
   - You can set options when creating an actor instance:
   ```python
   Counter.options(name="my_counter", lifetime="detached").remote()
   ```
   This creates a named, detached actor that persists even if the creating process exits.

3. **Async Methods**:
   - Actors can have async methods for concurrent processing:
   ```python
   @ray.remote
   class AsyncActor:
       async def async_method(self):
           # Do some async work
           pass
   ```

4. **Actor Pools**:
   - You can create a pool of actors for load balancing:
   
   ```python
   from ray.util.actor_pool import ActorPool
   
   actors = [Counter.remote() for _ in range(4)]
   pool = ActorPool(actors)
   ```

## Use Cases for Actors

1. **Shared State**: When you need to maintain state across multiple calls or tasks.
2. **Resource Management**: For managing access to a limited resource, like a database connection.
3. **Encapsulation**: To encapsulate complex logic and state in an object-oriented manner.
4. **Stateful Services**: For implementing services that need to maintain state, like a game server or a chat room.

## Comparison with Tasks

- Tasks are stateless and good for parallel, independent work.
- Actors are stateful and good for maintaining information across calls.
- Use tasks for distributing work that doesn't need to remember anything.
- Use actors when you need to keep track of changing information.

## Best Practices

1. Avoid creating too many actors, as each actor consumes resources.
2. Use actors for maintaining state, not for one-off computations.
3. Be mindful of the single-threaded nature of actors when designing your system.
4. Use `ray.get()` judiciously to avoid blocking unnecessarily.
