# JavaScript Debug Tricks

## Simple code to log every JavaScript function calls

Below code will simple log every JavaScript function call whenever a function is being executed. For faster debugging just paste the code in your browser console and it will start logging every function call.

```js
(function() {
    var call = Function.prototype.call;
    Function.prototype.call = function() {
        console.log(this, arguments); // Here you can do whatever actions you want
        return call.apply(this, arguments);
    };
}());
```

## Log all method calls inside an object or class

Below code will allow you to trace and log all function calls inside a given object or class. For faster debugging just paste in your browser console.

```js
function traceObject(constructor) {
  Object.keys(constructor).forEach(function (methodName) {
    if (typeof constructor[methodName] !== 'function') {
      return
    }

    var originalMethod = constructor[methodName]
    constructor[methodName] = function () {
      var args = Array.prototype.slice.call(arguments)
      args.unshift('called ' + methodName)
      console.log.apply(console, args)
      return originalMethod.apply(this, arguments)
    }
  })

  var proto = constructor.prototype
  if (proto !== undefined) {
    for (var methodName in proto) {
      var propDesc = Object.getOwnPropertyDescriptor(proto, methodName)
      if (typeof proto[methodName] !== 'function' || (propDesc &&
          (!propDesc.configurable || ('get' in propDesc) || ('set' in propDesc)))) {
        continue
      }

      ;(function (methodName) {
        var originalMethod = proto[methodName]
        proto[methodName] = function () {
          var args = Array.prototype.slice.call(arguments)
          args.unshift('called ' + methodName)
          console.log.apply(console, args)
          return originalMethod.apply(this, arguments)
        }
      })(methodName)
    }
  }
}
```

## Log function calls with function name

Below code log all function calls with function name. This code can be simply copied and pasted in browser console to intercept function calls.

```js
(function() {
    var call = Function.prototype.call;
    Function.prototype.call = function() {
          fn = Object(this)
          var F = typeof fn == 'function'
          var N = fn.name
          var S = F && ((N && ['', N]) || fn.toString().match(/function ([^\(]+)/))
          const name = (!F && 'not a function') || (S && S[1] || 'anonymous');
          console.log(name, arguments, this); // Here you can do whatever actions you want
          return call.apply(this, arguments);
    };
}());
```

