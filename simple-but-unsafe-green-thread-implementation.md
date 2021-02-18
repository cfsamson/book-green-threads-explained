# An implementation of green threads

Before we start I'll mention that the code we write is quite unsafe and is not a "best practice" when writing Rust code. I want to try to make this as safe as possible without introducing a lot of unneeded complexity, so I encourage you dear reader to suggest a [PR to the projects repo](https://github.com/cfsamson/example-greenthreads) if you see something that could be done a safer way without making our code too complex.

## Let's get going

The first thing we do is to delete our example in our `main.rs`so we start from scratch and add the following:

```rust
#![feature(llvm_asm)]
#![feature(naked_functions)]
use std::ptr;

const DEFAULT_STACK_SIZE: usize = 1024 * 1024 * 2;
const MAX_THREADS: usize = 4;
static mut RUNTIME: usize = 0;
```

We enable two features the `asm` feature that we covered earlier, and the `naked_functions` feature, that we need to explain.

### naked\_functions

You see, when Rust compiles a function, it adds a small prologue and epilogue to each function and this causes some issues for us when we switch contexts since we end up with a misaligned stack. This worked fine in our first simple example but once we need to switch back to the same stack again we end up in trouble. Marking the a function as `#[naked]`removes the prologue and epilogue. This attribute is mostly used in relation to inline assembly.

{% hint style="info" %}
If you are interested you can read more about the `naked_functions` feature in [RFC \#1201](https://github.com/rust-lang/rfcs/blob/master/text/1201-naked-fns.md)
{% endhint %}

Our `DEFAULT_STACK_SIZE` is set to 2 MB which is more than enough for our use. We also set `MAX_THREADS` to 4 since we don't need more for our example.

The last constant `RUNTIME` is a pointer to our runtime \(yeah, I know, it's not pretty with a mutable global variable but we need it later and we're only setting this variable on runtime initialization\).

Let's start fleshing out something to represent our data:

```rust
pub struct Runtime {
    threads: Vec<Thread>,
    current: usize,
}

#[derive(PartialEq, Eq, Debug)]
enum State {
    Available,
    Running,
    Ready,
}

struct Thread {
    id: usize,
    stack: Vec<u8>,
    ctx: ThreadContext,
    state: State,
}

#[derive(Debug, Default)]
#[repr(C)]
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
}
```

`Runtime` is going to be our main entry point. We are basically going to create a very small, simple runtime to schedule and switch between our threads. The runtime holds an array of `Threads` and a `current` field to indicate which thread we are currently running.

`Thread` holds data for a thread. Each thread has an `id` so we can separate them from each other. The `stack` is similar to what we saw in our first example in earlier chapters. The `ctx` field is a context representing the data our CPU needs to resume where it left off on a stack, and a `state` which is our thread state.

`State` is an `enum` representing the states our threads can be in:

* `Available` means the thread is available and ready to be assigned a task if needed.
* `Running` means the thread is running
* `Ready` means the thread is ready to move forward and resume execution

`ThreadContext` holds data for the registers that CPU needs to resume execution on a stack.

{% hint style="info" %}
Go back to the chapter [Background Information](background-information.md) to read about the registers if you don't remember. These are the registers marked as "callee saved" in the specification of the x86-64 architecture.
{% endhint %}

Let's move on:

```rust
impl Thread {
    fn new(id: usize) -> Self {
        Thread {
            id,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Available,
        }
    }
}
```

This is pretty easy. A new thread starts in the `Available` state indicating it is ready to be assigned a task.

One thing to note is that we allocate our stack here. That is not needed and is not an optimal use of our resources since we allocate memory for threads we might need instead of allocating on first use. However, this keeps complexity down in the parts of our code that has a more important focus than allocating memory for our stack.

{% hint style="warning" %}
The important thing to note is that once a stack is allocated it must not move! No`push()`on the vector or any other methods that might trigger a reallocation. In a better version of this code we would make our own type that only exposes the methods we consider safe to use.

it's worth mentioning that`Vec<T>`has a [method ](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.into_boxed_slice)called`into_boxed_slice()`which returns a heap allocated slice `Box<[T]>`. Slices can't grow, so if we store that instead we can avoid the reallocation problem.
{% endhint %}

## Implementing the Runtime

All the code in this segment is in `impl Runtime` block meaning that they are methods on the `Runtime` struct.

```rust
impl Runtime {
    pub fn new() -> Self {
        // This will be our base thread, which will be initialized in 
        // the `running` state
        let base_thread = Thread {
            id: 0,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Running,
        };

        let mut threads = vec![base_thread];
        let mut available_threads: Vec<Thread> = (1..MAX_THREADS).map(|i| Thread::new(i)).collect();
        threads.append(&mut available_threads);

        Runtime {
            threads,
            current: 0,
        }
    }
```

When we instantiate our `Runtime` we set up a base thread. This thread will be set to the `Running` state and will make sure we keep the run-time running until all tasks are finished.

Then we instantiate the rest of the threads and set the current thread to `0` which is our base thread.

```rust
    /// This is cheating a bit, but we need a pointer to our Runtime 
    /// stored so we can call yield on it even if we don't have a 
    /// reference to it.
    pub fn init(&self) {
        unsafe {
            let r_ptr: *const Runtime = self;
            RUNTIME = r_ptr as usize;
        }
    }
```

Right now we need this. As I mentioned when going through our constants we need this to be able to call `yield` later on. It's not pretty, but we know that our runtime will be alive as long as there is any thread to `yield` so as long as we don't abuse this it's safe to do.

```rust
    pub fn run(&mut self) -> ! {
        while self.t_yield() {}
        std::process::exit(0);
    }
```

This is where we start running our run-time. It will continually call `t_yield()` until it returns `false` which means that there is no more work to do and we can exit the process.

```rust
    fn t_return(&mut self) {
        if self.current != 0 {
            self.threads[self.current].state = State::Available;
            self.t_yield();
        } 
    }
```

This is our return function that we call when the thread is finished. `return` is another reserved keyword in Rust so we name this `t_return()`. Make a note that the _user_ of our threads does not call this, we set up our stack so this is called when the task is done.

If the calling thread is the `base_thread` we don't do anything. Our runtime will call `yield` for us on the base thread. If it's called from a spawned thread we know it's finished since all threads have a `guard` function on top of their stack \(which we'll show further down\) and the only place this function is called is on our `guard` function.

We set its state to `Available` letting the runtime know it's ready to be assigned a new task and then immediately call `t_yield` which will schedule a new thread to be run.

Next: our `yield` function:

```rust
    fn t_yield(&mut self) -> bool {
        let mut pos = self.current;
        while self.threads[pos].state != State::Ready {
            pos += 1;
            if pos == self.threads.len() {
                pos = 0;
            }
            if pos == self.current {
                return false;
            }
        }

        if self.threads[self.current].state != State::Available {
            self.threads[self.current].state = State::Ready;
        }

        self.threads[pos].state = State::Running;
        let old_pos = self.current;
        self.current = pos;

        unsafe {
            let old: *mut ThreadContext = &mut self.threads[old_pos].ctx;
            let new: *const ThreadContext = &self.threads[pos].ctx;
            llvm_asm!(
                "mov $0, %rdi
                 mov $1, %rsi"::"r"(old), "r"(new)
            );
            switch();
        }
        self.threads.len() > 0
    }
```

This is the heart of our run-time. We have to name this `t_yield` since `yield` is a reserved keyword in Rust.

Here we go through all the threads and see if anyone is in the `Ready` state which indicates it has a task it is ready to make progress on. This could be a database call that has returned in a real world application.

If no thread is `Ready` we're all done. This is an extremely simple scheduler using only a round-robin algorithm, a real scheduler might have a much more sophisticated way of deciding what task to run next.

{% hint style="info" %}
This is a very naive implementation tailor-made for our example. What happens if our thread is not ready to make progress \(not in a `Ready` state\) and still waiting for a response from i.e. a database?

it's not too difficult to work around this, instead of running our code directly when a thread is `Ready` we could instead poll it for a status. For example it could return `IsReady` if it's really ready to run or `Pending` if it's waiting for some operation to finish. In the latter case we could just leave it in its `Ready` state to get polled again later. Does this sound familiar? If you've read about how [Futures](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.16/futures/task/enum.Poll.html#variant.Pending) work in Rust, we are starting to connect some dots on how this all fits together.
{% endhint %}

If we find a thread that's ready to be run we change the state of the current thread from `Running` to `Ready`.

Then we call `switch` which will save the current context \(the old context\) and load the new context into the CPU. The new context is either a new task, or all the information the CPU needs to resume work on an existing task.

{% hint style="info" %}
### The Inconvenient Truth About Naked Functions

Naked functions are not like normal functions. They don't accept formal arguments for example. Usually, if we call a function with two arguments, the compiler will place each argument in a register described by the calling convention for the platform. When we call a`#[naked]`function we need to take care of this ourselves. Therefore we pass in the address to our "old" and "new"  `ThreadContext` using assembly. `%rdi` is the register for the first argument in the Linux calling convention \(which is the same for MacOS\) and `%rsi`is the register used for the second argument.
{% endhint %}

The `self.threads.len() > 0`part in the end is just a way for us to prevent the compiler from optimizing our code away. This happens to me on Windows but not on Linux and is a common problem when running benchmarks for example. Therefore we could use [`std::hint::black_box`](https://doc.rust-lang.org/std/hint/fn.black_box.html)to prevent the compiler from going too far and skipping steps we need in order to execute the code faster. I chose a different route and as long as it's commented it should be OK. The code never reaches this point anyway.

Next up is our `spawn()`function:

```rust
pub fn spawn(&mut self, f: fn()) {
    let available = self
        .threads
        .iter_mut()
        .find(|t| t.state == State::Available)
        .expect("no available thread.");

    let size = available.stack.len();
    unsafe {
        let s_ptr = available.stack.as_mut_ptr().offset(size as isize);
        let s_ptr = (s_ptr as usize & !15) as *mut u8;
        std::ptr::write(s_ptr.offset(-16) as *mut u64, guard as u64);
        std::ptr::write(s_ptr.offset(-24) as *mut u64, skip as u64);
        std::ptr::write(s_ptr.offset(-32) as *mut u64, f as u64);
        available.ctx.rsp = s_ptr.offset(-32) as u64;
    }
    available.state = State::Ready;
}

// Remember to close "}" the `impl Runtime` block here
```

While `t_yield` is the logically interesting function I think this the technically most interesting.

This is where we set up our stack like we talked about in the previous chapter and making sure our stack looks like the one specified in the psABI [stack layout](the-stack.md#how-to-set-up-the-stack).

When we spawn a new thread we first check if there are any available threads \(threads in `Available` state\). If we run out of threads we panic in this scenario but there are several \(better\) ways to handle that. We keep things simple for now.

When we find an available thread we get the stack length and a pointer to our `u8` byte-array.

In the next segment we have to use some unsafe functions. First we make sure that the memory segment we'll use is 16-byte aligned. Then write the address to our `guard` function that will be called when the task we provide finishes and the function returns. Secondly we'll write the address to a `skip` function which is there just to handle the gap when we return from `f`so that `guard` will get called on a 16 byte boundary. The next value we write to the stack is the address to `f`.

{% hint style="info" %}
Remember how we explained how the stack works in [The Stack](the-stack.md) chapter. We want the `f` function to be the first to run so we set the base pointer to `f` and make sure it's 16 byte aligned. We then push the address to the `skip`function and lastly the `guard` function. Doing this makes sure that `guard` is 16-byte aligned so we adhere to the ABI requirements.
{% endhint %}

After we've written our function pointers to the stack, we set the value of `rsp` which is the stack pointer to the address of our provided function, so we start executing that first when we are scheduled to run.

Lastly we set the state as `Ready` which means we have work to do and that we are ready to do it. Remember, it's up to our "scheduler" to actually start up this thread.

We're now finished implementing our `Runtime`, if you got all this you basically understand _how_ green threads work. However, there are still a few details needed to implement them.

## Guard, skip and switch functions

```rust
fn guard() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_return();
    };
}
```

The function means that the function we passed in has returned and that means our thread is finished running its task so we de-reference our `Runtime` and call `t_return()`. We could have made a function that does some additional work when a thread is finished but right now our `t_return()` function does all we need. It marks our thread as `Available` \(if it's not our base thread\) and `yields` so we can resume work on a different thread.

```rust
#[naked]
fn skip() { }
```

There is not much happening in the `skip` function. We use the `#[naked]`attribute so that this function essentially compiles down to just `ret`instruction. `ret`will just pop off the next value from the stack and jump to whatever instructions that address points to. In our case this is the `guard`function. As you probably remember from [previous chapters](background-information.md) this makes sure that we comply with the ABI requirements.

```rust
pub fn yield_thread() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_yield();
    };
}
```

This is just a helper function that lets us call `yield` from an arbitrary place in our code. This is pretty unsafe though, if we call this and our `Runtime` is not initialized yet or the runtime is dropped it will result in undefined behavior. However making this safer is not a priority for us just to get our example up and running.

We are very soon at the finish line, just one more function to go. This one should be possible to understand without many comments if you've gone through the previous chapters:

```rust
#[naked]
#[inline(never)]
unsafe fn switch() {
    llvm_asm!("
        mov     %rsp, 0x00(%rdi)
        mov     %r15, 0x08(%rdi)
        mov     %r14, 0x10(%rdi)
        mov     %r13, 0x18(%rdi)
        mov     %r12, 0x20(%rdi)
        mov     %rbx, 0x28(%rdi)
        mov     %rbp, 0x30(%rdi)

        mov     0x00(%rsi), %rsp
        mov     0x08(%rsi), %r15
        mov     0x10(%rsi), %r14
        mov     0x18(%rsi), %r13
        mov     0x20(%rsi), %r12
        mov     0x28(%rsi), %rbx
        mov     0x30(%rsi), %rbp
        "
    );
}
```

So here is our inline Assembly. As you remember from our first example this is just a bit more elaborate where we first read out the values of all the registers we need and then sets all the register values to the register values we saved when we suspended execution on the "new" thread.

This is essentially all we need to do to save and resume execution.

Here we see the `#[naked]`attribute used again. Usually every function has a prologue and an epilogue and we don't want that here since this is all assembly and we want to handle everything ourselves. If we don't include this we will fail to switch back to our stack the second time.

{% hint style="info" %}
Most of this inline assembly is explained in the end of the chapter [An example we can build upon](an-example-we-can-build-upon.md) so if this seems foreign to you, go and read that part of the chapter and come back.
{% endhint %}

There are two things in this function that differs from our first function:

The first is the attribute `#[inline(never)]`, this attribute prevents the compiler from `inlining` this function. I spent some time figuring this out, but the code will fail when running on `--release`builds if we don't include it.

The `"volatile"` `option` is new. As I warned before, inline assembly can be a bit gnarly, but this indicates that our assembly has side effects. When providing our parameters as `input` we need to make sure the compiler knows that we are changing one of the parameters passed in and not only reading from them.

```rust
0x00($1) # 0
0x08($1) # 8
0x10($1) # 16
0x18($1) # 24
```

I mentioned this briefly, but here you see it in action. These are `hex` numbers indicating the _offset_ from the memory pointer to which we want to read/write. I wrote down the base-10 numbers as comments so you see we only offset the pointer in 8 byte steps which is the same size as the `u64` fields on our `ThreadContext` struct.

This is also why it's important to annotate `ThreadContext` with `#[repr(C)]` so we know that the data will be represented in memory this way and we write to the right field. The Rust ABI makes no guarantee that they are represented in the same order in memory, however the C-ABI does.

## The main function

```rust
fn main() {
    let mut runtime = Runtime::new();
    runtime.init();
    runtime.spawn(|| {
        println!("THREAD 1 STARTING");
        let id = 1;
        for i in 0..10 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
        println!("THREAD 1 FINISHED");
    });
    runtime.spawn(|| {
        println!("THREAD 2 STARTING");
        let id = 2;
        for i in 0..15 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
        println!("THREAD 2 FINISHED");
    });
    runtime.run();
}
```

As you see here we initialize our runtime and spawn two threads one that counts to 10 and `yields` between each count, and one that counts to 15. When we `cargo run` our project we should get the following output:

```text
Finished dev [unoptimized + debuginfo] target(s) in 2.17s
Running `target/debug/green_threads`
THREAD 1 STARTING
thread: 1 counter: 0
THREAD 2 STARTING
thread: 2 counter: 0
thread: 1 counter: 1
thread: 2 counter: 1
thread: 1 counter: 2
thread: 2 counter: 2
thread: 1 counter: 3
thread: 2 counter: 3
thread: 1 counter: 4
thread: 2 counter: 4
thread: 1 counter: 5
thread: 2 counter: 5
thread: 1 counter: 6
thread: 2 counter: 6
thread: 1 counter: 7
thread: 2 counter: 7
thread: 1 counter: 8
thread: 2 counter: 8
thread: 1 counter: 9
thread: 2 counter: 9
THREAD 1 FINISHED.
thread: 2 counter: 10
thread: 2 counter: 11
thread: 2 counter: 12
thread: 2 counter: 13
thread: 2 counter: 14
THREAD 2 FINISHED.
```

Beautiful!! Our threads alternate since they yield control on each count until thread 1 finishes and thread 2 counts the last numbers before it finishes its task.

## Congratulations

You have now implemented a super simple, but working, example of green threads. It was quite a ride we had to take, but if you came this far and read through everything you deserve a little break. Thanks for reading!

