# React Most

[![Join the chat at https://gitter.im/jcouyang/react-most](https://badges.gitter.im/jcouyang/react-most.svg)](https://gitter.im/jcouyang/react-most?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
A Monadic Reactive State Container for React Components

[![Circle CI](https://circleci.com/gh/jcouyang/react-most.svg?style=svg)](https://circleci.com/gh/jcouyang/react-most)
[![npm](https://img.shields.io/npm/dm/react-most.svg?maxAge=2592000)](https://www.npmjs.com/package/react-most)
[![npm](https://img.shields.io/npm/v/react-most.svg?maxAge=2592000)]()

```
npm install react-most --save
```

<!-- TOC depthFrom:1 depthTo:2 withLinks:1 updateOnSave:1 orderedList:0 -->

- [React Most](#react-most)
	- [What](#what)
	- [Why not (just using existing state container e.g.) redux](#why-not-just-using-existing-state-container-eg-redux)
	- [Why indeed?](#why-indeed)
	- [How](#how)
	- [[API](https://github.com/jcouyang/react-most/wiki/API)](#apihttpsgithubcomjcouyangreact-mostwikiapi)
	- [[More Examples](./examples)](#more-examplesexamples)
	- [Performance](#performance)
	- [Thanks to...](#thanks-to)

<!-- /TOC -->

## What
`most.js` is very high performance Monadic reactive streams lib. Rich User Interaction App is natively fit Reactive Programming pretty well.

`React` is awesome for writing UI Components.

So, what `react-most` does is simply making you React Components Reactive.

`react-most` is simple and only 90 lines of code. only depends on most and react.

data flow is simple and one way only
![](https://raw.githubusercontent.com/jcouyang/react-most/master/docs/images/flow.dot.png)

- **Action** will generate value and add it into `Intent Stream`
- **Sink** create new Stream based on action type in `Intent Stream`
- **Sink** transform a value Stream into a `oldState=>newState` mapping Stream
- **State** subscribe to **Sinks** changes


## Why not (just using existing state container e.g.) redux
When I play around with redux, it's awesome but

1. it take little time to make it ready.
2. it's using too many concept that we probably don't even care, which make it's learning curve a little steep(it take a [gitbook](http://rackt.org/redux/index.html) to document just a state container?)
3. Reducers (long switch statements which are syntactically ugly but semantically ok) -- Andre
4. component is not composable, action is not composable, component and behaviour shouldn't be so tight coupling。

again, I couldn't agree more on Andre's [article about react/redux](http://staltz.com/why-react-redux-is-an-inferior-paradigm.html).

Inspired by Reactive Programming, Cycle.js and Redux, we can do something better to reduce the complexity of managing react state, the reactive way, by only introduce a little bit of reactive programming concepts.

## Why indeed?

Redux is awesome, but if you're big fan of Functional Reactive Programming, you would've imaged all user events, actions and data are Streams, then we can map,filter, compose, join those streams to React state.

### Pure Functional, Declarative & Monadic
finstead of imperative describe what you want to do with data at certain step, we simple define data transforms and compose them to data flow. No variable, no state, no side effort at all while you composing data flow.

### Composable and Reusable Sinks
sinks are composable and reusable, not like reducer in redux, where switch statement are hard to break and compose.

### Easy to Test
since UI and UI behaviour are loose couple, you can simply define a dump react component and test it by passing data. seperatly you can test behaviour by given actions, and verify it's state.

```js
let {do$, historyStreamOf} = require('../test-utils')
let todolist = TestUtils.renderIntoDocument(
  <Most >
    <RxTodoList history={true}>
    </RxTodoList>
  </Most>
)
let div = TestUtils.findRenderedComponentWithType(todolist, RxTodoList)
do$([()=>div.actions.done(1),
     ()=>div.actions.done(2),
     ()=>div.actions.remove(2),
     ()=>div.actions.done(1)])
return historyStreamOf(div)
  .take$(4)
  .then(state=>
    expect(state).toEqual({todos: [{id: 1, text: 5, done: false}]}))
```

### Async actions
when function is not async such as promise, simply convert it to Stream and flatMap it
```js
intent$.map(promise=>most.fromPromise)
	.flatMap(value=>dosomething)
```

### Transducers support
[transducer](https://github.com/cognitect-labs/transducers-js) is another high perfomance functional way to compose data flow other than monadic.

writing actions in transducers improve reusablity.

### Time Travel
since we have all action's Stream, we can easyliy reproduce all the action anytime, or get snapshot of state stream and going back in time.

you can pass history as 3rd parameter of `connect`
```js
connect(intent$=>[awesome flow], {history:true})(App)
```

connect will populate history stream with all state history

then you get a create action to deal with historyStream.

### Modular and Easy to Extend
if you're from Rx.js, it's easy to switch rx engine into react-most, by simply pass a props named `engine` to `Most` component

```html
import rxEngine from 'react-most/engine/rx'
<Most engine={rxEngine}>
  <App />
</Most>
```

## How

sorry we don't have a **book** to document how to use `react-most`, but
there's only 3 things you should notice when using `react-most`, I'll explain by a simple counter app.

also you can refer to [API](./docs/api.md) or [Examples](#more-examples).

### 1. Component Wrapper
```html
import Most from 'react-most'
<Most>
  <RxCounter />
</Most>
```
### 2. Define How to connect Component and Streams

```js
import {connect} from 'react-most'
import most from 'most'
const Counter = (props)=>{
  return <div>{props.value}</div>
}
let RxCounter = connect(function(intent$){
  let defaultState$ = most.of(_=>({value:0}))
  let addSink$ = intent$.filter(x=>x.type=='add').map(({increment})=>state=>({value: state.value+increment}))
  return {
    add: increment=>({type: 'add', increment}),
    defaultState$,
    addSink$,
  }
})(Counter);
```
here are things you may need to pay attention to:

#### 2.1. transform intent stream to state mapper stream

the transformer accept a Intetent Stream `intent$`(by convention, all Stream type variable name with suffix $), and create and return new Intent Streams(here we call those new stream -- `sinks`)

```js
  let addSink$ = intent$.filter(x=>x.type=='add').map(({increment})=>state=>({value: state.value+increment}))
```

here we filter out only `add` intent and do something about it.

when I mean something, I mean transform intent to be a state transformer. which means Intent Stream of **values**

--`{value: 1}`--`{value: 2}`--`{value:3}`-->

becomes a Stream of **functions** which tranform old state into new state

--`state=>({value: state.value+1})`--`state=>({value: state.value+2})`--`state=>({value: state.value+3})`-->

#### 2.2. define action mapper that can be use to added intent to your Intent Stream.

```js
    add: increment=>({type: 'add', increment}),
```
here it define a `add` action mapper, it define how you can use the action. it's pretty clear here that the search action will accept only one arg `increment`, and `{type: 'add', increment}` is something will be send to Intent Stream when action is called.

### 3. Use the actions
like redux, but much simpler, when you wrap your App, your App get a `actions` props, and you can pass it all along to any child Component.

```js
<button className="add1"
       type="checkbox"
       onClick={()=>this.props.actions.add(1)} />
```

## [API](https://github.com/jcouyang/react-most/wiki/API)

## [More Examples](./examples)
- [Type N Search](./examples/type-n-search) [(live)](https://reactive-react.github.io/react-most/examples/type-n-search/public/)
- [TodoMVC](./examples/todomvc) [(live)](https://reactive-react.github.io/react-most/examples/todomvc/public/)
- [Type N Search with Transducers](./examples/transducer-type-n-search) [(live)](https://reactive-react.github.io/react-most/examples/transducer-type-n-search/public/)
- [Type N Search with Time Travel](./examples/type-n-search-with-undo) [(live)](https://reactive-react.github.io/react-most/examples/type-n-search-with-undo/public/)

## Performance
`react-most` no more than creating stream from your actions, and bind it to state stream. no any other computations happen in `react-most`. so please refer to [most.js's perf](https://github.com/cujojs/most/tree/master/test/perf) which is realy Great!

I also do a simple benchmark with 8k times of performing counter increase action
```
Memory Usage Before: { rss: 32501760, heapTotal: 16486912, heapUsed: 11307128 }
Memory Usage After: { rss: 34418688, heapTotal: 18550784, heapUsed: 11932336 }
Elapsed 8ms
```
basically the same level of performance as redux(which is 10ms in the same testing)

## Thanks to...
- [most](https://github.com/cujojs/most)
- [React](http://facebook.github.io/react/)
- [redux](https://github.com/rackt/redux)
- [Om](https://github.com/omcljs/om)
- [Cycle](http://cycle.js.org/)
- [transdux](https://github.com/jcouyang/transdux)
