# JavaScript Debug Tricks

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
