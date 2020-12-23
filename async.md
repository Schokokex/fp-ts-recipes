# Async tasks

## Tasks that always succeed

If you're working with asynchronous tasks that are guaranteed to succeed, use [Task](https://gcanti.github.io/fp-ts/modules/Task.ts).

```code|lang-ts
import * as T from "fp-ts/lib/Task";

const deepThought: T.Task<number> = () => Promise.resolve(42);

deepThought().then(n => {
  console.log(`The answer is ${n}.`);
});
```

## Tasks that may fail

If you're working with asynchronous tasks that may fail, use [TaskEither](https://gcanti.github.io/fp-ts/modules/TaskEither.ts). If the JSON in this example is malformed (try it!), an "I'm sorry" message is displayed.

```code|lang-ts
import * as Either from "fp-ts/lib/Either";
import * as TaskEither from "fp-ts/lib/TaskEither";
import { pipe } from "fp-ts/lib/pipeable";

type Named = { name: string }

const fetchGreeting = TaskEither.tryCatch<Error, Named>(
  () => new Promise(resolve => resolve(JSON.parse('{ "name": "Carol" }'))),
  reason => new Error(String(reason))
);

fetchGreeting()
  .then(either =>
    pipe(
      either,
      Either.fold(
        err => `I'm sorry, I don't know who you are. (${err.message})`,
        (x: Named) => `Hello, ${x.name}!`
      )
    )
  )
  .then(console.log);
```

## Work with a list of tasks in parallel

JavaScript provides `Promises.all` to wait for a list of Promises.

```code|lang-ts
Promise.all([Promise.resolve(1), Promise.resolve(2)]).then(console.log); // [1, 2]
```

With `Task`s you can achieve the same using `sequence`. Both the `Promise.all` and the `sequence` approach run in parallel and wait until all results have arrived before they proceed.

```code|lang-ts
import * as A from "fp-ts/lib/Array";
import * as T from "fp-ts/lib/Task";

const tasks = [T.of(1), T.of(2)];
A.array
  .sequence(T.task)(tasks)()
  .then(console.log); // [ 1, 2 ]
```

## Run a list of tasks in sequence

If you need to run a list of `Task`s in sequence, i.e. you have to wait for one `Task` to finish before you run the second `Task`, you can use the `taskSeq` instance.

```code|lang-ts
import * as A from "fp-ts/lib/Array";
import * as T from "fp-ts/lib/Task";
import { pipe } from "fp-ts/lib/pipeable";

const log = <A>(x: A) => {
  console.log(x);
  return x;
};

const tasks = [
  pipe(T.delay(200)(T.of("first")), T.map(log)),
  pipe(T.delay(100)(T.of("second")), T.map(log))
];

// Parallel: logs 'second' then 'first'
A.array.sequence(T.task)(tasks)();

// Sequential: logs 'first' then 'second'
A.array.sequence(T.taskSeq)(tasks)();
```

## Work with tasks with different type

What if the types are different? We can't use `sequence` anymore

```code|lang-ts
import * as A from "fp-ts/lib/Array";
import * as T from "fp-ts/lib/Task";

// Task<number> ----v        v--- Task<string>
const tasks = [T.of(1), T.of("hello")];
A.array.sequence(T.task)(tasks);
/*
Argument of type '(Task<number> | Task<string>)[]' is not assignable to parameter of type 'Task<number>[]'.
  Type 'Task<number> | Task<string>' is not assignable to type 'Task<number>'.
    Type 'Task<string>' is not assignable to type 'Task<number>'.
      Type 'string' is not assignable to type 'number'.ts(2345)
*/
```

However we can use `sequenceT` (or `sequenceS`)

```code|lang-ts
import { sequenceT, sequenceS } from "fp-ts/lib/Apply";
import * as T from "fp-ts/lib/Task";

// x: T.Task<[number, string]>
const x = sequenceT(T.task)(T.of(1), T.of("hello"));

// y: T.Task<{ a: number, b: string }>
const y = sequenceS(T.task)({ a: T.of(1), b: T.of("hello") });
```

## Work with a list of dependent tasks

If you need the result of on task before you can continue with the next, you can `chain` the tasks like so:

```code|lang-ts
import * as T from "fp-ts/lib/Task";
import { pipe } from "fp-ts/lib/pipeable";

pipe(
  T.of(2),
  T.chain(result => T.of(result * 3)),
  T.chain(result => T.of(result + 4))
)().then(console.log); // 10
```

## Traverse: map and sequence

If you have a list of items that you need to `map` over before running them in `sequence`, you can use `traverse`, which is a shortcut for doing both operations in one step.

```code|lang-ts
import * as A from "fp-ts/lib/Array";
import * as T from "fp-ts/lib/Task";
import { access, constants } from "fs";

const checkPathExists = (path: string) => () =>
  new Promise(resolve => {
    access(path, constants.F_OK, err => resolve({ path, exists: !err }));
  });

const items = ["/bin", "/no/real/path"];

A.array
  .traverse(T.task)(items, checkPathExists)()
  .then(console.log); // [ { path: '/bin', exists: true }, { path: '/no/real/path', exists: false } ]
```

## Comparison with `Promise` methods

Following is a table comparing `Task`/`TaskEither` with `Promise`. It assumes the following imports:

```code|lang-ts
import * as T from "fp-ts/lib/Task";
import * as TE from "fp-ts/lib/TaskEither";
import { array } from "fp-ts/lib/Array";
import { fold } from "fp-ts/lib/Monoid";
```

```table
rows:
  - Action: resolve to success
    Promise: Promise.resolve(value)
    Task: T.task.of(value)
    TaskEither: TE.taskEither.of(value) or TE.right(value)
  - Action: resolve to failure
    Promise: Promise.reject(value)
    Task: N/A
    TaskEither: TE.left(value)
  - Action: transform the result of a task with the function "f"
    Promise: promise.then(f)
    Task: T.task.map(task, f)
    TaskEither: T.taskEither.map(taskEither, f)
  - Action: perform a task depending on the result of a previous one
    Promise: promise.then(r => getPromise(r))
    Task: T.task.chain(task, r => getTask(r))
    TaskEither: T.taskEither.chain(taskEither, r => getTaskEither(r))
  - Action: execute an array of tasks in parallel
    Promise: Promise.all(promises)
    Task: array.sequence(T.task)(tasks)
    TaskEither: array.sequence(TE.taskEither)(taskEithers)
  - Action: execute an array of tasks in parallel, collecting all failures and successes
    Promise: Promise.allSettled(promises)
    Task: N/A
    TaskEither: array.sequence(TE.taskEither)(taskEithers)
  - Action: execute an array of tasks and succeed/fail with a single value as soon as one of the tasks succeeds/fails
    Promise: Promise.race(promises)
    Task: fold(T.getRaceMonoid())(tasks)
    TaskEither: fold(T.getRaceMonoid())(taskEithers)
```
