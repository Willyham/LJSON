## LJSON 

LJSON is a drop-in replacement for [JSON](http://www.json.org) which also allows you to parse and stringify pure functions and their contents. There are good security reasons for functions to be out of the JSON specs, but most of those are only significant when you allow arbitrary, side-effective programs. With pure functions, one is able to interchange code while still being as safe as with regular JSON.

```JavaScript
var LJSON = require("./LJSON.js");

// A random JS object with a pure function inside.
var person = {
    name : "John",
    mail : function(msg){ return { author  : "John", message : msg}; }
};

// If JSON was used, the `mail` field would be stripped from `personVal`.
var personStr = LJSON.stringify(person); 
var personVal = LJSON.parse(personStr);
var mail      = personVal.mail("hello"); // would crash with JSON

// But, since `mail` is pure, LJSON can deal with it correctly:
console.log("Serialized value : " + personStr);
console.log("Calling mail     : " + LJSON.stringify(mail));
```

Output:

```JavaScript
Serialized value : {"name":"John","mail":(v0)=>({"author":"John","message":v0})}
Calling mail     : {"author":"John","message":"hello"}
```

See this and more examples on the [`Examples`](https://github.com/MaiaVictor/LJSON/tree/master/Examples) directory.

## More info

- [Installing](#installing)
- [Other languages](#other-languages)
- [Why?](#why?)
- [Using primitives](#using-primitives)
- [Safety](#safety)
- [TODO](#todo)

## Installing

    npm install ljson

Or just download `LJSON.js` and `parsinhora.js` and import directly.

## Other languages

Currently, there is a port to [PHP](https://github.com/Kanti/LJSON) kindly made by [Kanti](https://github.com/Kanti).

## Why?

Other than convenience, there are times when you simply can't avoid running user code. For example, if a feature in your online game requires players to define scripts to control their ingame characters, you could implement it by receiving their code as strings, and using `eval`:

```javascript
// client-side
script = ["function playerScript(player){",
    "if (player.targetInRange('poring'))",
        "player.castSpell('fire bolt',player.findTarget('poring'));",
"}"].join("\n");
server.send(script);

// server-side:
player.onSend("script",function(script){
    player.installScript(eval(script)); // what could go wrong
});
```

Except that is probably the worst idea ever. Trusting user defined code is a security-person's worst nightmare. Workarounds include sandboxes and defining your own safe DSL - but those solutions can be overkill. There is a simpler way: pure functions. Instead of starting with power to do anything (arbitrary functions) and struggling to control it, you start with no power at all (pure functions) and add only the right primitives to do what your app requires. The code above could be rewritten as:

```javascript
// client-side
function playerScript($,player){
    return $("if", $("targetInRange", player, "poring"),
        $("castSpell", player, "fire bolt", $("findTarget", player, "poring")));
};
var script = LJSON.stringify(playerScript);
server.send(script);

// server-side:
player.onSend("script", function(script){
    player.installScript(LJSON.parseWithLib(safeLib, script)); // not so bad
});
```

Where the `$` is an environment with the set of primitives your players are allowed to use, including things such as math operators, flow control and in-game commands. Of course, that lispy-like code isn't nearly as good looking as the former version, but is completely safe and pure. Functions defined that way can be stringified, communicated and parsed securely - just like JSON.

## Using primitives

LJSON defines functions and function application only - no primitives such as numeric addition. So, for example, this is undefined behavior:

    LJSON.stringify(function(x){ return x+1; });

Because the `+1` bit isn't defined on LJSON. To actually do things with JS numbers, arrays, etc., you need to enable the proper primitives. You can do that either manually or by using LJSON's primitive decorators:

    withLib(lib, fn)
    withStdLib(fn)
    parseWithLib(lib, source)
    parseWithStdLib(source)

`withLib` uses the first argument of a pure function as a way to access the primitives defined on the `lib` object. For example:

```javascript
var lib   = {triple:function(x){return x*3}};
nineTimes = LJSON.withLib(lib, function ($, x){
    return $("triple", $("triple",x));
});
console.log(nineTimes(9));
```

Here, `$` can be understood as "apply function from environment". Since our environment only defines one function, `triple`, that's the only thing `nineTimes` can do. That is, it could multiply a number by 3, by 9, by 27, by 81, etc. - but it couldn't multiply a number by 2. That's how restricted your environment is! Of course, defining your own environment would be cumbersome if you just want to use JS's common functions. For that, there is `LJSON.withStdLib`, which enables an standard environment with most common (pure/safe) functions such as math operators and strings:

```javascript
hypotenuse = function($,a,b){
    return $("sqrt",$("+",$("*",a,a),$("*",b,b)));
};
var hypotenuseStr = LJSON.stringify(hypotenuse);
var hypotenuseVal = LJSON.parseWithStdLib(hypotenuseStr);
console.log(hypotenuseVal(3,4)); // output: 5
```

Remember you have to enable a lib **after** stringifying, communicating/storing and parsing the function. It is the last step. After you call `withStdLib`, the function gains access to primitives outside of the LJSON specs, so `LJSON.stringify` will not work on it anymore.

## Safety

The fact you have to explicitly provide primitives to LJSON functions is what gives you confidence they won't do any nasty thing such as stealing your password, mining bitcoins or launching missiles. LJSON functions can only do what you give them power to. You are still able to serialize side-effective functions, but the side effects will happen on the act of the serialization and get stripped from the serialized output.

```JavaScript
function nastyPair(a,b){
    console.log("booom");
    return { 
        fst : a, 
        snd : (function nastyId(x){
            for (var i=0; i<3; ++i)
                console.log("mwahahhahha");
            return x;
        })(b)};
};
console.log(LJSON.stringify(nastyPair));

// output: 
// booom
// mwahahhahha
// mwahahhahha
// mwahahhahha
// (v0,v1)=>({fst:v0,snd:v1})
```

As a cool side effect of this, you can actually use JS primitives inside functions - as long as they can be eliminated at compile time. In other words, `LJSON.stringify` also works very well as a λ-calculator (due to JS engines speed):

```javascript
console.log(LJSON.stringify(function(a){
    // Things known at compile time are evaluated.
    var arr = [];
    for (var i=0; i<10; ++i)
        arr.push(i*10);

    // Even inside functions.
    var foo = function(x){
        if (arr[5] < 10)
            var value = "I'm never returned";
        else
            var value = "I'm always returned";
        return value;
    };

    // Even λ-calculus expressions!
    var C3  = (f)=>(x)=>f(f(f(x)));
    var C27 = C3(C3); // church number exponentiation of 3^3

    return [
        arr,
        foo,
        C27];
}));
```

That outputs:

```JavaScript
(v0)=>([
    [0,10,20,30,40,50,60,70,80,90],
    (v1)=>("I'm always returned"),
    (v2)=>((v3)=>(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v2(v3)))))))))))))))))))))))))))))])
```
