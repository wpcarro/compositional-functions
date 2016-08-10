# Composition Functions

Asynchronous programming is clearly a pain point for JavaScript developers. The async/await syntax would be a valuable addition to the JavaScript language. However the async/await syntax can only emit a Promise, which is one of any number of different asynchronous primitives used in JavaScript programs.

Other asynchronous primitives include:

1. Observable
2. Task

Some of these asynchronous primitives have valuable semantics which are a better fit for certain APIs. One such semantic is the ability to signal to the producing function that that a value is no longer required. When no observers exist for an value that will eventually be asynchronously produced, the process of producing that value may be able to be cancelled.

The current async/await syntax can only produce Promises, ceeding valuable syntactic space that will not be available to compose other asynchronous primitives. It also implies that the natural primitives for async programming is Promise, but that may not be the case in other systems.

A better approach is to add _extensible syntax_ to the language that can be extended by the community to compose any asynchronous primitive. This is the goal of the Compositional Function proposal.

## Promise Composition with a Composition Function

Here is an example of a composition function used to produce a Promise:

```JavaScript

import async from 'async'

async function getStockPriceByName(name) {
    var symbol = await getStockSymbol(name);
    var stockPrice = await getStockPrice(symbol);
    return stockPrice;
}
```

The code above desugars to this:

```JavaScript
import async from 'async'

function getStockPriceByName(name) {
    return async(function*(name) {
        var symbol = yield getStockSymbol(name);
        var stockPrice = yield getStockPrice(symbol);
        return stockPrice;
    });
}
```

Here is the definition of the Composition Function:

```JavaScript
function async(genF) {
    return new Promise(function(resolve, reject) {
        var gen = genF();
        function step(nextF) {
            var next;
            try {
                next = nextF();
            } catch(e) {
                // finished with failure, reject the promise
                reject(e); 
                return;
            }
            if(next.done) {
                // finished with success, resolve the promise
                resolve(next.value);
                return;
            } 
            // not finished, chain off the yielded promise and `step` again
            Promise.resolve(next.value).then(function(v) {
                step(function() { return gen.next(v); });      
            }, function(e) {
                step(function() { return gen.throw(e); });
            });
        }
        step(function() { return gen.next(undefined); });
    });    
}
```

## Task Composition with a Composition Function

Composition Functions can be applied to other asynchronous Primitives. Let's define an asynchronous Task primitive. The Task primitive represents the task of creating an asynchronous result.

Let's create a constructor for Task.

```JavaScript
function Task(get) {
    var self = this;

    this.get = function(valueFn, throwFn) {
        // send memoized error
        if (self.error) {
            throwFn(self.error);
        }
        // send memoized value
        else if ('value' in self) {
            valueFn(self.value);
        }
        else {
            return get(
                value => {
                    // memoize value
                    this.value = value;
                    valueFn(value);
                }, 
                error => {
                    // memoize error
                    this.error = error;
                    throwFn(error);
                });
        }
    };
}
```

You can retrieve the value from a Task using the Task's get method:

```JavaScript
var subscription = task.get(value => console.log(value), error => console.error(error));
```

The get method returns a subscription object, which can be disposed in order to stop listening for the Tasks eventual value.

```JavaScript
var subscription = task.get(value => console.log(value), error => console.error(error));
// signal to task that we're no longer interested in result
subscription.dispose();
```

Tasks can be sequenced using the 'when' method.

```JavaScript
Task.prototype.when = function(projection) {
    var self = this;
    return new Task(function(valueFn, throwFn) {
        var subscription = 
            self.get(
                x => {
                    try {
                        var nextTask = Task.resolve(projection(x));
                        subscription = nextTask.get(valueFn, throwFn);
                    }
                    catch(e) {
                        throwFn(e);
                    }
                },
                throwFn);

        return {
          dispose: function() {
            subscription.dispose();
          }
        };
    });
};
```

You can also use the resolve function to coerce non-Task values into Tasks:

```JavaScript
Task.resolve = function(v) {
    if (v instanceof Task) {
        return v;
    }
    else {
        // Wrap value in Task that immediately sends the value
        return new Task(function(valueFn, errorFn) {
            valueFn(v);
            // NOOP disposal
            return { dispose: function() {} };
        });
    }
};
```

Now we can define the composition function for Tasks:

```JavaScript
function task(genF) {
    return new Task(function(resolve, reject) {
        var gen = genF();
        function step(nextF) {
            var next;
            try {
                next = nextF();
            } catch(e) {
                // finished with failure, reject the promise
                reject(e); 
                return;
            }
            if(next.done) {
                // finished with success, resolve the promise
                resolve(next.value);
                return;
            } 
            // not finished, chain off the yielded promise and `step` again
            Task.resolve(next.value).when(function(v) {
                step(function() { return gen.next(v); });      
            }, function(e) {
                step(function() { return gen.throw(e); });
            });
        }
        step(function() { return gen.next(undefined); });
    });    
}
```

Now let's add a 'fetchJSON' function to our Task module. This function creates a Task to retrieve JSON from a URL. This action can be cancelled at any time.

```JavaScript
function fetchJSON(url) {
  return new Task((valueFn, errorFn) => {
    var xmlhttp = new XMLHttpRequest();

    xmlhttp.onreadystatechange = function() {
      if (xmlhttp.readyState == 4 && xmlhttp.status == 200) {
        valueFn(JSON.parse(xmlhttp.responseText));
      }
      else if (xmlhttp.status >= 400) {
        errorFn(xmlhttp.status);
      }
    });
  
    xmlhttp.open("GET", url, true);
    xmlhttp.send();
    
    return { dispose: () => xmlhttp.abort() };
  });
}
```

Now we export the Task constructor, composition function, and 'fetchJSON' function.


```JavaScript
//---------- task.js -------------//

export { Task, fetchJSON, task };
```

Now we can use fetchJSON to create Tasks, and sequence them with the 'task' composition functions to sequence Tasks. This makes it easy to create Task sequences that can be cancelled.

## Refactoring Promise code to use Tasks

Let's refactor the Promise example in the first section to use Tasks instead. First we use fetchJSON to rewrite the getStockSymbol and getStockPrice methods to use Tasks instead of Promises. 

```JavaScript

import * from 'task';

function getStockSymbol(name) {
    return fetchJSON('/stockName?' + name);
}

function getStockPrice(symbol) {
    return fetchJSON('/stockPrice?' + symbol);
}
```
Now we can getStockPriceByName to emit Tasks instead of Promises by changing the composition function from 'async' to 'task'.

```JavaScript
// ...continued

// Sequence tasks using the task composition function

var getStockPriceByName = task function(name) {
    var symbol = await getStockSymbol(name);
    var stockPrice = await getStockPrice(symbol);
    return stockPrice;
}

var subscription = 
    getStockPriceByName("Johnson and Johnson").
    get(
        price => console.log(price),
        error => console.error(error));

// Cancel the outgoing requests
subscription.dispose();
```


## Use of other Async Primitives in Popular Frameworks

Two of the most popular MVC frameworks are building data fetching APIs using asynchronous primitives other than Promises. React and Angular 2.0 are in the process of building APIs which use Observable. The Observable interface looks like this:

```JavaScript
interface Subscription {
    dispose(): void
}

interface Observer {
    onNext(value):void,
    onError(error):void,
    onCompleted():void,
}

interface Observable {
    subscribe(Observer): Subscription;
}
```

An Observable is capable of pushing multiple values to a consumer. Unlike a Promise, consumers can unsubscribe from an Observable and elect to receive no further values. As subscription and unsubscription are explicit operations, an Observable can cancel scheduled or pending asynchronous operations when no more listeners are present. This is one of the main advantages of using an Observable over an ES2015 Promise.

The Composition Function for Observables resolves each Observable to the last value received. If any Observable being composed emits no value, the overall Observable completes without a value.

```JavaScript
function observable(genF) {
    return {
        subscribe: function(observer) {
            var gen = genF(),
                // tracks the subscription to the inner Observable
                subscription;
                
            function step(nextF) {
                var next,
                    lastValue,
                    valueReceived;
                try {
                    next = nextF();
                } catch(e) {
                    // finished with failure, reject the Observable
                    observer.onError(e); 
                    return;
                }
                if(next.done) {
                    // finished with success, sending along final value
                    try {
                        observer.onNext(next.value);
                        observer.onCompleted();
                    }
                    catch(e) {
                        observer.onError(e);
                    }
                    return;
                } 
                // not finished, chain off the yielded Observable
                
                // track whether or not value was received
                valueReceived = false;
                subscription = next.value.subscribe({
                    // record last value and track that value
                    // has been received.
                    onNext: function(v) {
                        valueReceived = true;
                        lastValue = v;
                    },
                    // forward along any errors that occur
                    onError: function(e) {
                        observer.onError(e);
                    },
                    // once stream is complete send last value
                    // if a value was received.
                    onCompleted: function() {
                        if (valueReceived) {
                            observer.onNext(lastValue);
                        }
                        observer.onCompleted();
                    });
            }
            step(function() { return gen.next(undefined); });
            
            // return Subscription interface
            return {
                // if subscription to overall Observable is disposed, 
                // unsubscribe from inner Observable subscription if
                // present. This may interrupt a long series of 
                // scheduled operations.
                dispose: function() {
                    if (subscription) {
                        subscription.dispose();
                        subscription = null;
                    }
                }
            }
        }
    };    
}

export { observable };
```

## Composition Functions in Angular 2.0

Here's an example of using an Observable composition function to simplify data access in Angular 2.0:

```JavaScript

import {http} from '../public/http';
import {observable} from 'observable';

observable function getStockPriceByName(stockName) {
    var symbol = await getStockSymbol(stockName);
    var price = await getStockPrice(symbol);
    return price;
}

var price = getStockPriceByName("Netflix");

var subscription = price.subscribe(price => console.log("The price of Netflix is ", price));

exitButton.onclick = function() {
    // cancel network request when user leaves the form
    subscription.dispose();
}
```

For more information on the Angular 2.0 data fetching APIs see https://github.com/jeffbcross/http-design/.

## Composition Functions in React

Like Angular, React is also considering using Observable for data fetching. See this GitHub discussion for details on the React data fetching APIs see https://github.com/facebook/react/issues/3398.

# Composition Functions

The composition function proposal is intended to allow for the composition of any async primitives that results in a single result. 

## Syntax

```
CompositionFunctionDeclaration :
    ImportedBinding [no LineTerminator here] function BindingIdentifier ( FormalParameters ) { FunctionBody }

CompositionFunctionExpression :
    ImportedBinding [no LineTerminator here] function BindingIdentifier? ( FormalParameters ) { FunctionBody }

CompositionMethod :
    ImportedBinding PropertyName (StrictFormalParameters)  { FunctionBody }

CompositionArrowFunction :
    ImportedBinding [no LineTerminator here] ArrowParameters [no LineTerminator here] => ConciseBody

Declaration :
    ...
    CompositionFunctionDeclaration

PrimaryExpression :
    ...
    CompositionFunctionExpression

MethodDefinition :
    ...
    CompositionMethod

AssignmentExpression :
    ...
    CompositionArrowFunction

UnaryExpression :
    ...
    await [Lexical goal InputElementRegExp] UnaryExpression

Note:  await would only be legal inside a CompositionFunction body.  
       This could use similar formalism to ES6 parameterized grammar.
```
