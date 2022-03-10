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
