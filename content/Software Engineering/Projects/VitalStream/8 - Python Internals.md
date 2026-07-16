Core Python language mechanics discussed while going through the codebase — separate from [[7 - Packages]], which is about third-party/stdlib packages, not language constructs. Coming from C++/JS, no prior Python — each topic starts with the simple version (mapped to something already familiar) before going into the actual mechanism. Depth isn't cut, just sequenced so the jargon lands on top of a mental model instead of ahead of one.

---

## `__init__.py` — the package marker

### Simple version
In JS/Node, `require('./app')` or `import x from './app'` looks for an `index.js` inside `app/` and treats that as what the folder exports — `index.js`'s presence is what makes a folder importable as one thing. Python's `__init__.py` plays the same role: it's the file that runs when you `import app`, and its presence marks `app/` as an importable package, same job as `index.js`.

Where it differs from Node: Node genuinely *requires* `index.js` (or a `"main"` field in `package.json`) to resolve a bare folder import. Python is looser — since Python 3.3, a plain folder with no `__init__.py` still imports fine (Python silently falls back to treating it as what's called a "namespace package," covered below). So technically the file isn't required anymore.

We still add an empty `__init__.py` to every folder in this repo (`app/`, `app/routers/`, `scripts/`, `tests/`) for the same reason you'd keep an empty `index.js` in a JS project even when not strictly needed: predictable behavior, no silent fallback resolution happening behind your back.

C++ doesn't have a real equivalent here — there's no folder-level import concept, you just `#include` specific files. This is a genuinely new idea coming from C++, a very familiar one coming from JS.

### The actual mechanism
Python resolves imports through an actual search process, not magic. `sys.path` is a list of directories Python searches (think of it like C++'s include search paths, or Node's `node_modules` resolution chain, but simpler: it's just a flat list of folders). For each folder on `sys.path`, Python asks: "does this folder have what I'm looking for?"

When it finds a folder with `__init__.py` inside, it's a **regular package**: the search stops right there (first match wins, nothing else on `sys.path` with the same name even gets looked at), `__init__.py` runs exactly once, and the result is cached — so re-importing the same module elsewhere in the program doesn't re-run the file, it just reuses the cached module object. Anything you define inside `__init__.py` becomes accessible as `packagename.thatthing` — this is the one place `__init__.py` can act like a JS barrel file (`export * from './something'`) if you wanted it to, though this repo doesn't use it that way — the files are all empty, pure markers.

**In this repo**: `app/__init__.py`, `app/routers/__init__.py`, `scripts/__init__.py`, `tests/__init__.py` are all empty. Nothing to export, nothing to initialize — they exist purely to force the "regular package" resolution path described above rather than the fallback path below.

**A real bug this connects to**: importing `app.kafka_consumer` first imports `app` (running `app/__init__.py`), then imports `kafka_consumer` as a piece of it. When `kafka_consumer.py` does `from app.cache import set_latest_reading` (`kafka_consumer.py:14`), it copies that function into its *own* local namespace — like doing `const { setLatestReading } = require('./cache')` in JS instead of `const cache = require('./cache'); cache.setLatestReading(...)`. Once you copy a reference out like that, replacing the *original* later doesn't change the copy you're holding. That's exactly why a test-time mock aimed at `app.cache.set_latest_reading` silently failed to take effect and had to be redirected at `app.kafka_consumer.set_latest_reading` instead — the full story is in [[5 - Debugging Journal]] (Bug 3). Worth knowing this is a general "how imports bind names" issue, not a pytest-specific quirk.

### Regular package vs. implicit namespace package
Here's the fallback path mentioned above, and why the distinction actually matters rather than being trivia.

If Python searches a `sys.path` folder and finds a bare directory named `foo/` with **no** `__init__.py` inside, it doesn't give up — it notes "there's a folder here" and keeps checking every *other* folder on `sys.path` for the same name too. Only once it's checked everywhere, and never found a real `__init__.py`-having folder or a `foo.py` file, does it stitch all those bare `foo/` folders together into what's called a **namespace package** — a package assembled from multiple locations instead of one. This is the mechanism that, for example, lets `google-cloud-storage` and `google-cloud-pubsub` — two entirely separate pip-installed packages — both contribute to one shared `google.cloud` namespace.

The asymmetry that causes bugs: a real package/`__init__.py` **stops the search immediately** (first match wins, full stop); a bare namespace folder **doesn't stop anything** — it keeps accumulating across the whole search. So you can't half-opt-in: if even one folder named `foo` anywhere on `sys.path` has an `__init__.py`, `foo` becomes a plain regular package using *only* that one folder — any other bare `foo/` folders elsewhere are just ignored, not merged in.

| | Regular package (`__init__.py` present) | Namespace package (no `__init__.py`) |
|---|---|---|
| Analogy | folder with `index.js` | folder Node would still resolve some other way |
| Backed by | one specific file, runs once | no file, nothing runs |
| Location | exactly one folder | can be stitched together from several folders |
| Search behavior | stops the search — first match wins | keeps searching, merges everything found |
| When it's used | normal application code (this repo) | plugin ecosystems spanning multiple installed packages |

**Gotcha worth remembering**: if you deleted `app/__init__.py` right now, `import app.main` would probably still work fine — Python would just quietly fall back to treating `app/` as a namespace package, since nothing else on your machine is named `app`. Tests would likely still pass. Nothing breaks loudly. That's exactly why it's easy to get sloppy about these files — the failure mode isn't a crash, it's losing guarantees you didn't notice you had (predictable single-location resolution, the option to add init code later, clean behavior in packaging/type-checking tools that treat the two cases differently). This repo keeps explicit empty `__init__.py` files everywhere specifically to not rely on the fallback — reasonable, since this is one repo with one deployable, not a multi-package plugin ecosystem that would actually benefit from namespace packages.

---

## `__init__` (the method) — this one is just the constructor

### Simple version
Directly maps to what you already know:

```cpp
// C++
class Device {
public:
    Device(std::string externalId, std::string label)
        : externalId(externalId), label(label) {}
};
```

```js
// JS
class Device {
  constructor(externalId, label) {
    this.externalId = externalId;
    this.label = label;
  }
}
```

```python
# Python
class Device:
    def __init__(self, external_id, label):
        self.external_id = external_id
        self.label = label
```

Same job: runs automatically right after the object is created, sets up initial state. `self` is Python's more explicit version of `this` — the only difference is Python makes you name it yourself as the first parameter instead of it being implicit.

Two small differences from C++, neither worth losing sleep over:
- Python splits "allocate the object" (`__new__`) from "initialize it" (`__init__`) into two separate steps — but you basically never touch `__new__` day to day, so `__init__` = constructor covers 99% of real code.
- No constructor overloading. C++ lets you define several constructors with different argument types; Python (like JS) gets exactly one `__init__`, so "optional variants" are faked with default parameter values — the same trick you'd already reach for in JS (`function foo(x, y = 10)`).

### The actual mechanism (for when you need it)
`SomeClass(args)` is really two calls chained together: `cls.__new__(cls, args)` builds the raw, empty object, and then — only if that returned something that actually *is* an instance of `cls` — `obj.__init__(args)` fills in its state. `__init__` isn't allowed to return anything except `None`; returning anything else is an error.

In multiple inheritance (Python allows a class to inherit from more than one parent, unlike C++'s more restricted version of this or JS's total lack of it), `super().__init__(...)` doesn't necessarily call "the parent" — it calls "whoever's next" in a computed order across all the parent classes (called the MRO). Not relevant anywhere in this codebase (no multiple inheritance here), but worth knowing the term if it comes up.

### The genuinely interesting part: nobody wrote a single `__init__` in this codebase
In C++ or JS, if a class needs a constructor, you write it. In this project, every class's constructor is generated automatically instead, by three different mechanisms, each trusting its input a different amount:

1. **`AlertCandidate`** (`app/alerts.py:7-12`) uses the `@dataclass` decorator — you just list the fields (`metric`, `value`, `severity`, `message`) and Python writes the constructor for you. Closest JS comparison: imagine declaring class fields and having the `constructor(){ this.x = x; ... }` boilerplate generated automatically. Constructed with plain keyword args in `alerts.py:24-31`. No validation at all — it trusts you.

2. **`Device`, `Reading`, `Alert`** (`app/models.py`) — these inherit from a SQLAlchemy base class (`Base`, `app/db.py:10-11`), which supplies a free constructor that takes whatever keyword arguments you give it and dumps them straight onto the object. Like a JS constructor that just does `Object.assign(this, args)` with zero checking. `Device(external_id=..., label=...)` (`routers/devices.py:23`) works despite `models.py` never defining `__init__` itself.

3. **The Pydantic classes** in `app/schemas.py` (`DeviceCreate`, `ReadingIn`, etc.) also get a free constructor — but this one actually validates and coerces the input, raising an error on bad data. Closest comparison: a runtime validation library like `zod` in the JS/TS world, checking shape and types before letting the object exist. `ReadingIn.model_validate(raw)` (`kafka_consumer.py:42`) can and does throw on a malformed Kafka message.

**Why it's built this way, not by accident**: those three trust levels line up exactly with how trustworthy the data source is. Internal value objects built entirely by our own code (`AlertCandidate`) need no validation. Database rows (`Device`/`Reading`/`Alert`) skip validation too, because the database's own constraints (uniqueness, foreign keys, not-null) are the actual safety net for that data. Anything crossing an external boundary — an HTTP request body, a raw Kafka message from a device — goes through Pydantic, which is the layer that actually distrusts its input. That boundary-aligned split is the pattern worth being able to explain, not a side effect of which library happened to be convenient.

---

## `try` / `except` — same idea as `try`/`catch`, one real difference

### Simple version
Same shape as C++/JS — run some code, and if a specific kind of error happens, jump to a handler instead of crashing:

```cpp
// C++
try {
    riskyThing();
} catch (const std::exception& e) {
    // handle it
}
```

```js
// JS
try {
  riskyThing();
} catch (e) {
  // handle it
}
```

```python
# Python
try:
    risky_thing()
except SomeError:
    # handle it
```

`except SomeError:` is like C++'s typed `catch (const SomeError& e)` — it only catches that specific error type (or subclasses of it), not everything. JS's `catch (e)` catches anything and leaves type-checking to you (`e instanceof SomeError`); Python's `except` does that filtering up front, in the syntax itself.

**Where it's used in this repo** — `scripts/simulate_device.py:47-56`:
```python
try:
    while args.count == 0 or sent < args.count:
        for device_id in device_ids:
            reading = make_reading(device_id, force_anomaly=random.random() < args.anomaly_rate)
            publish_reading(reading)
            print(f"sent {reading}")
        sent += 1
        time.sleep(args.interval)
except KeyboardInterrupt:
    pass
```
`KeyboardInterrupt` is what Python raises when the user hits **Ctrl+C**. `pass` just means "do nothing" — Python requires every block to have a body (no empty `{}` like C++/JS), so `pass` is the explicit "intentionally nothing here" placeholder. In plain English: *keep looping forever sending fake readings, but if the user hits Ctrl+C, stop quietly instead of crashing with a traceback.*

### The actual mechanism
The interesting part isn't the syntax — it's *how* Ctrl+C becomes a catchable error at all, and this is genuinely different across all three languages:
- **C++**: Ctrl+C (SIGINT) kills the process by default. Intercepting it means installing a signal handler via `signal()`/`sigaction()` — a completely separate mechanism from `try`/`catch`.
- **Node/JS**: Ctrl+C emits a `SIGINT` event, handled via `process.on('SIGINT', () => {...})` — an event-listener mechanism, again unrelated to `try`/`catch`.
- **Python**: the interpreter itself converts that same OS signal into an ordinary `KeyboardInterrupt` exception and raises it wherever the program happens to be executing at that moment — inside the loop body, inside `time.sleep()`, anywhere. It then propagates up the call stack exactly like any other exception, so plain `try`/`except` catches it. Python is the odd one out here: "user pressed Ctrl+C" goes through the *same* mechanism as any other bug, instead of a separate signal-handling API.

That's why the `try` wraps the *entire* `while` loop including `time.sleep()`, rather than one specific line — you don't know which exact statement will be running when Ctrl+C lands, so you catch it at the coarse, outer level instead of guessing.

**Gotcha worth knowing**: `KeyboardInterrupt` deliberately does **not** inherit from `Exception` — it inherits from the higher `BaseException`. So a broad `except Exception:` (a common "catch anything that goes wrong" handler) will **not** catch a `KeyboardInterrupt` — it still propagates and stops the program. That's intentional: Python doesn't want a generic bug handler accidentally swallowing the user's Ctrl+C. A bare `except:` with no type at all *does* catch everything, `KeyboardInterrupt` included — which is exactly why bare `except:` is considered bad practice, the same way C++'s `catch (...)` is a smell: it can silently hide real bugs alongside whatever you actually meant to catch.

---

## Lazy singleton race condition — `app/kafka_producer.py`

A real bug found and fixed while reviewing the code, not a hypothetical — worth remembering because it's the exact same class of bug C++11 had to specifically fix.

### The bug (original code)
```python
_producer: KafkaProducer | None = None

def get_producer() -> KafkaProducer:
    global _producer
    if _producer is None:            # check
        _producer = KafkaProducer(...)  # act
    return _producer
```
This "check `None`, then construct, then assign" pattern is **not** thread-safe. Two threads can both pass the `if _producer is None:` check before either finishes constructing and assigning:
1. Thread A checks → `True`, starts building `KafkaProducer(...)` (this opens real sockets, does broker metadata negotiation — not instant)
2. Before Thread A finishes, Thread B also checks → still `True`, since Thread A hasn't assigned yet
3. Thread B also builds its own `KafkaProducer(...)`
4. Whichever thread assigns to the global `_producer` last "wins" — the other's fully-built producer is discarded with nothing calling `.close()` on it, leaking its socket and internal background sender thread

**Why the GIL doesn't save you**: the GIL guarantees individual bytecode instructions don't tear, not that a multi-line check→construct→assign sequence is atomic. Worse, `KafkaProducer(...)` construction does real socket I/O, and I/O calls release the GIL while waiting on the network — practically guaranteeing another thread gets scheduled right in the middle of construction. This isn't a rare theoretical edge case.

### The C++ connection
This is *exactly* the double-checked-locking bug that predates C++11. A lazy function-local static in old C++ (`if (!p) p = new Foo();`) had the identical race. C++11 fixed it by guaranteeing function-local `static` initialization is thread-safe by the language spec itself ("magic statics" — the compiler inserts a hidden guard so only one thread ever runs the initializer). Python gives **no equivalent guarantee** for a plain module-level `if x is None: x = ...` — you have to build the equivalent yourself.

### Did it matter in this codebase?
Traced every caller of `get_producer()` (via `publish_reading`/`publish_alert`): `scripts/simulate_device.py` is a single-threaded CLI loop, and `app/kafka_consumer.py`'s `run()` is a single-threaded `for message in consumer:` loop. So as wired up, this was a **latent** bug, not a live one — nothing called it concurrently. The concrete scenario that *would* make it live: `app/routers/devices.py`'s endpoints are plain `def`, not `async def` — FastAPI/Starlette runs sync path operations in a background thread pool, so concurrent HTTP requests to those routes really do run on different threads today. None of them currently touch the Kafka producer, but the day an endpoint calls `publish_reading`/`publish_alert` directly, this race goes live.

### The fix — double-checked locking, applied in `app/kafka_producer.py:11-31`
```python
_producer: KafkaProducer | None = None
_producer_lock = threading.Lock()

def get_producer() -> KafkaProducer:
    global _producer
    if _producer is None:
        with _producer_lock:
            if _producer is None:   # re-check after acquiring the lock
                _producer = KafkaProducer(...)
    return _producer
```
The outer unlocked check avoids taking the lock on every call once initialized (the common case, forever, after the first call). The inner locked re-check is the actual fix — it closes the window where two threads both passed the outer check before either had a chance to finish constructing.

**Alternative considered**: `functools.cache` on a zero-arg function — CPython's `lru_cache`/`cache` implementation holds an internal lock around cache population, so it's thread-safe for exactly this "compute once, reuse forever" shape, with less code. Went with the explicit lock instead here since it makes the thread-safety guarantee visible in the code rather than resting on a CPython implementation detail that isn't part of the language spec.

---

## `global` — why `get_producer()` breaks without it

### Simple version
The surprising thing coming from C++/JS: **in Python, assigning to a name inside a function always creates a new local variable by default, even if an outer variable with that name already exists.** `global` turns that default off for one specific name.

```js
// JS — just works, modifies the outer variable directly
let counter = 0;
function increment() { counter = counter + 1; }
```
```cpp
// C++ — just works, modifies the global directly
int counter = 0;
void increment() { counter = counter + 1; }
```
```python
# Python — creates a NEW local, does not touch the global
# (and actually raises UnboundLocalError before it even gets this far)
counter = 0
def increment():
    counter = counter + 1
```
C++ and JS only shadow an outer variable if you explicitly declare a local one (`let`/`const`/`var`, or a typed declaration). Python flips this: the mere act of `x = ...` anywhere in a function body is what makes `x` local for that whole function — no separate declaration syntax needed to create the shadow, and no way to opt out except `global`.

### Where it bites — `get_producer()`, `app/kafka_producer.py:15-31`
```python
_producer: KafkaProducer | None = None

def get_producer() -> KafkaProducer:
    global _producer
    if _producer is None:
        with _producer_lock:
            if _producer is None:
                _producer = KafkaProducer(...)
    return _producer
```
Delete `global _producer` (line 16) and this breaks on the very first call, not eventually. Python compiles the function body before ever running it, and during that compile step it scans the whole body for any assignment to a name. It finds `_producer = KafkaProducer(...)` nested inside the `if`/`with` — and regardless of whether that branch actually executes, decides "`_producer` is local for this entire function." That applies to *every* reference to the name, including the very first line, `if _producer is None:` — reading a local before it's been assigned raises `UnboundLocalError: local variable '_producer' referenced before assignment`, on the read, not the write. `global _producer` tells the compiler up front to target the module-level variable for every reference in this function instead.

### The mechanism
This is a compile-time decision, not a runtime lookup. Local variables compile to fast, array-indexed slot access (`LOAD_FAST`/`STORE_FAST` bytecode); globals go through a dictionary lookup into the module's namespace (`LOAD_GLOBAL`/`STORE_GLOBAL`). `global x` is what tells the compiler to emit the `_GLOBAL` variants for `x` throughout the function instead of the `_FAST` variants — a static instruction about which bytecode to generate, not something with runtime behavior of its own.

**One-liner to lock in**: C++/JS need a keyword to *create a local shadow*; Python needs a keyword to *avoid creating one*. Reading an outer/global name inside a Python function needs no special syntax at all (same as C++/JS) — it's specifically *assignment* that flips Python's default from "touch the outer one" to "make a new local," and `global` is the only way back.

A sibling keyword, `nonlocal`, solves the same problem one scope level up — referring to an *enclosing function's* variable from a nested function, rather than the module global. Not used anywhere in this codebase, but the same idea.
