# Why Futures

Before we go into the details about Futures in Rust, let's take a quick look
at the alternatives for handling concurrent programming in general and some
pros and cons for each of them.

## Threads provided by the operating system

Now one way of accomplishing this is letting the OS take care of everything for 
us. We do this by simply spawning a new OS thread for each task we want to
accomplish and write code like we normally would.

**Pros:**

- Simple
- Easy to use
- Switching between tasks is reasonably fast
- You get parallelism for free

**Cons:**

- OS level threads come with a rather large stack. If you have many tasks
waiting simultaneously (like you would in a web-server under heavy load) you'll
run out of memory pretty soon.
- There are a lot of syscalls involved. This can be pretty costly when the number
of tasks is high.
- The OS has many things it needs to handle. It might not switch back to your
thread as fast as you'd wish.
- Might not be an option on some systems

Using OS threads in Rust looks like this:

```rust
use std::thread;

fn main() {
    println!("So we start the program here!");
    let t1 = thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(200));
        println!("We create tasks which gets run when they're finished!");
    });

    let t2 = thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(100));
        println!("We can even chain callbacks...");
        let t3 = thread::spawn(move || {
            thread::sleep(std::time::Duration::from_millis(50));
            println!("...like this!");
        });
        t3.join().unwrap();
    });
    println!("While our tasks are executing we can do other stuff here.");

    t1.join().unwrap();
    t2.join().unwrap();
}
```

## Green threads

Green threads has been popularized by GO in the recent years. Green threads
uses the same basic technique as operating systems does to handle concurrency.

Green threads are implemented by setting up a stack for each task you want to
execute and make the CPU "jump" from one stack to another to switch between
tasks.

The typical flow will be like this:

1. Run som non-blocking code
2. Make a blocking call to some external resource
3. CPU jumps to the "main" thread which schedules a different thread to run and
"jumps" to that stack
4. Run some non-blocking code on the new thread until a new blocking call or the
task is finished
5. "jumps" back to the "main" thread and so on

These "jumps" are know as context switches. Your OS is doing it many times each
second as you read this.


The main advantages are:

1. Simple to use. The code will look like it does when using OS threads.
2. A "context switch" is reasonably fast
3. Each stack only gets a little memory to start with so you can have hundred of
thousands of green threads running.
4. It's easy to incorporate [_preemtion_](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/green-threads#preemptive-multitasking)
which puts a lot of control in the hands of the runtime implementors.

The main cons are:

1. The stacks might need to grow. Solving this is not easy and will have a cost.
2. You need to save all the CPU state on every switch
3. It's not a _zero cost abstraction_ (which is one of the reasons Rust removed
them early on).
4. Complicated to implement correctly if you want to support many different
platforms.

If you were to implement green threads in Rust, it could look something like
this:

    The example presented below is from an earlier book I wrote about green
    threads called [Green Threads Explained in 200 lines of Rust.](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)
    If you want to know what's going on everything is explained in detail
    in that book.

```rust
#![feature(asm)]
#![feature(naked_functions)]
use std::ptr;

const DEFAULT_STACK_SIZE: usize = 1024 * 1024 * 2;
const MAX_THREADS: usize = 4;
static mut RUNTIME: usize = 0;

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

impl Runtime {
    pub fn new() -> Self {
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

    pub fn init(&self) {
        unsafe {
            let r_ptr: *const Runtime = self;
            RUNTIME = r_ptr as usize;
        }
    }

    pub fn run(&mut self) -> ! {
        while self.t_yield() {}
        std::process::exit(0);
    }

    fn t_return(&mut self) {
        if self.current != 0 {
            self.threads[self.current].state = State::Available;
            self.t_yield();
        }
    }

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
            switch(&mut self.threads[old_pos].ctx, &self.threads[pos].ctx);
        }
        self.threads.len() > 0
    }

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
            ptr::write(s_ptr.offset(-24) as *mut u64, guard as u64);
            ptr::write(s_ptr.offset(-32) as *mut u64, f as u64);
            available.ctx.rsp = s_ptr.offset(-32) as u64;
        }
        available.state = State::Ready;
    }
}

fn guard() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_return();
    };
}

pub fn yield_thread() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_yield();
    };
}

#[naked]
#[inline(never)]
unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    asm!("
        mov     %rsp, 0x00($0)
        mov     %r15, 0x08($0)
        mov     %r14, 0x10($0)
        mov     %r13, 0x18($0)
        mov     %r12, 0x20($0)
        mov     %rbx, 0x28($0)
        mov     %rbp, 0x30($0)
   
        mov     0x00($1), %rsp
        mov     0x08($1), %r15
        mov     0x10($1), %r14
        mov     0x18($1), %r13
        mov     0x20($1), %r12
        mov     0x28($1), %rbx
        mov     0x30($1), %rbp
        ret
        "
    :
    :"r"(old), "r"(new)
    :
    : "volatile", "alignstack"
    );
}

fn main() {
    let mut runtime = Runtime::new();
    runtime.init();
    runtime.spawn(|| {
        println!("THREAD 1 STARTING");
        let id = 1;
        for i in 0..10 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
        println!("THREAD 1 FINISHED");
    });
    runtime.spawn(|| {
        println!("THREAD 2 STARTING");
        let id = 2;
        for i in 0..15 {
            println!("thread: {} counter: {}", id, i);
            yield_thread();
        }
        println!("THREAD 2 FINISHED");
    });
    runtime.run();
}
```

### Callback based approach

You probably already know this from Javascript since it's extremely common.
The whole idea behind a callback based approach is to save a pointer to a
set of instructions we want to run later on.

The basic idea of not involving threads as a primary way to achieve concurrency
is the common denominator for the rest of the approaches. Including the one
Rust uses today which we'll soon get to.

**Advantages:**

- Easy to implement in most languages
- No context switching
- Low memory overhead (in most cases)

**Drawbacks:**

- Each task must save the state it needs for later, the memory usage will grow
linearly with the number of tasks i .
- Can be hard to reason about, also known as "callback hell".
- Sharing state between tasks is a hard problem in Rust using this approach due
to it's ownership model.

The

If we did that in Rust it could look something like this:

```rust
fn program_main() {
    println!("So we start the program here!");
    set_timeout(200, || {
        println!("We create tasks which gets run when they're finished!");
    });
    set_timeout(100, || {
        println!("We can even chain callbacks...");
        set_timeout(50, || {
            println!("...like this!");
        })
    });
    println!("While our tasks are executing we can do other stuff here.");
}

fn main() {
    RT.with(|rt| rt.run(program_main));
}

use std::sync::mpsc::{channel, Receiver, Sender};
use std::{cell::RefCell, collections::HashMap, thread};

thread_local! {
    static RT: Runtime = Runtime::new();
}

struct Runtime {
    callbacks: RefCell<HashMap<usize, Box<dyn FnOnce() -> ()>>>,
    next_id: RefCell<usize>,
    evt_sender: Sender<usize>,
    evt_reciever: Receiver<usize>,
}

fn set_timeout(ms: u64, cb: impl FnOnce() + 'static) {
    RT.with(|rt| {
        let id = *rt.next_id.borrow();
        *rt.next_id.borrow_mut() += 1;
        rt.callbacks.borrow_mut().insert(id, Box::new(cb));
        let evt_sender = rt.evt_sender.clone();
        thread::spawn(move || {
            thread::sleep(std::time::Duration::from_millis(ms));
            evt_sender.send(id).unwrap();
        });
    });
}

impl Runtime {
    fn new() -> Self {
        let (evt_sender, evt_reciever) = channel();
        Runtime {
            callbacks: RefCell::new(HashMap::new()),
            next_id: RefCell::new(1),
            evt_sender,
            evt_reciever,
        }
    }

    fn run(&self, program: fn()) {
        program();
        for evt_id in &self.evt_reciever {
            let cb = self.callbacks.borrow_mut().remove(&evt_id).unwrap();
            cb();
            if self.callbacks.borrow().is_empty() {
                break;
            }
        }
    }
}
```