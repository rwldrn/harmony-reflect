# Proxy Handler API

This API documents all the functions that a proxy handler object may implement to trap operations on a proxy object. These functions are also knowns as *traps*. All of these functions correspond one-to-one to a function with the same name and signature in the [Reflect API](api.md). If a trap wants to perform the "default behavior" for an intercepted operation, it can simply call that corresponding function from the handler API.

As an example, the below proxy logs all `get` and `set` operations applied to it, then forwards the operations to its target object:

    var proxy = Proxy({}, {
      // intercepts proxy[name]
      get: function(target, name, receiver) {
        console.log('get',name);
        return Reflect.get(target, name, receiver);
      },
      // intercepts proxy[name] = value
      set: function(target, name, value, receiver) {
        console.log('set',name,value);
        return Reflect.set(target, name, value, receiver);
      }
    });
    
All traps are *optional*: if missing, the proxy will *automatically* apply the intercepted operation to its `target` object.

## get(target, name, receiver)

Called when the proxy's `name` property is accessed. `receiver` denotes the `this`-binding for the getter function, in case `name` is an accessor property.

This trap should return the value of the property, or `undefined` if the property does not exist.

`receiver` may be bound to the proxy object itself, so be careful when you touch the object inside the trap: this can easily lead to infinite recursion.

This trap intercepts the following operations:

  *  `proxy[name]`
  *  `Object.create(proxy)[name]` (i.e. if a proxy is used as a prototype, and the child does not override the property, the proxy's `get` trap may be triggered. In this case, `receiver` is bound to the child object.)
  *  `Reflect.get(proxy,name,receiver)`

## set(target, name, value, receiver)

Called when the proxy's `name` property is assigned to `value`. `receiver` denotes the `this`-binding for the setter function, in case `name` is an accessor property.

This trap should return a boolean indicating whether or not the update happened successfully.

`receiver` may be bound to the proxy object itself, so be careful when you touch the object inside the trap: this can easily lead to infinite recursion.

This trap intercepts the following operations:

  *  `proxy[name] = value`
  *  `Object.create(proxy)[name] = value` (i.e. if a proxy is used as a prototype, and the child does not override the property, the proxy's `set` trap may be triggered. In this case, `receiver` is bound to the child object.)
  *  `Reflect.set(proxy,name,value,receiver)`

## has(target, name)

Called when the proxy is queried for an own or inherited property named `name`.
This trap should return a boolean indicating whether the proxy has an own or inherited property called `name`.

This trap intercepts the following operations:

  *  `name in proxy`
  *  `name in Object.create(proxy)` (i.e. if a proxy is used as a prototype, its `has` trap is triggered if any of its child objects do not have the property)
  *  `Reflect.has(proxy,name)`

## hasOwn(target, name)

Called when the proxy is queried for an own property named `name`.
This trap should return a boolean indicating whether the proxy has an own (not inherited) property called `name`.

This trap intercepts the following operations:

  *  `Object.prototype.hasOwnProperty.call(proxy, name)`
  *  `Reflect.hasOwn(proxy,name)`

## keys(target)

Called when the proxy is queried for its own enumerable property names.
This trap should return an array of strings.

This trap intercepts the following operations:

  *  `Object.keys(proxy)`
  *  `Reflect.keys(proxy)`

## apply(target, receiver, args)

Called when the proxy is applied as a function with `receiver` as its `this`-binding and `args` as the array of actual arguments. This trap can return anything.

This trap is "active" _only_ if `typeof target === "function"`. That is, if the target object is not callable, then calling the proxy will throw a TypeError rather than calling the trap.

`receiver` may be bound to the proxy object itself, so be careful when you touch the object inside the trap: this can easily lead to infinite recursion.

This trap intercepts the following operations:

  * `proxy(...args)` (in this case, `receiver` is bound to `undefined`)
  * `Function.prototype.apply.call(proxy, receiver, args)`
  * `Function.prototype.call.call(proxy, receiver, ...args)`
  * `Reflect.apply(proxy,receiver,args)`

## construct(target, args)

Called when a proxy is treated as a constructor function to create a new instance object.
This trap can return anything.

This trap is "active" _only_ if `typeof target === "function"`. That is, if the target object is not callable, then calling `new` on a proxy will throw a TypeError rather than calling this trap.

This trap intercepts the following operations:

  *  `new proxy(...args)`
  *  `Reflect.construct(proxy,args)`

## getOwnPropertyDescriptor(target, name)

Called when the proxy object is queried for an own property descriptor.
This trap should return either a property descriptor object or `undefined`.

`target` is the proxy's target object, `name` is a string.

This trap intercepts the following operations:

  *  `Object.getOwnPropertyDescriptor(proxy, name)`
  *  `Reflect.getOwnPropertyDescriptor(proxy, name)`

The proxy throws a TypeError if:

  *  This trap returns `undefined`, but the `name` property of `target` is non-configurable. If a target property is non-configurable, a proxy cannot hide it.
  *  This trap returns a property descriptor that is not compatible with the corresponding property in `target` (e.g. `target[name]` is non-configurable and this trap returns a configurable descriptor).
  *  This trap returns a non-configurable property that doesn't exist on `target`. A non-configurable property can only be exposed if the `target` object has a corresponding property.
  
Examples:

    var target = { x: 0 };
    var handler = {
      getOwnPropertyDescriptor: function(target, name) {
        if (name === "y") {
          return {value: 1, configurable: true};
        } else {
          return Reflect.getOwnPropertyDescriptor(target, name);
        }
      }
    };
    var proxy = Proxy(target, handler);
    Object.getOwnPropertyDescriptor(proxy, "x")
      // calls handler.getOwnPropertyDescriptor(target, "x")
      // returns {value: 0, writable: true, configurable: true, enumerable: true}
    Object.getOwnPropertyDescriptor(proxy, "y")
      // calls handler.getOwnPropertyDescriptor(target, "y")
      // returns {value: 1, writable: false, configurable: true, enumerable: false}
    Object.getOwnPropertyDescriptor(proxy, "z")
      // calls handler.getOwnPropertyDescriptor(target, "z")
      // returns undefined

    Object.defineProperty(target, "y", {value:2,configurable:false});
    Object.getOwnPropertyDescriptor(proxy, "y")
      // calls handler.getOwnPropertyDescriptor(target, "y")
      // throws TypeError: trap returns an incompatible descriptor
    

## defineProperty(target, name, desc)

Called when a new property is defined on the proxy object.
This trap should return a boolean to indicate whether or not the definition succeeded.

`target` is the proxy's target object, `name` is a string, `desc` is a property descriptor object. Engines may "normalize" descriptors so that the `desc` argument is bound to a fresh property descriptor object instead of the original object that the client passed to `Object.defineProperty`.

This trap intercepts the following operations:

  *  `Object.defineProperty(proxy, name, desc)`
  *  `Reflect.defineProperty(proxy, name, desc)`

The proxy throws a TypeError if:

  *  This trap returns `true` while `Object.defineProperty(target, name, desc)` would throw. One cannot successfully define incompatible descriptors.
  * This trap returns `true`, `desc` is non-configurable and `target` has no `name` property. A non-configurable property can only be defined successfully if the `target` object has a corresponding property.

## getOwnPropertyNames(target)

Called when the proxy is queried for all of its own (i.e. not inherited) property names.
This trap should return an array of strings.

This trap intercepts the following operations:

  *  `Object.getOwnPropertyNames(proxy)`
  *  `Reflect.getOwnPropertyNames(proxy)`

The proxy throws a TypeError if:

  *  The `target` has a non-configurable property that is not listed in the result. Proxies cannot hide non-configurable properties.
  *  The result contains new property names that do not appear in `target` and  `Object.isExtensible(target)` is false. If the target is non-extensible, a proxy cannot report new non-existent properties.

## deleteProperty(target, name)

Called when a property is deleted on the proxy.
This trap should return a boolean that indicates whether or not the deletion was successful.

`name` is a string.

This trap intercepts the following operations:

  *  `delete proxy[name]`
  *  `Reflect.deleteProperty(proxy, name)`

## enumerate(target)

Called when the proxy is queried for its enumerable own and inherited properties.
This trap should return an array of strings.

This trap intercepts the following operations:

  *  `for (var name in proxy) {...}`
  *  `for (var name in Object.create(proxy)) {...}` (i.e. a `for-in` loop that encounters a proxy in the prototype chain)
  *  `Reflect.enumerate(proxy)`

## iterate(target)

Called when the proxy's properties are iterated over using a `for-of` loop.
This trap should return an iterator, which is an object with a `next()` method.

This trap intercepts the following operations:

  *  `for (var elem of proxy) {...}`. The `for-of` loop is new in ECMAScript 6. See the [MDN docs](https://developer.mozilla.org/en/JavaScript/Reference/Statements/for...of).
  *  `Reflect.iterate(proxy)`

[Iterators](https://developer.mozilla.org/en/JavaScript/Guide/Iterators_and_Generators) are also new in ES6 and iterate over the elements as if by executing:

    var iterator = handler.iterator(target);
    try {
      while (true) {
        var elem = iterator.next();
        ...
      }
    } catch (e) {
      if (e !== StopIteration) throw e;
    }
    

## freeze(target)

Called when an attempt is made to freeze the proxy object.
This trap should return a boolean indicating whether the proxy was successfully frozen.

This trap intercepts the following operations:

  *  `Object.freeze(proxy)`
  *  `Reflect.freeze(proxy)`

## seal(target)

Called when an attempt is made to seal the proxy object.
This trap should return a boolean indicating whether the proxy was successfully sealed.

This trap intercepts the following operations:

  *  `Object.seal(proxy)`
  *  `Reflect.seal(proxy)`

## preventExtensions(target)

Called when an attempt is made to make the proxy object non-extensible.
This trap should return a boolean indicating whether the proxy was successfully made non-extensible.

This trap intercepts the following operations:

  *  `Object.preventExtensions(proxy)`
  *  `Reflect.preventExtensions(proxy)`