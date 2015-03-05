
# Observable content changes

This JavaScript package, suitable for browsers and Node.js, provides a system
for synchronously observing content changes to arrays, objects, and other
instances.
These observers have a common, composable style, expose their internal state for
debugging, and reuse state tracking objects to reduce garbage collection.

-   Changes can be captured before or after they are made.
-   The last argument of a change notification is the object observed, so a
    single handler can service multiple objects.
-   Handler methods can return a child observer object, which will be implicitly
    cancelled before the next change, so observers can be stacked.

## Installation

```
npm install --save pop-observe@1
```

## Examples

Observing the length of an array.

```js
var array = [];
var observer = observePropertyChange(array, "length", function (length) {
    console.log(length);
});
array.push(10);
observer.cancel();
```

Observing values at indexes.

```js
var array = [];
var handler = {
    handlePropertyChange: function (plus, minus, index, object) {
        console.log("index", index, "was", minus, "became", plus);
    }
};
var observer0 = observePropertyChange(array, 0, handler);
var observer1 = observePropertyChange(array, 1, handler);
var observer2 = observePropertyChange(array, 2, handler);
array.set(0, 10);
array.set(1, 20);
array.set(2, 30);
```

Mirroring arrays.

```js
var swap = require("pop-swap");
var array = [];
var mirror = [];
var observer = observeRangeChange(array, function (plus, minus, index) {
    swap(mirror, index, minus.length, plus);
});
array.push(1, 2, 3);
array.shift();
array.pop();
expect(mirror).toEqual([1]);
observer.cancel();
```

Tracking an array with a plain object.

```js
var array = [];
var object = {};
var observer = observeMapChange(array, function (type, plus, minus, index) {
    if (type === "delete") {
        delete object[index];
    } else { // type === "create" || type === "update"
        object[index] = plus;
    }
};
```

Observing a property of a property.
Note that the cancel method gets rid of observers on a.b and b.c.
Note that the b.c observer gets canceled every time b changes.

```js
var a = {b: {c: 10}};
var observer = observePropertyChange(a, "b", function (b) {
    return observePropertyChange(b, "c", function (c) {
        console.log("a.b.c", c);
    });
});
a.b = {c: 20};
a.b.c = 30;
observer.cancel();
```

## Change notification arguments

-   Property change observers provide (plus, minus, name, object) change
    notifications when the value of a specific, named property changes.
    Each observer sees the old value (minus), new value (plus), property name
    (name), and the object (object) such that a single property change handler
    can service multiple observers.
    Observing a property of an ordinary object replaces that property with a
    getter and setter.
-   Range change observers provide (plus, minus, index, object) change
    notifications when ordered values are removed (captured in a minus array),
    then added (captured in the plus array), at a particular index.
    Each handler also receives the object observed, so a single handler can
    service multiple observers.
-   Map change observers provide (plus, minus, key, object) change notifications
    when the value for a specific key in a map has changed.
  
## Behavior on Arrays

-   Observing any property change on an array transforms that array into an
    observable array and property changes are dispatched for the "length" or any
    value by its index.
-   Observing range changes on an array transforms that array into an observable
    array and all of its methods produce these change notifications.
-   Observing any map change on an array transforms the array into an observable
    array and map changes are dispatched for changes to the value at the given
    index.

## Custom types

-   Arbitrary constructors can mix in or inherit the ObservableObject type
    to support the observable interface directly and do not need to provide any
    further support.
-   Arbitrary constructors can mix in or inherit the ObservableRangeChange type
    and must explicitly dispatch change notifications when range change
    observers are active.
-   This library does not provide any map implementations but provides the
    ObservableMap for any to inherit or mix in.

## Interface

Each type of observer provides before and after methods for observation and
manual dispatch.
For properties, manual dispatch is necessary when a property is hidden behind a
getter and a setter *if* the value as returned by the getter changes without the
setter ever being invoked.
Arrays require manual dispatch only if the value at a given index changes
without invoking an array mutation method.
For this reason, observable arrays have a `set(index, value)` method.
All ranged and map collections must implement manual dispatch when their
`dispatchesRangeChanges` or `dispatchesMapChanges` properties are true.

-   ObservableObject.observePropertyChange(object, handler, note, capture)
-   ObservableObject.observePropertyWillChange(object, handler, note)
-   ObservableObject.dispatchPropertyChange(object, name, plus, minus, capture)
-   ObservableObject.dispatchPropertyWillChange(object, name, plus, minus)
-   ObservableObject.getPropertyChangeObservers(object, name, capture)
-   ObservableObject.getPropertyWillChangeObservers(object, name)
-   ObservableObject.makePropertyObservable(object, name)
-   ObservableObject.preventPropertyObserver(object, name)

-   ObservableObject.prototype.observePropertyChange(handler, note, capture)
-   ObservableObject.prototype.observePropertyWillChange(handler, note)
-   ObservableObject.prototype.dispatchPropertyChange(name, plus, minus, capture)
-   ObservableObject.prototype.dispatchPropertyWillChange(name, plus, minus)
-   ObservableObject.prototype.getPropertyChangeObservers(name, capture)
-   ObservableObject.prototype.getPropertyWillChangeObservers(name)
-   ObservableObject.prototype.makePropertyObservable(name)
-   ObservableObject.prototype.preventPropertyObserver(name)

-   PropertyChangeObserver.prototype.cancel()

-   ObservableRange.prototype.observeRangeChange(handler, name, note, capture)
-   ObservableRange.prototype.observeRangeWillChange(handler, name, note)
-   ObservableRange.prototype.dispatchRangeChange(handler, name, note, capture)
-   ObservableRange.prototype.dispatchRangeWillChange(handler, name, note)
-   ObservableRange.prototype.makeRangeChangesObservable()

-   RangeChangeObserver.prototype.cancel()

-   ObservableMap.prototype.observeMapChange(handler, name, note, capture)
-   ObservableMap.prototype.observeMapWillChange(handler, name, note)
-   ObservableMap.prototype.dispatchMapChange(type, key, plus, minus, capture)
-   ObservableMap.prototype.dispatchMapWillChange(type, key, plus, minus)
-   ObservableMap.prototype.makeMapChangesObservable()

-   MapChangeObserver.prototype.cancel()

## Handlers

Handlers may be raw functions, or objects with one or more handler methods.
Observers for different kinds of changes and before and after changes call
different methods of the handler object based on their availability at the time
that the observer is created.
For example, `observePropertyWillChange(array, "length", handler)` will create a
property observer that will delegate to the
`handler.handleLengthPropertyWillChange(plus, minus, key, object)` method, or
just that generic `handler.handlePropertyWillChange(plus, minus, key, object)`
method if the specific method does not exist.

-   observable object, property observers (plus, minus, key, object)
    -   after change
        -   specific: handleProperty*Name*Change
        -   general: handlePropertyChange
    -   before change
        -   specific: handleProperty*Name*WillChange
        -   general: handlePropertyWillChange

Range changes do not operate on a given property name, but the
`observeRangeChange(handler, name, note, capture)` method allows you to give the
range change a name, for example, the name of the array observed on a given
object.

```js
var handler = {
    handleValuesRangeChange: function (plus, minus, index, object) {
        // ...
    }
};
var observer = repetition.values.observeRangeChange(handler, "values");
// ...
repetition.values.push(10);
// ...
observer.cancel();
```

-   observable range (plus, minus, index, object)
    -   after change
        -   specific: handle*Name*RangeChange
        -   general: handleRangeChange
    -   before change
        -   specific: handle*Name*RangeWillChange
        -   general: handleRangeWillChange

Likewise, `observeMapChange(handler, name, note, capture)` accepts a name for a
specific handler method.

-   observable map (plus, minus, key, object)
    -   after change
        -   specific: handle*Name*MapChange
        -   general: handleMapChange
    -   before change
        -   specific: handle*Name*MapWillChange
        -   general: handleMapWillChange

## Observers

Observers are re-usable objects that capture the state of the observer.
Most importantly, they provide the `cancel` method, which disables the observer
and returns it to a free list for the observer methods to re-use.
They are suitable for run-time inspection of the state of the observer.

They also carry an informational `note` property, if the caller of the observe
method provided one.
This is intended for use by third parties to provide additional debugging
information about an observer, for example, where the observer came from.

-   PropertyChangeObserver
    -   object
    -   propertyName
    -   observers
    -   handler
    -   handlerMethodName
    -   childObserver
    -   note
    -   capture
    -   value
-   RangeChangeObserver
    -   object
    -   name
    -   observers
    -   handler
    -   handlerMethodName
    -   childObserver
    -   note
    -   capture
-   MapChangeObserver
    -   object
    -   name
    -   observers
    -   handler
    -   handlerMethodName
    -   childObserver
    -   note
    -   capture

## Copyright and License

Copyright (c) 2015 Motorola Mobility, Montage Studio, Kristopher Michael Kowal,
and contributors.
All rights reserved.
BSD 3-Clause license.

