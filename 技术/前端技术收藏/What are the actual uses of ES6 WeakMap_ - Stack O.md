What are the actual uses of ES6 WeakMap? - Stack Overflow

## Fundamentally

**WeakMaps provide a way to extend objects from the outside without interfering with garbage collection.** Whenever you want to extend an object but can't because it is sealed - or from an external source - a WeakMap can be applied.

A WeakMap is a map (dictionary) where the **keys** are weak - that is, if all references to the *key* are lost and there are no more references to the value - the *value* can be garbage collected. Let's show this first through examples, then explain it a bit and finally finish with real use.

Let's say I'm using an API that gives me a certain object:

```
var obj = getObjectFromLibrary();
```

Now, I have a method that uses the object:

```
function useObj(obj){
   doSomethingWith(obj);
}
```

I want to keep track of how many times the method was called with a certain object and report if it happens more than N times. Naively one would think to use a Map:

```
var map = new Map(); // maps can have object keys
function useObj(obj){
    doSomethingWith(obj);
    var called = map.get(obj) || 0;
    called++; // called one more time
    if(called > 10) report(); // Report called more than 10 times
    map.set(obj, called);
}
```

This works, but it has a memory leak - we now keep track of every single library object passed to the function which keeps the library objects from ever being garbage collected. Instead - we can use a `WeakMap`:

```
var map = new WeakMap(); // create a weak map
function useObj(obj){
    doSomethingWith(obj);
    var called = map.get(obj) || 0;
    called++; // called one more time
    if(called > 10) report(); // Report called more than 10 times
    map.set(obj, called);
}
```

And the memory leak is gone.

## Use cases

Some use cases that would otherwise cause a memory leak and are enabled by `WeakMap`s include:

- Keeping private data about a specific object and only giving access to it to people with a reference to the Map. A more ad-hoc approach is coming with the private-symbols proposal but that's a long time from now.
- Keeping data about library objects without changing them or incurring overhead.
- Keeping data about a small set of objects where many objects of the type exist to not incur problems with hidden classes JS engines use for objects of the same type.
- Keeping data about host objects like DOM nodes in the browser.
- Adding a capability to an object from the outside (like the event emitter example in the other answer).

## Let's look at a real use

It can be used to extend an object from the outside. Let's give a practical (adapted, sort of real - to make a point) example from the real world of Node.js.

Let's say you're Node.js and you have `Promise` objects - now you want to keep track of all the currently rejected promises - however, you do *not* want to keep them from being garbage collected in case no references exist to them.

Now, you *don't* want to add properties to native objects for obvious reasons - so you're stuck. If you keep references to the promises you're causing a memory leak since no garbage collection can happen. If you don't keep references then you can't save additional information about individual promises. Any scheme that involves saving the ID of a promise inherently means you need a reference to it.

### Enter WeakMaps

WeakMaps mean that the **keys** are weak. There are no ways to enumerate a weak map or to get all its values. In a weak map, you can store the data based on a key and when the key gets garbage collected so do the values.

This means that given a promise you can store state about it - and that object can still be garbage collected. Later on, if you get a reference to an object you can check if you have any state relating to it and report it.

This was used to implement [unhandled rejection hooks](https://github.com/iojs/io.js/issues/256) by Petka Antonov as [this](https://iojs.org/api/process.html#process_event_unhandledrejection):

```
process.on('unhandledRejection', function(reason, p) {
    console.log("Unhandled Rejection at: Promise ", p, " reason: ", reason);
    // application specific logging, throwing an error, or other logic here
});
```

We keep information about promises in a map and can know when a rejected promise was handled.