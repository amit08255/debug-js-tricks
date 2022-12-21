# JavaScript Debug Tricks

## Get intercepted version of function

```js
function intercept(f, cb) {
  return function() {
    cb(arguments);
    f.apply(f, Array.prototype.slice.call(arguments, 0));
  };
}
```

## Detect if a function is native JavaScript function

Detects if function is native to JavaScript and not defined by user or external library.

```js
function isNativeFn(fn) {
    try {
        void new Function(fn.toString());    
    } catch (e) {
        return true;
    }
    return false;
}
```

A better solution for this is:

```js
  // Used to resolve the internal `[[Class]]` of values
  var toString = Object.prototype.toString;
  
  // Used to resolve the decompiled source of functions
  var fnToString = Function.prototype.toString;
  
  // Used to detect host constructors (Safari > 4; really typed array specific)
  var reHostCtor = /^\[object .+?Constructor\]$/;

  // Compile a regexp using a common native method as a template.
  // We chose `Object#toString` because there's a good chance it is not being mucked with.
  var reNative = RegExp('^' +
    // Coerce `Object#toString` to a string
    String(toString)
    // Escape any special regexp characters
    .replace(/[.*+?^${}()|[\]\/\\]/g, '\\$&')
    // Replace mentions of `toString` with `.*?` to keep the template generic.
    // Replace thing like `for ...` to support environments like Rhino which add extra info
    // such as method arity.
    .replace(/toString|(function).*?(?=\\\()| for .+?(?=\\\])/g, '$1.*?') + '$'
  );
  
  function isNative(value) {
    var type = typeof value;
    return type == 'function'
      // Use `Function#toString` to bypass the value's own `toString` method
      // and avoid being faked out.
      ? reNative.test(fnToString.call(value))
      // Fallback to a host object check because some environments will represent
      // things like typed arrays as DOM methods which may not conform to the
      // normal native pattern.
      : (value && type == 'object' && reHostCtor.test(toString.call(value))) || false;
  }
```

## Get current stack trace as array string

```js
function getStackTrace () {

  var stack;

  try {
    throw new Error('');
  }
  catch (error) {
    stack = error.stack || '';
  }

  stack = stack.split('\n').map(function (line) { return line.trim(); });
  return stack.splice(stack[0] == 'Error' ? 2 : 1);
}
```

**V2 to ignore stack trace from node_modules:**

```js
function getStackTrace() {
    let stack;

    try {
        throw new Error('');
    } catch (error) {
        stack = error.stack || '';
    }

    stack = stack.split('\n').map((line) => line.trim());
    stack = stack.splice(stack[0] === 'Error' ? 2 : 1);
    return stack.filter((x) => (/\/node_modules\//.test(x) !== true));
}
```

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
          var fn = Object(this)
          var F = typeof fn == 'function'
          var N = fn.name
          var S = F && ((N && ['', N]) || fn.toString().match(/function ([^\(]+)/))
          const name = (!F && 'not a function') || (S && S[1] || 'anonymous');
          console.log(name, arguments, this); // Here you can do whatever actions you want
          return call.apply(this, arguments);
    };
}());
```

## Ignore function in blacklist and log function calls with function name

Below code log all function calls with function name using global whitelist and blacklist. Adding `*` in blacklist will block all except those in whitelist. This code can be simply copied and pasted in browser console to intercept function calls.

```js
(function() {
    window.functionBlackList = { hasOwnProperty: true };
    window.functionWhiteList = {};
    var call = Function.prototype.call;
    Function.prototype.call = function() {
          fn = Object(this)
          var F = typeof fn == 'function'
          var N = fn.name
          var S = F && ((N && ['', N]) || fn.toString().match(/function ([^\(]+)/))
          const name = (!F && 'not a function') || (S && S[1] || 'anonymous');
          
          if (window.functionBlackList['*'] && window.functionWhiteList[name]) {
            console.log(name, arguments, this); // Here you can do whatever actions you want
          } else if (!window.functionBlackList[name]) {
            console.log(name, arguments, this); // Here you can do whatever actions you want
          }

          return call.apply(this, arguments);
    };
}());
```

## Log non-native function calls (best for tracking function calls)

```js
(function () {
    const { call } = Function.prototype;
    Function.prototype.call = function () {
        let fn = Object(this);
        try {
            void new Function(fn.toString());
        } catch (e) {
            return call.apply(this, arguments);
        }

        const F = typeof fn === 'function';
        const N = fn.name;
        const S = F && ((N && ['', N]) || fn.toString().match(/function ([^\(]+)/));
        const name = (!F && 'not a function') || (S && S[1] || 'anonymous');
        console.log("\n", name, arguments); // Here you can do whatever actions you want
        return call.apply(this, arguments);
    };
}());
```

**v2 of code to ignore functions with name starting with underscore:**

```js
(function () {
    const { call } = Function.prototype;
    Function.prototype.call = function () {
        let fn = Object(this);
        try {
            void new Function(fn.toString());
        } catch (e) {
            return call.apply(this, arguments);
        }

        const F = typeof fn === 'function';
        const N = fn.name;
        const S = F && ((N && ['', N]) || fn.toString().match(/function ([^\(]+)/));
        const name = (!F && 'not a function') || (S && S[1] || 'anonymous');

        // Do not log function calls whose name start with underscore
        if (name[0] !== '_') {
            console.log("\n", name, arguments); // Here you can do whatever actions you want
        }

        return call.apply(this, arguments);
    };
}());
```

## Intercept all addEventListener for debugging

```js
(function() {
    Error.stackTraceLimit = Infinity;
    
    var _interfaces = Object.getOwnPropertyNames(window).filter(function(i) {
      return /^HTML/.test(i);
    }).map(function(i) {
      return window[i];
    });

    // var _interfaces = [ HTMLDivElement, HTMLImageElement, HTMLUListElement, HTMLElement, HTMLDocument ];
    for (var i = 0; i < _interfaces.length; i++) {
      (function(original) {
        _interfaces[i].prototype.addEventListener = function(type, listener, useCapture) {
          console.log('addEventListener ' + type, listener, useCapture);
          console.trace();
          console.log('--------');

          return original.apply(this, arguments);
        }
      })(_interfaces[i].prototype.addEventListener);
    }
})();
```

## Log ReactJS function component hooks execution stack trace

Add below code at top of file just after all imports are added.

```js
import React from 'react';

function getStackTrace() {
    let stack;

    try {
        throw new Error('');
    } catch (error) {
        stack = error.stack || '';
    }

    stack = stack.split('\n').map((line) => line.trim());
    stack = stack.splice(stack[0] === 'Error' ? 2 : 1);
    return stack.filter((x) => (/\/node_modules\//.test(x) !== true));
}

const reactInterceptor = (name, originalFunction) => {
    let lastMessage = null;

    return (...args) => {
        const trace = getStackTrace();
        const msg = trace.join('\n');

        if (lastMessage !== msg) {
            console.log(`React.${name} called`);
            console.log(msg);
            console.log('===============');
            lastMessage = msg;
        }

        return originalFunction(...args);
    };
};

React.useEffect = reactInterceptor('useEffect', React.useEffect);
React.useState = reactInterceptor('useState', React.useState);
```

**Version 2 to log arguments and filter with stack trace of length 1 i.e. not execution track from app:**

```js
import React from 'react';

function getStackTrace() {
    let stack;

    try {
        throw new Error('');
    } catch (error) {
        stack = error.stack || '';
    }

    stack = stack.split('\n').map((line) => line.trim());
    stack = stack.splice(stack[0] === 'Error' ? 2 : 1);
    return stack.filter((x) => (/\/node_modules\//.test(x) !== true));
}

const reactInterceptor = (name, originalFunction) => {
    let lastMessage = null;

    return (...args) => {
        const trace = getStackTrace();
        const msg = trace.join('\n');

        if (lastMessage !== msg && trace.length > 1) {
            console.log(`React.${name} called`);
            console.log(args);
            console.log('- - - - - -');
            console.log(msg);
            console.log('===============');
            lastMessage = msg;
        }

        return originalFunction(...args);
    };
};

React.useEffect = reactInterceptor('useEffect', React.useEffect);
React.useState = reactInterceptor('useState', React.useState);
```

**Version 3 to remove the interceptor function from stack trace:**

```js
import React from 'react';

function getStackTrace() {
    let stack;

    try {
        throw new Error('');
    } catch (error) {
        stack = error.stack || '';
    }

    stack = stack.split('\n').map((line) => line.trim());
    stack = stack.splice(stack[0] === 'Error' ? 2 : 1);

    const result = [];

    stack.map((x, index) => {
        if (index > 0 && /\/node_modules\//.test(x) !== true) {
            result.push(x);
        }
        return null;
    });

    return result;
}

const reactInterceptor = (name, originalFunction) => {
    let lastMessage = null;

    return (...args) => {
        const trace = getStackTrace();
        const msg = trace.join('\n');

        if (lastMessage !== msg && trace.length > 1) {
            console.log(`React.${name} called`);
            console.log(args);
            console.log('- - - - - -');
            console.log(msg);
            console.log('===============');
            lastMessage = msg;
        }

        return originalFunction(...args);
    };
};

React.useEffect = reactInterceptor('useEffect', React.useEffect);
React.useState = reactInterceptor('useState', React.useState);
```

**Version 4 React interceptor with events intercepting:**

The event interceptor has optional third parameter whose value is name events to intercept. By default it has `*` added which means intercept every event occurred:

```js
import React from 'react';

function getStackTrace() {
    let stack;

    try {
        throw new Error('');
    } catch (error) {
        stack = error.stack || '';
    }

    stack = stack.split('\n').map((line) => line.trim());
    stack = stack.splice(stack[0] === 'Error' ? 2 : 1);

    const result = [];

    stack.map((x, index) => {
        if (index > 0 && /\/node_modules\//.test(x) !== true) {
            result.push(x);
        }
        return null;
    });

    return result;
}

const reactInterceptor = (name, originalFunction) => {
    let lastMessage = null;

    return (...args) => {
        const trace = getStackTrace();
        const msg = trace.join('\n');

        if (lastMessage !== msg && trace.length > 1) {
            console.log(`React.${name} called`);
            console.log(args);
            console.log('- - - - - -');
            console.log(msg);
            console.log('===============');
            lastMessage = msg;
        }

        return originalFunction(...args);
    };
};

const eventInterceptor = (name, originalFunction) => (...args) => {
    const trace = getStackTrace();
    const msg = trace.join('\n');
    console.log(`event ${name} called`);
    console.log(args);
    console.log('- - - - - -');
    console.log(msg);
    console.log('===============');

    return originalFunction(...args);
};

const reactCreateElementInterceptor = (name, originalFunction, events = { '*': true }) => (...args) => {
    if (args.length >= 2 && typeof args[0] === 'string') {
        if (args[1] !== undefined && args[1] !== null && typeof args[1] === 'object') {
            Object.keys(args[1]).forEach((key) => {
                if ((events['*'] || events[key]) && key.startsWith('on') && typeof args[1][key] === 'function') {
                    args[1][key] = eventInterceptor(`${args[0]}.${key}`, args[1][key]);
                }
            });
        }
    }

    return originalFunction(...args);
};

React.useEffect = reactInterceptor('useEffect', React.useEffect);
React.useState = reactInterceptor('useState', React.useState);
React.createElement = reactCreateElementInterceptor('createElement', React.createElement, { onClick: true });
```

## Log HTTP requests

For faster debugging just paste the code in your browser console and it will start logging every HTTP call.

```js
(function(XHR) {
    "use strict";
    var open = XHR.prototype.open;
    var send = XHR.prototype.send;
    
    XHR.prototype.open = function(method, url, async, user, pass) {
        this._url = url;
        console.log('Opening request: ', method, url, this);
        open.call(this, method, url, async, user, pass);
    };
    
    XHR.prototype.send = function(data) {
        var self = this;
        var oldOnReadyStateChange;
        var url = this._url;
        
        function onReadyStateChange() {
            // log request when finished
            if (self.readyState === 4) {
                console.log(url, this._url, this.response);
            }

            if(oldOnReadyStateChange) {
                oldOnReadyStateChange();
            }
        }
        
        oldOnReadyStateChange = this.onreadystatechange; 
        this.onreadystatechange = onReadyStateChange;
        
        send.call(this, data);
    }
})(XMLHttpRequest);
```

# Log HTTP requests with log downloads

For faster debugging just paste the code in your browser console and it will start logging every HTTP call.
Call `window.exportLogsFile();` function in console to download logs file.

```js
function downloadJSON(objArr) {
    //Convert JSON Array to string.
    let json = JSON.stringify(objArr, null, 4);
    //Convert JSON string to BLOB.
    json = [json];
    let blob1 = new Blob(json, { type: "text/plain;charset=utf-8" });

    //Check the Browser.
    let isIE = false || !!document.documentMode;
    if (isIE) {
        window.navigator.msSaveBlob(blob1, "logs.txt");
    } else {
        let url = window.URL || window.webkitURL;
        let link = url.createObjectURL(blob1);
        let a = document.createElement("a");
        a.download = "logs.txt";
        a.href = link;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
    }
}

(function(XHR) {
    "use strict";
    const logs = [];
    var open = XHR.prototype.open;
    var send = XHR.prototype.send;

    window.exportLogsFile = function(){
        downloadJSON(logs);
    }
    
    XHR.prototype.open = function(method, url, async, user, pass) {
        this._url = url;
        console.log('Opening request: ', method, url, this);
        open.call(this, method, url, async, user, pass);
    };
    
    XHR.prototype.send = function(data) {
        var self = this;
        var oldOnReadyStateChange;
        var url = this._url;
        
        function onReadyStateChange() {
            // log request when finished
            if (self.readyState === 4) {
                logs.push({
                    url: this._url,
                    response: this.response,
                    status: this.status,
                })
            }

            if(oldOnReadyStateChange) {
                oldOnReadyStateChange();
            }
        }
        
        oldOnReadyStateChange = this.onreadystatechange; 
        this.onreadystatechange = onReadyStateChange;
        
        send.call(this, data);
    }
})(XMLHttpRequest);
```
