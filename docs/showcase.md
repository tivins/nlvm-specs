# NL Showcase — Task Manager

This document presents a complete, realistic NL program: a **command-line task manager** that reads and writes tasks
to a file. The program is split across 6 source files following NL's one-class-per-file convention, and exercises the
language's core principles in a natural way.

> **Scope:** This showcase uses only features that are fully specified as of version 0.8.25. It deliberately avoids
> under-specified areas (operator overloading details, `match` exhaustiveness, `instanceof` syntax). For the full
> language reference, see [specs.md](specs.md).

---

## Principles illustrated

| Principle | Where it appears |
|-----------|-----------------|
| **Null safety** (`T\|null`, `??`) | `Task.description`, `FileRepository.load`, `Main` input handling |
| **Immutability** (`readonly`, `const`) | `Task` (readonly class), const methods and parameters |
| **Checked exceptions** (`throws`, `try`/`catch`/`finally`) | `FileRepository` I/O, `Main` error handling |
| **Typed enums** | `Priority` (string-backed) |
| **Interfaces and inheritance** | `Repository<T>` interface, `FileRepository` implementation |
| **Templates** | `Repository<T>` generic interface |
| **Closures / anonymous functions** | Filtering and printing tasks |
| **Standard library** | `system.Out`, `system.In`, `system.io.File`, `system.List<T>`, `system.String` |
| **Entry point and exit codes** | `Main.main(string[] args)` |
| **`nodiscard`** | `TaskService.addTask` returns a result that must not be ignored |

---

## Source files

```
app/
├── model/
│   ├── Priority.nl
│   └── Task.nl
├── storage/
│   ├── Repository.nl
│   └── FileRepository.nl
├── service/
│   └── TaskService.nl
└── Main.nl
```

---

### `app/model/Priority.nl` — Typed enum

A string-backed enum representing task priority levels. Typed enums give each case an explicit value accessible via
`.value`, and `from()` / `tryFrom()` provide safe conversion from strings.
See [specs.md § Enums](specs.md#enums).

```nl
namespace app.model;

enum Priority: string {
    Low = "low",
    Medium = "medium",
    High = "high",
}
```

---

### `app/model/Task.nl` — Readonly class, null safety, const methods

`Task` is a **readonly class**: all its properties are immutable after construction, making task objects safe to share
without defensive copies. The optional `description` field uses **`string|null`** — the compiler ensures that `null`
can only be assigned to nullable types and that nullable values are checked before use.
See [specs.md § Readonly](specs.md#readonly), [specs.md § Union types](specs.md#union-types-and-explicit-nullable).

```nl
namespace app.model;

use app.model.Priority;

class readonly Task implements Stringable {
    public int id;
    public string title;
    public string|null description;
    public Priority priority;
    public bool done;

    public construct(int id, string title, string|null description, Priority priority) {
        this.id = id;
        this.title = title;
        this.description = description;
        this.priority = priority;
        this.done = false;
    }

    public construct(int id, string title, string|null description, Priority priority, bool done) {
        this.id = id;
        this.title = title;
        this.description = description;
        this.priority = priority;
        this.done = done;
    }

    public string toString() {
        string status = this.done ? "[x]" : "[ ]";
        string desc = this.description ?? "";
        string base = status + " #" + this.id + " " + this.title + " (" + this.priority.value + ")";
        if (desc != "") {
            return base + " — " + desc;
        }
        return base;
    }
}
```

**Key points:**
- `class readonly` — every property is immutable after `construct` completes ([specs.md § Readonly](specs.md#readonly)).
- `string|null description` — compile-time null safety; the caller must handle `null` explicitly ([specs.md § Union types](specs.md#union-types-and-explicit-nullable)).
- `this.description ?? ""` — nullish coalescing provides a default value when the description is `null` ([specs.md § Conditional operators](specs.md#conditional-operators)).
- `implements Stringable` — the `toString()` method allows string concatenation and `system.Out.print` to convert `Task` to a string automatically ([specs.md § Stringable](specs.md#stringable-interface)).

---

### `app/storage/Repository.nl` — Interface with template

A generic interface defining the contract for task persistence. The template parameter `<type T>` makes the interface
reusable for any entity type. Concrete implementations provide the actual storage mechanism.
See [specs.md § Template class](specs.md#template-class), [specs.md § Extends, Implements](specs.md#extends-implements).

```nl
namespace app.storage;

template <type T>
interface Repository {
    public system.List<T> findAll() throws IOException;
    public void save(const system.List<T> items) throws IOException;
}
```

**Key points:**
- `template <type T>` — the interface is generic; `FileRepository` will instantiate it as `Repository<Task>`.
- `throws IOException` — file-based persistence can fail; the interface makes this explicit in the contract.
- `const system.List<T> items` — the `const` parameter guarantees the `save` method will not modify the caller's list.

---

### `app/storage/FileRepository.nl` — Checked exceptions, file I/O, closures

The concrete implementation stores tasks as one line per task in a plain text file. This file exercises **checked
exceptions** (every `throws IOException` must be handled or propagated), **`try`/`finally`** for resource cleanup,
and **closures** for filtering.
See [specs.md § Exceptions](specs.md#exceptions), [stdlib.md § system.io](stdlib.md#systemiofile).

```nl
namespace app.storage;

use app.model.Task;
use app.model.Priority;

class FileRepository implements Repository<Task> {
    private string filePath;

    public construct(string filePath) {
        this.filePath = filePath;
    }

    public system.List<Task> findAll() throws IOException {
        auto tasks = new system.List<Task>();

        if (!system.io.File.exists(this.filePath)) {
            return tasks;
        }

        auto handle = system.io.File.open(this.filePath);
        try {
            string|null line;
            while ((line = handle.readLine()) != null) {
                Task|null task = this.parseLine(line);
                if (task != null) {
                    tasks.pushBack(task);
                }
            }
        } finally {
            handle.close();
        }

        return tasks;
    }

    public void save(const system.List<Task> items) throws IOException {
        string content = "";
        for (const auto task : items) {
            content = content + this.formatLine(task) + "\n";
        }
        system.io.File.writeAllText(this.filePath, content);
    }

    private Task|null parseLine(string line) {
        string[] parts = line.split("|");
        if (parts.length() < 5) {
            return null;
        }

        int id;
        try {
            id = system.Int.parse(parts[0]);
        } catch (NumberFormatException ex) {
            return null;
        }

        string title = parts[1];
        string|null description = parts[2] == "" ? null : parts[2];
        Priority|null priority = Priority.tryFrom(parts[3]);
        bool done = parts[4] == "1";

        if (priority == null) {
            return null;
        }

        return new Task(id, title, description, priority, done);
    }

    private string formatLine(const Task task) const {
        string desc = task.description ?? "";
        string doneFlag = task.done ? "1" : "0";
        return task.id + "|" + task.title + "|" + desc + "|" + task.priority.value + "|" + doneFlag;
    }
}
```

**Key points:**
- `implements Repository<Task>` — the template interface is instantiated with a concrete type.
- `try { ... } finally { handle.close(); }` — ensures the file handle is released even if an exception occurs ([specs.md § Exception handling](specs.md#exception-handling)).
- `Priority.tryFrom(parts[3])` — safe enum conversion returning `Priority|null` instead of throwing ([specs.md § Enums methods](specs.md#enums-methods)).
- `const Task task` — the parameter is read-only; only `const` methods of `Task` can be called ([specs.md § Const methods](specs.md#class-methods)).
- `formatLine(...) const` — the method is `const`, guaranteeing it does not modify `this` ([specs.md § Const methods](specs.md#class-methods)).

---

### `app/service/TaskService.nl` — Validation, `nodiscard`, closures

The service layer handles business logic: generating IDs, validating input, and filtering tasks. The `addTask` method
is marked **`nodiscard`** because ignoring its return value (the created task) is likely a bug.
See [specs.md § Nodiscard](specs.md#nodiscard), [specs.md § Anonymous Functions](specs.md#anonymous-functions).

```nl
namespace app.service;

use app.model.Task;
use app.model.Priority;
use app.storage.Repository;

class TaskService {
    private Repository<Task> repository;
    private system.List<Task> tasks;
    private int nextId;

    public construct(Repository<Task> repository) throws IOException {
        this.repository = repository;
        this.tasks = repository.findAll();
        this.nextId = this.computeNextId();
    }

    public nodiscard Task addTask(string title, string|null description, Priority priority) {
        auto task = new Task(this.nextId, title, description, priority);
        this.nextId++;
        this.tasks.pushBack(task);
        return task;
    }

    public Task[] listPending() const {
        auto all = this.allTasks();
        return all.filter((Task t) => !t.done);
    }

    public Task[] listByPriority(Priority priority) const {
        auto all = this.allTasks();
        return all.filter((Task t) => t.priority == priority);
    }

    public bool markDone(int id) {
        for (int i = 0; i < this.tasks.size(); i++) {
            auto existing = this.tasks.get(i);
            if (existing.id == id) {
                auto updated = new Task(existing.id, existing.title, existing.description,
                                        existing.priority, true);
                this.tasks.set(i, updated);
                return true;
            }
        }
        return false;
    }

    public void flush() throws IOException {
        this.repository.save(this.tasks);
    }

    public int count() const {
        return this.tasks.size();
    }

    private Task[] allTasks() const {
        auto result = new Task[this.tasks.size()];
        for (int i = 0; i < this.tasks.size(); i++) {
            result[i] = this.tasks.get(i);
        }
        return result;
    }

    private int computeNextId() const {
        int maxId = 0;
        for (int i = 0; i < this.tasks.size(); i++) {
            auto task = this.tasks.get(i);
            if (task.id > maxId) {
                maxId = task.id;
            }
        }
        return maxId + 1;
    }
}
```

**Key points:**
- `nodiscard Task addTask(...)` — the compiler warns if the returned `Task` is discarded ([specs.md § Nodiscard](specs.md#nodiscard)).
- `listPending()` and `listByPriority()` use **closures** passed to `filter()` — anonymous functions capture local context ([specs.md § Anonymous Functions](specs.md#anonymous-functions)).
- `const` methods (`listPending`, `listByPriority`, `count`, `allTasks`, `computeNextId`) guarantee they do not modify the service's state.
- `markDone` creates a **new** `Task` object (since `Task` is readonly, its properties cannot be changed after construction). This is the idiomatic pattern for updating readonly objects.

---

### `app/Main.nl` — Entry point, user interaction, error handling

The entry point ties everything together. It reads user commands from `system.In`, delegates to `TaskService`, and
handles errors with `try`/`catch`/`finally`. The exit code signals success or failure to the operating system.
See [specs.md § Entry point](specs.md#entry-point).

```nl
namespace app;

use app.model.Priority;
use app.model.Task;
use app.storage.FileRepository;
use app.service.TaskService;

class Main {
    public static int main(string[] args) {
        string dataFile = "tasks.dat";
        if (args.length() > 1) {
            dataFile = args[1];
        }

        TaskService|null service = null;
        try {
            auto repo = new FileRepository(dataFile);
            service = new TaskService(repo);
        } catch (IOException ex) {
            system.Err.println("Failed to load tasks: " + ex.message);
            return 1;
        }

        system.Out.println("Task Manager — " + service.count() + " task(s) loaded.");
        system.Out.println("Commands: add, list, done, quit");

        bool running = true;
        while (running) {
            system.Out.print("> ");
            string|null input = system.In.readLine();
            string command = (input ?? "quit").trim();

            switch (command) {
                case "add":
                    Main.handleAdd(service);
                    break;
                case "list":
                    Main.handleList(service);
                    break;
                case "done":
                    Main.handleDone(service);
                    break;
                case "quit":
                    running = false;
                    break;
                default:
                    system.Out.println("Unknown command: " + command);
                    break;
            }
        }

        try {
            service.flush();
            system.Out.println("Tasks saved.");
        } catch (IOException ex) {
            system.Err.println("Failed to save tasks: " + ex.message);
            return 1;
        }

        return 0;
    }

    private static void handleAdd(TaskService service) {
        system.Out.print("Title: ");
        string title = (system.In.readLine() ?? "").trim();
        if (title == "") {
            system.Out.println("Title cannot be empty.");
            return;
        }

        system.Out.print("Description (or empty): ");
        string rawDesc = (system.In.readLine() ?? "").trim();
        string|null description = rawDesc == "" ? null : rawDesc;

        system.Out.print("Priority (low/medium/high) [medium]: ");
        string rawPriority = (system.In.readLine() ?? "").trim();
        string priorityInput = rawPriority == "" ? "medium" : rawPriority;

        Priority|null priority = Priority.tryFrom(priorityInput);
        if (priority == null) {
            system.Out.println("Invalid priority: " + priorityInput);
            return;
        }

        Task task = service.addTask(title, description, priority);
        system.Out.println("Created: " + task.toString());
    }

    private static void handleList(const TaskService service) {
        Task[] pending = service.listPending();
        if (pending.length() == 0) {
            system.Out.println("No pending tasks.");
            return;
        }

        system.Out.println("Pending tasks:");
        pending.forEach((Task t) => {
            system.Out.println("  " + t.toString());
        });
    }

    private static void handleDone(TaskService service) {
        system.Out.print("Task ID: ");
        string rawId = (system.In.readLine() ?? "").trim();
        if (rawId == "") {
            return;
        }

        int id;
        try {
            id = system.Int.parse(rawId);
        } catch (NumberFormatException ex) {
            system.Out.println("Invalid ID: " + rawId);
            return;
        }

        if (service.markDone(id)) {
            system.Out.println("Task #" + id + " marked as done.");
        } else {
            system.Out.println("Task #" + id + " not found.");
        }
    }
}
```

**Key points:**
- `public static int main(string[] args)` — the required entry-point signature; the return value is the process exit code ([specs.md § Entry point](specs.md#entry-point)).
- `input ?? "quit"` — if `system.In.readLine()` returns `null` (EOF), the program gracefully defaults to quitting.
- `switch (command)` with `break` — fall-through semantics require explicit `break` ([specs.md § Switch/Match](specs.md#switchmatch)).
- `try { ... } catch (IOException ex) { ... }` — checked exceptions from file I/O must be handled.
- `try { id = system.Int.parse(rawId); } catch (NumberFormatException ex) { ... }` — runtime exceptions do not *require* handling, but *can* be caught for robustness ([specs.md § Exceptions](specs.md#exceptions)).
- `const TaskService service` in `handleList` — guarantees the method only calls `const` methods on the service.
- `pending.forEach((Task t) => { ... })` — closure passed to the array's built-in `forEach` method.

---

## Running the program

```
$ nlc app/Main.nl app/model/*.nl app/storage/*.nl app/service/*.nl -o taskmanager
$ nlvm taskmanager
Task Manager — 0 task(s) loaded.
Commands: add, list, done, quit
> add
Title: Write NL showcase
Description (or empty): Complete realistic program for the spec repo
Priority (low/medium/high) [medium]: high
Created: [ ] #1 Write NL showcase (high) — Complete realistic program for the spec repo
> add
Title: Review coherence tracker
Description (or empty):
Priority (low/medium/high) [medium]:
Created: [ ] #2 Review coherence tracker (medium)
> list
Pending tasks:
  [ ] #1 Write NL showcase (high) — Complete realistic program for the spec repo
  [ ] #2 Review coherence tracker (medium)
> done
Task ID: 1
Task #1 marked as done.
> list
Pending tasks:
  [ ] #2 Review coherence tracker (medium)
> quit
Tasks saved.
```

---

## Summary of NL principles

This showcase demonstrates how NL's features work together in practice:

1. **Null safety** is not an afterthought — nullable types (`T|null`) are part of the type system, and the compiler
   rejects unsafe access at compile time. The `??` operator provides concise defaults.

2. **Immutability** is enforced at multiple levels: `readonly` classes guarantee objects cannot be mutated after
   construction; `const` methods and parameters restrict what code is allowed to modify.

3. **Checked exceptions** force developers to acknowledge failure modes. File I/O declares `throws IOException`, and
   the compiler ensures every call site either handles or propagates the exception.

4. **Templates** enable type-safe generic code (`Repository<T>`) without sacrificing compile-time checking —
   the compiler monomorphizes templates into concrete types.

5. **Typed enums** provide self-documenting constants with safe conversion (`tryFrom` returns `null` instead of
   throwing).

6. **Closures** integrate naturally with array/collection methods (`filter`, `forEach`), enabling concise
   functional-style code alongside the OOP structure.

7. **One class per file, namespace = directory** — the project layout mirrors the namespace hierarchy, making
   navigation predictable.
