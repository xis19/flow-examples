# Flow Examples

Examples for using [flow language](https://github.com/apkar/foundationdb/tree/master/flow) developed in FoundationDB project. These
examples are intended to learn the basics of the language.

## Compile

This package requires Mono for compiling actor compiler and cmake>=3.12. On
Ubuntu systems, Mono can be installed via:

```bash
sudo apt install mono-mcs libmono-system-data4.0-cil libmono-system-xml-linq4.0-cil libmono-system-data-datasetextensions4.0-cil
```

Actual building the package is as simple as:

```bash
mkdir build
cd build
cmake ../flow-examples
make
```

## Examples

### hello.cpp

```cpp
void hello() {
  Promise<string> p;
  Future<string> f = p.getFuture();
  p.send( "Hello, World!" );
  cout<< f.get() << endl; // f is already set
}

void hello2() {
  Promise<string> p;
  Future<string> f = p.getFuture();
  cout << "Before send: promise isSet = " << p.isSet() << ", future.isReady = "
       << f.isReady() << endl;
  p.send( "Hello, World!" );
  cout << "After send: promise isSet = " << p.isSet() << ", future.isReady = "
       << f.isReady() << endl;
  cout<< f.get() << endl; // f is already set
}
```

The output for the above code is:

```shell
# ./hello
Running test hello
Hello, World!

Running test hello2
Before send: promise isSet = 0, future.isReady = 0
After send: promise isSet = 1, future.isReady = 1
Hello, World!

```

In `hellow2()`, before sending the value, the future `f` is not ready. `f` only
becomes ready after Promise sends the value.

### calc.cpp & calc.actor.cpp

The ACTOR waits for `f`, then does the computation, and finally returns a future.

```cpp
ACTOR Future<int> asyncAdd(Future<int> f, int offset) {
    int value = wait( f );
    return value + offset;
}
```

The main function in `calc.cpp` shows that we can have the future of this `asyncAdd`,
which only becomes available after `p` sends a value to `f`.

```cpp
int main(int argc, char** argv) {
  Promise<int> p;
  Future<int> f = p.getFuture();
  Future<int> result = asyncAdd(f, 10);
  cout << "Future f.isReady = " << f.isReady() << ", result.isReady = "
       << result.isReady() << endl;
  p.send( 5 );
  cout << "Send 5 to f" << endl;
  cout << "Result is " << result.get() << endl; // f is already set

  return 0;
}
```

The output looks like this:

```shell
root@fce4696cc1b9:/opt/foundation/foundationdb/flow-examples# ./calc
Future f.isReady = 0, result.isReady = 0
Send 5 to f
Result is 15
```

So it's clear that `asyncAdd()` is not blocking.

### void.cpp & void.actor.cpp

```cpp
void test1() {
  Future<Void> f = Void(), n = Never();
  cout << "f = Void(), f.isReady() = " << f.isReady() << endl;
  cout << "n = Never(), n.isReady() = " << f.isReady() << endl;
}

void test2() {
  Future<Void> result = dummy();
  cout << "result = dummy(), result.isReady() = " << result.isReady() << endl;
}

void test3() {
  Future<Void> result = foo();
  cout << "result = foo(), result.isReady() = " << result.isReady() << endl;
}
```

In `test1()`, both `f` and `n` becomes immediately ready. In `test2()`, result is
also ready. The output is:

```bash
root@fce4696cc1b9:/opt/foundation/foundationdb/flow-examples# ./void
Running test test1
f = Void(), f.isReady() = 1
n = Never(), n.isReady() = 1

Running test2...
dummy onChange changed
result = dummy(), result.isReady() = 1
```

The `dummy()` is defined as:

```cpp
ACTOR Future<Void> dummy() {
  state Future<Void> onChange = Void();

  loop choose {
    when (wait(onChange)) {
      cout << "onChange changed\n";
      break;
    }
  }
  return Void();
}

ACTOR Future<Void> foo() {
  state Future<Void> onChange = dummy();

  loop choose {
    when (wait(onChange)) {
      cout << "foo onChange changed\n";
      break;
    }
  }
  cout << "foo returned.\n";
  return Void();
}
```

What happens if we change `dummy()` not use assign a value for `onChange`? I.e.,

```c++
ACTOR Future<Void> dummy() {
  state Future<Void> onChange;
...
}
```

This will result in segfault:

```bash
root@fce4696cc1b9:/opt/foundation/foundationdb/flow-examples# gdb void
GNU gdb (Ubuntu 7.9-1ubuntu1) 7.9
Copyright (C) 2015 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from void...done.
(gdb) r
Starting program: /opt/foundation/foundationdb/flow-examples/void
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Running test1...
f = Void(), f.isReady() = 1
n = Never(), n.isReady() = 1

Running test2...

Program received signal SIGSEGV, Segmentation fault.
0x0000000000410416 in Error::code (this=0x22) at /opt/foundation/foundationdb/flow/Error.h:43
43		int code() const { return error_code; }
(gdb) where
#0  0x0000000000410416 in Error::code (this=0x22) at /opt/foundation/foundationdb/flow/Error.h:43
#1  0x0000000000411c94 in SAV<Void>::isSet (this=0x0) at flow/flow.h:372
#2  0x0000000000410cbf in Future<Void>::isReady (this=0x7fffffffe320) at flow/flow.h:605
#3  0x0000000000416bbd in (anonymous namespace)::DummyActorState<(anonymous namespace)::DummyActor>::a_body1loopBody1 (this=0x7ffff7f49040, loopDepth=1) at void.actor.g.cpp:76
#4  0x00000000004166f7 in (anonymous namespace)::DummyActorState<(anonymous namespace)::DummyActor>::a_body1loopHead1 (this=0x7ffff7f49040, loopDepth=1) at void.actor.g.cpp:65
#5  0x0000000000415feb in (anonymous namespace)::DummyActorState<(anonymous namespace)::DummyActor>::a_body1 (this=0x7ffff7f49040, loopDepth=0) at void.actor.g.cpp:32
#6  0x0000000000415543 in (anonymous namespace)::DummyActor::DummyActor (this=0x7ffff7f49000) at void.actor.g.cpp:158
#7  0x00000000004155e1 in dummy () at void.actor.cpp:7
#8  0x000000000040e744 in test2 () at void.cpp:17
#9  0x000000000040e822 in main (argc=1, argv=0x7fffffffe558) at void.cpp:23
```

Inspect void.actor.g.cpp, it's the `__when_expr_0.isReady()` generating the
error, where `sav` of Future class is nullptr.

```cpp
	int a_body1loopBody1(int loopDepth)
	{
															#line 11 "void.actor.cpp"
		StrictFuture<Void> __when_expr_0 = onChange;
															#line 10 "void.actor.cpp"
		if (static_cast<DummyActor*>(this)->actor_wait_state < 0) return a_body1Catch1(actor_cancelled(), std::max(0, loopDepth - 1));
															#line 76 "void.actor.g.cpp"
		if (__when_expr_0.isReady()) { if (__when_expr_0.isError()) return a_body1Catch1(__when_expr_0.getError(), std::max(0, loopDepth - 1)); else return a_body1loopBody1when1(__when_expr_0.get(), loopDepth); };
		static_cast<DummyActor*>(this)->actor_wait_state = 1;
															#line 11 "void.actor.cpp"
		__when_expr_0.addCallbackAndClear(static_cast<ActorCallback< DummyActor, 0, Void >*>(static_cast<DummyActor*>(this)));
															#line 81 "void.actor.g.cpp"
		loopDepth = 0;

		return loopDepth;
	}
```

This seems to be a nice property: all the state variables of type Future should
be initialized (be a static one like `Void()` or returned by another actor),
because later on there is going to be a `wait()` on this Future.
`test3()` shows a composition of these two actors.

`test4()` and `test5()` show that if `Never()` is used. The future is **NOT** firing any
continutations (callbacks generated by flow compiler).

```cpp
void test4() {
  Future<Void> result = never();
  cout << "result = never(), result.isReady() = " << result.isReady() << endl;
}

void test5() {
  const int not_used = 1;
  Future<Void> result = never2(not_used);
  cout << "result = never2(), result.isReady() = " << result.isReady() << endl;
}

ACTOR Future<Void> never() {
  wait( Future<Void>(Never()) );
  // Not reached, because wiat() never returns.
  cout << "never returned.\n";
  return Void();
}

ACTOR Future<Void> never2(int select) {
  loop {
    state Future<Void> reg = Never();

    choose {
      when( wait( reg )) { break; }
    }
  }
  // Not reached, because "reg" never breaks from the loop
  cout << "never2 returned.\n";
  return Void();
}
```

Results:

```bash
Running test4...
result = never(), result.isReady() = 0

Running test5...
result = never2(), result.isReady() = 0
```

The behavior of `Never()` is defined by `Future` and `SAV` class in
[flow/flow.h](https://github.com/apkar/foundationdb/blob/master/flow/flow.h),
where `SAV::send(Never)` doesn't fire the callback:

```cpp
class Future {
  ...
	Future(const T& presentValue)
		: sav(new SAV<T>(1, 0))
	{
		sav->send(presentValue);
	}
	Future(Never)
		: sav(new SAV<T>(1, 0))
	{
		sav->send(Never());
	}

struct SAV {
  ...
	template <class U>
	void send(U && value) {
		ASSERT(canBeSet());
		new (&value_storage) T(std::forward<U>(value));
		this->error_state = Error::fromCode(SET_ERROR_CODE);
		while (Callback<T>::next != this)
			Callback<T>::next->fire(this->value());
	}

	void send(Never) {
		ASSERT(canBeSet());
		this->error_state = Error::fromCode(NEVER_ERROR_CODE);
	}

```

### Introducing `g_network` and `delay()` (See loop.cpp & delay.actor.cpp)

Sometimes we want the actor to perform an action after a time delay. The flow
language provides a `delay()` function for that (`flow/flow.h`):

```cpp
inline double now() { return g_network->now(); }
inline Future<Void> delay(double seconds, int taskID = TaskDefaultDelay) { return g_network->delay(seconds, taskID); }
inline Future<Void> delayUntil(double time, int taskID = TaskDefaultDelay) { return g_network->delay(std::max(0.0, time - g_network->now()), taskID); }
```

So the `delay()` is internally managed by g_network, which can be viewed as an
event manager. As a result, the `main()` needs to initialze it (`loop.cpp`):

```cpp
int main(int argc, char **argv) {
  int randomSeed = platform::getRandomSeed();
  g_random = new DeterministicRandom(randomSeed);
  g_nondeterministic_random = new DeterministicRandom(platform::getRandomSeed());
  g_network = newNet2( NetworkAddress(), false );
  ... (elided)
}
```

Note the `g_network->run()` is the event handling loop, which is called before
the code exits `main()`.

In the `delayTest()` actor, a call to `g_network->stop()` causes the event loop
to exit.

```cpp
ACTOR Future<Void> delay_five() {
  state Future<Void> reg = Never();
  state Future<Void> onChange = Void();
  loop choose {
    when( wait( reg )) { break; }
    when( wait( onChange ) ) {
      wait( delay(5) );
      break;
    }    
  }
  cout << "delay_five returned.\n";
  return Void();
}

ACTOR void delayTest() {
  wait( delay_five() );
  cout << "ACTOR delayTest done...\n" << endl;
  g_network->stop();
}
```

Results are:

```bash
# ./loop delay
Running delayTest...

delayTest running... (expecting 5s delay)
delay_five returned.
ACTOR delayTest done...

delayTest existing...
```

## Common Pitfalls

Here are some mistakes I have made when using flow.

### Forget to overwrite Future with Never()

A common pattern in flow is a loop waiting on several futures. Whenever a
future is ready, we perform an action. It's very important to reset the future
inside the action block.

```cpp
ACTOR Future<Void> infinite_loop() {
  state Future<Void> onChange = Void();
  state int count = 0;

  loop choose {
    when( wait( delay(0.01) ) ) { break; }
    when( wait( onChange ) ) {
      // onChange = Never();
      count++;
      if (count % 1000 == 0) {
        std::cout << "Loop count " << count << std::endl;
      }
    }
  }
  cout << "loop returned.\n";
  return Void();
}
```

In the above example, instead of existing after 0.01 seconds, there is an
infinite loop printing out messages. Run the `./loop delay` command to see
the output.

The reason is `onChange` is set to a Void outside the loop, so the first time
inside the loop, the condition `wait(onChange)` immediately returns. Inside the
block, `onChange` is not reset to `Never()` or another future. As a result, the
next run of the loop sees `onChange` and enter the block again and again. This
causes an infinite loop and we never exit the `infinite_loop()`.

### No wait, no catch

Take a look at this example. Will exceptions raised from `raise_exception()` be
caught in `exceptTest()`?

```cpp

ACTOR Future<int> raise_exception() {
  wait(delay(0.1));
  cout << "Throw exception in " << __FUNCTION__ << endl;
  throw value_too_large();
}

ACTOR Future<Void> exceptTest() {
  try {
    state Future<int> s = raise_exception();
    state Future<Void> f = delay(1.0);
    loop choose {
      when (wait(f)) {
        break;
      }
      // No wait means no exceptions caught.
      // when (int i = wait(s)) {}
    }
  } catch (Error& err) {
    cout << "Caught error: " << err.name() << endl;
  }
  g_network->stop();
  return Void();
}
```

Surprisely, the answer is **NO**:

```bash
root@4f6ed515f435:/opt/foundation/foundationdb/flow-examples# ./loop except
Running exceptTest...

exceptTest running... (expecting no exceptions being caught)
Throw exception in a_body1cont1
exceptTest existing...
```

In order to catch the exception, we need to have a `wait` on future `s` in the
`exceptTest()`. Uncomment the code in `exceptTest()`, then we can catch the
exception (this time we see *"Caught error: value_too_large"*):

```bash
root@4f6ed515f435:/opt/foundation/foundationdb/flow-examples# ./loop except
Running exceptTest...

exceptTest running... (expecting no exceptions being caught)
Throw exception in a_body1cont1
Caught error: value_too_large
exceptTest existing...
```

**TODO: explain how exceptions are handled in the above code.**

Typically, the code pattern to handle Futures returned by actors is to use an
actor collection and wait on the actor collection:

```cpp
  state PromiseStream<Future<Void>> addActor;
  state Future<Void> collection = actorCollection( self->addActor.getFuture() );

  self->addActor.send( actor1() );
  self->addActor.send( actor2() );

  try {
    loop choose {
      ...
      when (wait(collection) ) { ASSERT(false); throw internal_error(); }
    }
  } catch (Error& e) {...}
```

### Broken promise is an error

`Promise()` in flow is interesting. Internally, it uses `struct SAV` that maintains
two counters, one for the number of promises and one for the number of futures.
Whenever `Promise::getFuture()` is called, the counter for futures increases by
one. If we are destorying a promise and there are futures referencing this
promise, then these futures will get a `broken_promise()` error.

```cpp
ACTOR Future<int> promise_broken(Future<int>* f) {
  state Promise<int> p;

  *f = p.getFuture();
  wait(delay(0.1));
  // Exiting without sending value results in broken promise.
  // p.send(1);
  return 2;
}

ACTOR Future<Void> brokenTest() {
  try {
    state Future<int> s;
    state Future<int> f = promise_broken(&s);
    loop choose {
      when (int v = wait(f)) {
        cout << "Got value from function " << v << endl;
        f = Never();
      }
      when (int v = wait(s)) {
        cout << "Got value from promise " << v << endl;
        s = Never();
      }
    }
  } catch (Error& err) {
    cout << "Error: " << err.name() << endl;
  }
  g_network->stop();
  return Void();
}
```

Test run results are:

```bash
root@4f6ed515f435:/opt/foundation/foundationdb/flow-examples# ./loop broken
Running brokenTest...

brokenTest running... (expecting broken promise)
Error: broken_promise
brokenTest existing...
```

What happens is that the Promise `p` in `promise_broken()` only lives in the
function. Once this function exits, `p` sets the internal `SAV` to a broken
promise error.

Note if Promise `p` send a value in `promise_broken()`, then the output is the
following (no errors):

```bash
root@4f6ed515f435:/opt/foundation/foundationdb/flow-examples# ./loop broken
Running brokenTest...

brokenTest running... (expecting broken promise)
Got value from promise 1
Got value from function 2
(... keep running for ever...)
```

See `Promise`, `Future`, and `SAV` implementations in
[flow/flow.h](https://github.com/apkar/foundationdb/blob/master/flow/flow.h).

