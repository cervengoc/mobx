# Autorun

`mobx.autorun` can be used in those cases where you want to create a reactive function that will never have observers itself.
This is usually the case when you need to bridge from reactive to imperative code, for example for logging, persistence or UI-updating code.
When `autorun` is used, the provided function will always be triggered once immediately and then again each time one of its dependencies changes.
In contrast, `computed(function)` creates functions that only re-evaluate if it has
observers on its own, otherwise its value is considered to be irrelevant.
As a rule of thumb: use `autorun` if you have a function that should run automatically but that doesn't result in a new value.
Use `computed` for everything else. Autoruns are about initiating _effects_, not about producing new values.
If a string is passed as first argument to `autorun`, it will be used as debug name.

The function passed to autorun will receive one argument when invoked, the current reaction (autorun), which can be used to dispose the autorun during execution.

Just like the [`@observer` decorator/function](./observer-component.md), `autorun` will only observe data that is used during the execution of the provided function.

```javascript
var numbers = observable([1,2,3]);
var sum = computed(() => numbers.reduce((a, b) => a + b, 0));

var disposer = autorun(() => console.log(sum.get()));
// prints '6'
numbers.push(4);
// prints '10'

disposer();
numbers.push(5);
// won't print anything, nor is `sum` re-evaluated
```


## Caution about dependency detection

The `autorun` mechanism dynamically gathers all of its dependencies during each run. `mobx` will register an observable as a dependency if it's accessed during an execution of the `autorun` function. Because of this fact, take special care to ensure that any dependencies which you mean to have are really accessed when they have to be a dependency. For example avoid using early returns like below before accessing any dependency, because the `autorun` will stop working and will get into a *zombie* state, having no dependencies.

```javascript
// a non-observable variable
let shouldRun = false;
let value = observable(5);

// this autorun will not gather any dependency at first run due to the early return, so it won't run any more
const disposer = autorun(() => {
  if (!shouldRun) return;
  console.log(value.get());
});
```

As a possible solution, ensure that every variable which is a dependency of the `autorun` is observable.

```javascript
// every dependency is observable
let shouldRun = observable(false);
let value = observable(5);

// this autorun will work properly
const disposer = autorun(() => {
  if (!shouldRun.get()) return;
  console.log(value.get());
});
```

If you can't create an observable from `shouldRun`, access the dependency before the early return for example like this.

```javascript
// every dependency is observable
let shouldRun = false;
let value = observable(5);

// this autorun will work properly
const disposer = autorun(() => {
  const somethingToLog = value.get(); // gather dependencies in any case
  if (!shouldRun.get()) return;
  console.log(somethingToLog);
});
```

## Error handling

Exceptions thrown in autorun and all other types reactions are catched and logged to the console, but not propagated back to the original causing code.
This is to make sure that a reaction in one exception does not prevent the scheduled execution of other, possibly unrelated, reactions.
This also allows reactions to recover from exceptions; throwing an exception does not break the tracking done by MobX,
so as subsequent run of a reaction might complete normally again if the cause for the exception is removed.

It is possible to override the default logging behavior of Reactions by calling the `onError` handler on the disposer of the reaction.
Example:

```javascript
const age = observable(10)
const dispose = autorun(() => {
    if (age.get() < 0)
        throw new Error("Age should not be negative")
    console.log("Age", age.get())
})

age.set(18)  // Logs: Age 18
age.set(-10) // Logs "Error in reaction .... Age should not be negative
age.set(5)   // Recovered, logs Age 5

dispose.onError(e => {
    window.alert("Please enter a valid age")
})

age.set(-5)  // Shows alert box
```

A global onError handler can be set as well through `extras.onReactionError(handler)`. This can be useful in tests or for monitoring.
