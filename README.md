# setImmediate Issues

__Solution can be see on the solution branch in the Fluxible package__

__What is expected from the following code?__

``` javascript
import React from 'react';
import ApplicationStore from '../stores/ApplicationStore';

class Html extends React.Component {
    render() {
        return (
            <html>
            <head>
                <meta charSet="utf-8" />
                <title>{this.props.context.getStore(ApplicationStore).getPageTitle()}</title>
                <meta name="viewport" content="width=device-width, user-scalable=no" />
                <link rel="stylesheet" href="http://yui.yahooapis.com/pure/0.5.0/pure-min.css" />
            </head>
            <body>
                <div id="app" dangerouslySetInnerHTML={{__html: this.props.markup}}></div>
                <script dangerouslySetInnerHTML={{__html: this.props.state}}></script>
                <script src={'/public/js/' + this.props.clientFile}></script>
            </body>
            </html>
        );
    }
}

export default Html;

```

We expect that the DOM gets parsed and then our client file gets executed which rehydrates then mounts. And that is observed.

Now lets add an additional file

``` javascript
import React from 'react';
import ApplicationStore from '../stores/ApplicationStore';

class Html extends React.Component {
    render() {
        return (
            <html>
            <head>
                <meta charSet="utf-8" />
                <title>{this.props.context.getStore(ApplicationStore).getPageTitle()}</title>
                <meta name="viewport" content="width=device-width, user-scalable=no" />
                <link rel="stylesheet" href="http://yui.yahooapis.com/pure/0.5.0/pure-min.css" />
            </head>
            <body>
                <div id="app" dangerouslySetInnerHTML={{__html: this.props.markup}}></div>
                <script dangerouslySetInnerHTML={{__html: this.props.state}}></script>
                <script src={'/public/js/' + this.props.clientFile}></script>
                <script src={'/public/js/external.js'}></script> <<<<<< ADDITIONAL FILE
            </body>
            </html>
        );
    }
}

export default Html;

```

What I'm expecting is the same as above, DOM parse, then client js execution to rehydrate and mount, then finally call this external file. Well that's not the case...

![Current Observation](/doc/images/Current.png)

What is observed is the DOM gets parsed, and client js is executed and rehydrates BUT setImmediate is wrapped around the actual mounting pushing it into the "next tick" but the DOM parse is still on its "tick". and then continues to run the rest of the client js file, then the external file. Once this "tick" is done it finally pulls the mount exectuion

``` javascript
rehydratePromise
            .then(function (contextValue) {
                // Ensures that errors in callback are not swallowed by promise
                setImmediate(callback, null, contextValue);             
            }, function (err) {
                // Ensures that errors in callback are not swallowed by promise
                setImmediate(callback, err); 
            });
```

This IMO is bad... from the documentation ( https://github.com/YuzuJS/setImmediate and http://jphpsf.github.io/setImmediate-shim-demo/ ) it should be used for breaking apart large sync calls to make it more efficient like in their example. But using it in approprately can case crucial code to be pushed back into the task queue and be block by the remaining execution of the current "tick". AKA blocking the rendering of our application


With the following [change in Fluxible.js](https://github.com/kirbdee/FluxibleSample/blob/solution/node_modules/fluxible/lib/Fluxible.js#L212) we can no observe the expected behavior of the HTML above: DOM parse, then client js execution, to rehydrate and mount, then finally call this external file.


``` javascript
rehydratePromise
            .then(function (contextValue) {                           
                callback(null, contextValue);
            }, function (err) {                           
                callback(err);
            });
```

![Solution Observation](/doc/images/Solution.png)

__But what about "Ensures that errors in callback are not swallowed by promise" part?__

There a far better solutions for error handling than sticking the ENTIRE call back into the queue.

Instead of throwing the error... you already know explicitly that it's an error so why not handle it at this point or pass it to a function to do something about it..

``` javascript
app.rehydrate(dehydratedState, (err, context) => {
    if (err) {
        throw err; <<< Yes if this get's thrown there's nothing to catch it oh now what will we do!?
    }
    window.context = context;
    const mountNode = document.getElementById('app');

    debugClient('React Rendering');
    ReactDOM.render(
        createElementWithContext(context),
        mountNode,
        () => debugClient('React Rendered')
    );
});
```

One example is this:

``` javascript
const functionThatHandlesError = (err) => {
    //DO SOMETHING ABOUT IT
};

const render = (err,context) => {
    if (err) {
        functionThatHandlesError(err); <<<< --- this will "catch" all the errors you have
    }
    window.context = context;
    const mountNode = document.getElementById('app');

    debugClient('React Rendering');
    ReactDOM.render(
        createElementWithContext(context),
        mountNode,
        () => debugClient('React Rendered')
    );
    
};

app.rehydrate(dehydratedState, render);
```

Now you can have the err object actually passed into something you control rather than throwing it into the abyss

__You might also say well they should just be using external.js with the async tag!__ 

while sure that's cool but async in it self can cause timing issues on crucial pieces of your application and can cause your application to seem "slower", while it waits for things, and really should only be used when you really know if the call isn't super crucial and can load whenever. With this solution even if you don't have a synchronous external.js it will bring the render back into the same "tick" and have the application rendered and ready before the DOMContentLoaded event.
