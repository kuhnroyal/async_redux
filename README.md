[![pub package](https://img.shields.io/pub/v/async_redux.svg)](https://pub.dartlang.org/packages/async_redux)
[![pub package](https://img.shields.io/badge/Awesome-Flutter-blue.svg?longCache=true&style=flat-square)](https://github.com/Solido/awesome-flutter)

# Async Redux | *state management*

Async Redux is an optimized Redux version, which is very easy to learn and use,
yet powerful and tailored for Flutter.
It helps you write Flutter apps that are **easy to test, maintain and extend**.

# Developer Overview

The main concepts in Redux are: *store*, *state*, *actions* and *reducers*.

The **store** holds all the application **state**,
and you can only change the state by dispatching **actions**.
Each action has a **reducer**, which changes the state:

```dart
// Create a store, which holds the initial app state.
var store = Store<int>(initialState: 1);

// Create an action class with a `reduce` method that changes the state.
class Increment extends Action {  
  int reduce() => state + 1; 
}

void main() {
  // Dispatch the action to change the state.
  store.dispatch(Increment());
  print(store.state); // 2
}
```

To use the `store` in your widgets, add a `StoreProvider` to the top of your widget tree:

```dart
Widget build(BuildContext context) {
  return StoreProvider<int>(
    store: store,
    child: MaterialApp(home: MyHomePage()), ...
    );
}
```

You can then use it anywhere in your widgets like this:

```dart
class MyWidget extends StatelessWidget {  
  Widget build(BuildContext context) {
    return Column(children: [        
        
      Text(context.state.toString()), // Use the state.        
        
      ElevatedButton(
        child: Text('+')),            
        onPressed: () => context.dispatch(Increment())) // Dispatch the action.               
    ]);    
  }}
```

Your actions can download information from the internet, or do any other asynchronous work:

```dart
var store = Store<String>(initialState: '');

class LoadText extends Action {

  // This reducer returns a Future.  
  Future<String> reduce() async {
  
    // Download some information from the internet.
    var response = await http.get('http://numbersapi.com/42');    
    
    // Change the state with the downloaded information.
    return response.body;      
  }
}
```

If some error happens, you can simply throw an `UserException`.
A dialog (or other UI) will open automatically, showing the error message to the user.

```dart
var store = Store<String>(initialState: '');

class LoadText extends Action {
    
  Future<String> reduce() async {  
    var response = await http.get('http://numbersapi.com/42');    

    if (response.statusCode == 200) return response.body;
    else throw UserException('Failed to load data');         
  }
}
```

Then:

* If you want to show a spinner while the text is loading, you can use `isWaiting`.
* If you want to show an error message as part of your widget tree, you can use `isFailed`.

```dart
class MyWidget extends StatelessWidget {
  Widget build(BuildContext context) {
  
    if (context.isWaiting(LoadText)) return CircularProgressIndicator();
    
    if (context.isFailed(LoadText)) return Text('Loading failed...');
    
    return Column(children: [
      
       Text(context.state), // Show the state
                      
       ElevatedButton(
         child: Text('Load')),            
         onPressed: () => context.dispatch(LoadText())) // Dispatch the action.               
    ]);    
}}
```

Your actions can also dispatch other actions, and use `dispatchAndWait` to
wait for an action to finish:

```dart
class LoadTextAndIncrement extends Action {

  Future<AppState> reduce() async {    
    
    // Dispatch and wait for the action to finish
    await dispatchAndWait(LoadText());
    
    // Only then, increment the state
    return state.copy(count: state.count + 1);
  }
}
```

You can also dispatch actions in parallel and wait for them to finish:

```dart 
class BuyAndSell extends Action {

  Future<AppState> reduce() async {
  
    // Dispatch and wait for both actions to finish
    await dispatchAndWaitAll([
      BuyAction('IBM'), 
      SellAction('TSLA')
    ]);
    
    return state.copy(message: 'New cash balance is ${state.cash}');
  }
}
``` 

You can also use waitCondition to wait until the state changes in a certain way:

```dart
class SellStockForPrice extends Action {
  final String stock;
  final double limitPrice;
  SellStockForPrice(this.stock, this.limitPrice);

  Future<AppState?> reduce() async {  
  
    // Wait until the stock price is higher than the limit price
    await waitCondition(
      (state) => state.stocks[stock].price >= limitPrice
    );
      
    dispatch(SellStock(stock));
    
    // No further state change    
    return null; 
}
```

You can add **mixins** to your actions, to accomplish common tasks:

* `CheckInternet` ensures actions only run with internet, otherwise an error dialog
  prompts users to check their connection:

  ```dart
  class LoadText extends Action with CheckInternet {
      
  Future<String> reduce() async {
      var response = await http.get('http://numbersapi.com/42');
      ...      
  }}
  ```

* `NoDialog` can be added to `CheckInternet` so that no dialog is opened.
  Instead, you can display some information in your widgets:

  ```dart
  class LoadText extends Action with CheckInternet, NoDialog { ... }
  
  if (context.isFailed(LoadText)) Text('No Internet connection');
  ```

* `AbortWhenNoInternet` aborts the action silently (without showing any dialogs)
  if there is no internet connection.

* `NonReentrant` prevents reentrant actions, so that when you dispatch an action that's
  already running it gets aborted.

* `Retry` retries the action a few times with exponential backoff, if it fails.
  Add `UnlimitedRetries` to retry the action indefinitely:

  ```dart
  class LoadText extends ReduxAction<AppState> with Retry, UnlimitedRetries, NonReentrant { 
  ```

Testing your app is very easy. Just dispatch actions and wait for them to finish.
Then, verify the new state or check if some error was thrown:

```dart
class AppState {
  User user;
  int selected;
  List<Item> items;    
}

test('Selecting an item', () async {   

    var store = Store<AppState>(
      initialState: AppState(
        user: User(name: 'John'),
        selected: -1,
        items: [Item(id: 1), Item(id: 2), Item(id: 3)]
      ));
    
    // Found item 2.                
    await store.dispatchAndWait(SelectItem(2));    
    expect(store.state.selected, 2);
    
    // Failed to find item 42.
    var status = await store.dispatchAndWait(SelectItem(42));    
    expect(status.originalError, isA<>(UserException));
});       
```

# Team Lead Overview

If you are a Team Lead, you'll have features to help you set up the app's infrastructure in a
central place, and allow your developers to concentrate solely on the business logic.

When you create the store, you can add
a `persistor` to save and load the state from the local device disk,
a `stateObserver` to collect app metrics,
an `errorObserver` to log errors,
an `actionObserver` to print information to the console during development,
and a `globalWrapError` to catch all errors thrown by actions and decide what to do with them.

```dart
var store = Store<String>(
  initialState: '',
  persistor: MyPersistor(),
  stateObserver: [MyStateObserver()],
  errorObserver: [MyErrorObserver()],
  actionObservers: [MyActionObserver()],
  globalWrapError: MyGlobalWrapError(),  
```  

For example, the following `GlobalWrapError` is designed to handle all `PlatformException` errors
throw by **Firebase**. It converts them into `UserException` errors, which are built-in Async Redux
types that automatically display their message to the user in an error dialog:

```dart
Object? wrap(error, stackTrace, action) {
  return (error is PlatformException)
    ? UserException('Error connecting to Firebase')
    : error;
}   
```

Another interesting feature for Team Leads is the ability to create a base action class that all
your actions will extend, and add some common functionality to it. For example, you can add getters
for the important parts of your state, and also "selectors" to help you find more complex
information:

```dart
class AppState {
  User user;
  int selected;
  List<Item> items;    
}

class Action extends ReduxAction<AppState> {

  // Getters  
  User get user => state.user;
  Item get selected => state.selected;
  List<Item> get items => state.items;
  
  // Selectors 
  Item? findItemById(int id) => items.firstWhereOrNull((item) => item.id == id);
  Item? searchItemByText(String text) => items.firstWhereOrNull((item) => item.text.contains(text));
  int get selectedItemIndex => items.indexOf(selected);     
}
```

Now, all your actions can use these getters and selectors to access the state in their reducers:

```dart 
class SelectItem extends Action {
  final int id;
  SelectItem(this.id);
    
  AppState reduce() {
    Item? item = findItemById(id);
    if (item == null) throw UserException('Item not found');
    return state.copy(selected: item);
  }    
}
``` 

<br>

---

---

[//]: # (The below documentation is very detailed. For an overview, go to)

[//]: # (the <a href="https://medium.com/@marcglasberg/https-medium-com-marcglasberg-async-redux-33ac5e27d5f6?sk=87aefd759273920d444185aa9d447ba0">)

[//]: # (Medium story</a>.)

# Example projects

Please visit:

* **Fully documented:**
  > The <a href="https://github.com/marcglasberg/SameAppDifferentTech/">Same App
  Different Tech
  Project</a> is a repository in GitHub, containing the same mobile app implemented using a variety
  of different tech stacks, including
  a <a href="https://github.com/marcglasberg/SameAppDifferentTech/blob/main/MobileAppFlutterRedux/README.md">
  Redux App Example</a>.

* **Documented in the source code only:**
  > The <a href="https://github.com/marcglasberg/redux_app_example">Redux App Example</a>
  repository in GitHub also contains a full-fledged example with a complete app showcasing the
  fundamentals and best practices described in this Readme.

# Table of Contents

* [What is Redux?](#what-is-redux)
* [Why use this Redux version over others?](#why-use-this-redux-version-over-others)
* [Store and State](#store-and-state)
* [Actions](#actions)
    * [Sync Reducer](#sync-reducer)
    * [Async Reducer](#async-reducer)
    * [Changing state is optional](#changing-state-is-optional)
    * [Before and After the Reducer](#before-and-after-the-reducer)
* [Connector](#connector)
    * [How to provide the ViewModel to the StoreConnector](#how-to-provide-the-viewmodel-to-the-storeconnector)
* [Alternatives to the Connector](#alternatives-to-the-connector)
    * [Provider](#provider)
* [Processing errors thrown by Actions](#processing-errors-thrown-by-actions)
    * [Giving better error messages](#giving-better-error-messages)
    * [User exceptions](#user-exceptions)
    * [Converting third-party errors into UserExceptions](#converting-third-party-errors-into-userexceptions)
    * [UserExceptionAction](#userexceptionaction)
* [Testing](#testing)
    * [Mocking actions and reducers](#mocking-actions-and-reducers)
    * [Testing UserExceptions](#testing-userexceptions)
    * [Test files](#test-files)
* [Route Navigation](#route-navigation)
* [Events](#events)
    * [Can I put mutable events into the store state?](#can-i-put-mutable-events-into-the-store-state)
    * [When should I use events?](#when-should-i-use-events)
    * [Advanced event features](#advanced-event-features)
* [Progress indicators](#progress-indicators)
* [Waiting until an Action is finished](#waiting-until-an-action-is-finished)
* [Waiting until the state meets a certain condition](#waiting-until-the-state-meets-a-certain-condition)
* [State Declaration](#state-declaration)
    * [Selectors](#selectors)
    * [Cache (Reselectors)](#cache-reselectors)
* [Action Subclassing](#action-subclassing)
    * [Abstract Before and After](#abstract-before-and-after)
* [Dependency Injection](#dependency-injection)
* [IDE Navigation](#ide-navigation)
* [Persistence](#persistence)
    * [Saving and Loading](#saving-and-loading)
* [Logging](#logging)
* [Observing rebuilds](#observing-rebuilds)
* [How to interact with the database](#how-to-interact-with-the-database)
* [How to deal with Streams](#how-to-deal-with-streams)
    * [So, how do you use streams?](#so-how-do-you-use-streams)
    * [Where the stream subscriptions themselves are stored](#where-the-stream-subscriptions-themselves-are-stored)
    * [How do streams pass their information to the store and ultimately to the widgets?](#how-do-streams-pass-their-information-to-the-store-and-ultimately-to-the-widgets)
    * [To sum up:](#to-sum-up)
* [Undo and Redo](#undo-and-redo)
* [Recommended Directory Structure](#recommended-directory-structure)
* [Where to put your business logic](#where-to-put-your-business-logic)
* [Architectural discussion](#architectural-discussion)
    * [Is AsyncRedux really Redux?](#is-asyncredux-really-redux)
    * [Besides the reduction of boilerplate, what are the main advantages of the AsyncRedux architecture?](#besides-the-reduction-of-boilerplate-what-are-the-main-advantages-of-the-asyncredux-architecture)
    * [Is AsyncRedux a minimalist or lightweight Redux version?](#is-asyncredux-a-minimalist-or-lightweight-redux-version)
    * [Is the AsyncRedux architecture useful for small projects?](#is-the-asyncredux-architecture-useful-for-small-projects)

<br>

## What is Redux?

A single **store** object holds all the **state**, which is immutable. When you need to modify some
state, you **dispatch** an **action**. Then a **reducer** creates a new copy of the state, with the
desired changes. Your widgets are **connected** to the store (through **store-connectors** and
**view-models**), so they know that the state changed, and rebuild as needed.

<br>

## Why use this Redux version over others?

Plain vanilla Redux is too low-level, which makes it very flexible but results in a lot of
boilerplate, and a steep learning curve. Combining reducers is a manual task, and you have to list
them one by one. If you forget to list some reducer, you will not know it until your tests point out
that some state is not changing as you expected.

Reducers can't be async, so you need to create middleware, which is also difficult to set up and
use. You have to list them one by one, and if you forget one of them you will also not know it until
your tests point it out. The `redux_thunk` package can help with that, but adds some more
complexity.

It's difficult to know which actions fire which reducers, and hard to navigate the code in the IDE.
In IntelliJ, you may press CTRL+B to navigate between a method use and its declaration. However,
this is of no use if actions and reducers are independent classes. You have to search for action
"usages", which is not so convenient since it also list dispatches.

It's also difficult to list all actions and reducers, and you may end up implementing some reducer
just to realize it already exists with another name.

Testing reducers is simple, since they are pure functions, but integration tests are difficult. In
the real world you need to test complex middleware that fires other middleware and many reducers,
with intermediate state changes that you want to test for. Especially if you are doing BDD or
Acceptance Tests you may need to wait for some middleware to finish, and then dispatch some other
actions, and test for intermediate states.

Another problem is that vanilla Redux assumes it holds all the application state, and this is not
practical in a real Flutter app. If you add a simple `TextField` with a `TextEditingController`, or
a `ListView` with a `ScrollController`, then you have state outside the Redux store. Suppose your
middleware is downloading some information, and it wishes to scroll a `ListView` as soon as the info
arrives. This would be simple if the list scroll position is saved in the Redux store. However, this
state must be in the `ScrollController`, not the store.

**AsyncRedux solves all of these problems and more:**

* It's much easier to learn and use than regular Redux.
* It comes with its own testing tools that make even complex tests easy to set up and run.
* You can navigate between action dispatches and their corresponding reducers with a single IDE
  command or click.
* You can also use your IDE to list all actions/reducers.
* You don't need to add or list reducers and middleware anywhere.
* In fact, reducers can be async, so you don't need middleware.
* There is no need for generated code (as some Redux versions do).
* It has the concept of "events", to deal with Flutter state controllers.
* It helps you show errors thrown by reducers to the user.
* It's easy to add both logging and store persistence.

<br>

## Store and State

Declare your store and state, like this:

```dart
var state = AppState.initialState();

var store = Store<AppState>(
  initialState: state,
);
```  

Note: _Your state can be any **immutable** object, but typically you create a class
called `AppState` to help with the state creation and manipulation. I later give
some [recommendations](#state-declaration) on how to create this class. In special, you can use the
<a href="https://pub.dev/packages/fast_immutable_collections">fast_immutable_collections</a> package
when you need immutable lists, sets, maps and multimaps._


<br>

## Actions

If you want to change the store state you must "dispatch" some action. In AsyncRedux all actions
extend `ReduxAction`.

The reducer of an action is simply a method of the action itself, called `reduce()`. All actions
must override this method.

The reducer has direct access to:

- The store state (which is a getter of the `Action` class).
- The action state itself (the class fields, passed to the action when it was instantiated and
  dispatched).
- The `dispatch` method, so that other actions may be dispatched from the reducer.

<br>

The abstract `reduce()` method signature has a return type of `FutureOr<AppState?>`, but
your concrete reducers must return one or the other: `AppState?` or `Future<AppState?>`.

That's necessary because AsyncRedux knows if a reducer is sync or async by checking your `reduce()`
method signature. If it is `FutureOr<AppState?>`, it can't know if it's sync or async,
and will throw a `StoreException`:

```
Reducer should return `St?` or `Future<St?>`. Do not return `FutureOr<St?>`.
```

### Sync Reducer

If you want to do some synchronous work, simply declare the reducer to return `AppState?`, then
change the state and return it.

For example, let's start with a simple action to increment a counter by some value:

```dart
class IncrementAction extends ReduxAction<AppState> {

  final int amount;

  IncrementAction({this.amount});

  @override
  AppState? reduce() {
	return state.copy(counter: state.counter + amount));
  }
}
```

This action is dispatched like this:

```dart
store.dispatch(IncrementAction(amount: 3));
```

Note the reducer above has direct access to both the counter state (`state.counter`)
and to the action state (the field `amount`).

We will show you later how to easily test sync reducers, using the **StoreTester**.

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main.dart">
Increment Example</a>.

<br>

### Async Reducer

If you want to do some asynchronous work, simply declare the reducer to return `Future<AppState?>`
then change the state and return it. There is no need of any "middleware", like for other Redux
versions.

Note: In IntelliJ, to convert the reducer from sync to async, press `Alt+ENTER` and
select `Convert to async function body`.

As an example, suppose you want to increment a counter by a value you get from the database. The
database access is async, so you must use an async reducer:

```dart
class QueryAndIncrementAction extends ReduxAction<AppState> {

  @override
  Future<AppState?> reduce() async {
	int value = await getAmount();
	return state.copy(counter: state.counter + value));
  }
}
```

This action is dispatched like this:

```dart
store.dispatch(QueryAndIncrementAction());
```

Please note: While the `reduce()` method of a *sync* reducer runs synchronously with the dispatch,
the `reduce()` method of an *async* reducer will be called synchronously, but will always return
the state in a later microtask.

We will show you later how to easily test async reducers, using the **StoreTester**.

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_increment_async.dart">
Increment Async Example</a>.

#### One important rule

When your reducer is async (i.e., returns `Future<AppState>`) you must make sure you **do not return
a completed future**, meaning all execution paths of the reducer must pass through at least
one `await` keyword. In other words, don't return a Future if you don't need it.

If you don't follow this rule, AsyncRedux may seem to work ok, but will eventually misbehave.

If your reducer has no `await`s, you must return `AppState?` instead of `Future<AppState?>`,
or simply add `await microtask;` to the start of your reducer, or return `null`. For example:

```dart 
// These are right:
AppState? reduce() {
  return state;
}

AppState? reduce() {
  someFunc();
  return state;
}

Future<AppState?> reduce() async {
  await someFuture();
  return state;
}

Future<AppState?> reduce() async {
  await microtask;
  return state;
}

Future<AppState?> reduce() async {
  if (state.someBool) return await calculation();
  return null;
}

// But these are wrong:
Future<AppState?> reduce() async {
  return state;
}

Future<AppState?> reduce() async {
  someFunc();
  return state;
}

Future<AppState?> reduce() async {
  if (state.someBool) return await calculation();
  return state;
}
```

It's generally easy to make sure you are not returning a completed future.
In the rare case your reducer function is very complex, and you are unsure that all code paths
pass through an `await`, just add `assertUncompletedFuture();` at the very END of your `reduce`
method, right before the `return`. If you do that, an error will be shown in the console if
the `reduce` method ever returns a completed future.

If you're an advanced user interested in the details, check the
<a href="https://github.com/marcglasberg/async_redux/blob/master/test/sync_async_test.dart">
sync/async tests</a>.

<br>

### Changing state is optional

For both sync and async reducers, returning a new state is optional. If you don't plan on changing
the state, simply return `null`. This is the same as returning the state unchanged.

Why is this useful? Because some actions may simply start other async processes, or dispatch other
actions.

For example, suppose you want to have two separate actions, one for querying some value from the
database, and another action to change the state:

```dart
class QueryAction extends ReduxAction<AppState> {

  @override
  Future<AppState> reduce() async {
    int value = await getAmount();
    dispatch(IncrementAction(amount: value));
    return null;
  }
}

class IncrementAction extends ReduxAction<AppState> {
  
  final int amount;

  IncrementAction({this.amount});

  @override
  AppState reduce() {
    return state.copy(counter: state.counter + amount));
  }
}
```

Note the `reduce()` methods have direct access to `state` and `dispatch`. There is no need to
write `store.state` and `store.dispatch` (although you can, if you want).

<br>

### Before and After the Reducer

Sometimes, while an async reducer is running, you want to prevent the user from touching the screen.
Also, sometimes you want to check preconditions like the presence of an internet connection, and
don't run the reducer if those preconditions are not met.

To help you with these use cases, you may override methods `ReduxAction.before()`
and `ReduxAction.after()`, which run respectively before and after the reducer.

The `before()` method runs before the reducer. If you want it to run synchronously, it should
return `void`:

```dart
void before() { ... }
```

To run it asynchronously, return `Future<void>`:

```dart
Future<void> before() async { ... }
```

If it throws an error, then `reduce()` will NOT run. This means you can use it to check any
preconditions and throw an error if you want to prevent the reducer from running. For example:

```dart
Future<void> before() async => await checkInternetConnection();
```

This method is also capable of dispatching actions, so it can be used to turn on a modal barrier:

```dart
void before() => dispatch(BarrierAction(true));
```

Note: If this method runs asynchronously, then `reduce()` will also be async, since it must wait for
this one to finish.

The `after()` method runs after `reduce()`, even if an error was thrown by `before()` or `reduce()`
(akin to a "finally" block).

Avoid `after()` methods which can throw errors. If the `after()` method throws an error, then this
error will be thrown *asynchronously* (after the "asynchronous gap")
so that it doesn't interfere with the action. Also, this error will be missing the original
stacktrace.

The `after()` method can also dispatch actions, so it can be used to turn off some modal barrier
when the reducer ends, even if there was some error in the process:

```dart
void after() => dispatch(BarrierAction(false));
```

Complete example:

```dart
// This action increments a counter by 1, and then gets some description text.
class IncrementAndGetDescriptionAction extends ReduxAction<AppState> {

  @override
  Future<AppState> reduce() async {
	dispatch(IncrementAction());
	String description = await read(Uri.http("numbersapi.com","${state.counter}");
	return state.copy(description: description);
  }

  void before() => dispatch(BarrierAction(true));

  void after() => dispatch(BarrierAction(false));
}
```

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_before_and_after.dart">
Before and After Example</a>.

#### Wrapping the reducer

You may wrap the reducer to allow for some pre or post-processing. For example, suppose you want to
abort the reducer if the state changed since while the reducer was running:

```dart
Reducer<St> wrapReduce(Reducer<St> reduce) => () async {
   var oldState = state; // Remember: `state` is a getter for the current state.
   AppState? newState = await reduce(); // This may take some time, and meanwhile the state may change. 
   return identical(oldState, state) ? newState : null;
};
```

#### Aborting the dispatch

You may override the action's `abortDispatch` to completely prevent the action to run if some
condition is true; In more detail, if this method returns `true`, methods `before`, `reduce`
and `after` will not be called, and the action will not be visible to the `StoreTester`. This is
only useful under rare circumstances, and you should only use it if you know what you are doing. For
example:

```dart
@override
bool abortDispatch() => state.user.name == null;
```

#### Action status

You can use `action.status.isCompletedOk` to check if a dispatched action finished with no
errors (in more detail, if the action's methods `before` and `reduce` finished without throwing
any errors):

```dart
var action = MyAction(); 
await store.dispatchAndWait(action);
print(action.isCompletedOk);
```

Better yet, you can get this information directly through the `dispatchAndWait` method:

```dart       
var status = await store.dispatchAndWait(MyAction());
print(status.isCompletedOk);
```

One use case is when you want to save some info, and you want to leave the current screen if and
only if the save process succeeded:

```dart
class SaveAction extends ReduxAction<AppState> {      
  Future<AppState> reduce() async {
    bool isSaved = await saveMyInfo(); 
    if (!isSaved) throw UserException("Save failed.");	 
    ...
  }
}

var status = await dispatch(SaveAction(info));
if (status.isCompletedOk) Navigator.pop(context); // Or: dispatch(NavigateAction.pop()) 
```

This is all the information you can get from the action status:

```dart
var status = await dispatch(MyAction(info));
print(status.isCompleted);
print(status.isCompletedOk);
print(status.isCompletedFailed);
print(status.originalError);
print(status.wrappedError);
print(status.status.hasFinishedMethodBefore);
print(status.status.hasFinishedMethodReduce);
print(status.status.hasFinishedMethodAfter);
```

#### What's the order of execution of sync and async reducers?

A reducer is only sync if both `reduce()` return `AppState` AND `before()` return `void`. If you
any of them return a `Future`, then the reducer is async.

When you dispatch **sync** reducers, they are executed synchronously, in the order they are called.
For example, this code will wait until `MyAction1` is finished and only then run `MyAction2`,
assuming both are sync reducers:

```dart
dispatch(MyAction1()); 
dispatch(MyAction2());
```

Dispatching an async reducer without writing `await` is like starting any other async processes
without writing `await`. The process will start immediately, but you are not waiting for it to
finish. For example, this code will start both `MyAsyncAction1` and `MyAsyncAction2`, but is says
nothing about how long they will take or which one finishes first:

```dart
dispatch(MyAsyncAction1()); 
dispatch(MyAsyncAction2());
```

If you want to wait for some async action to finish, you must write `await dispatch(...)`
instead of simply `dispatch(...)`. Then you can actually wait for it to finish. For example, this
code will wait until `MyAsyncAction1` is finished, and only then run `MyAsyncAction2`:

```dart
await dispatch(MyAsyncAction1()); 
await dispatch(MyAsyncAction2());
```

<br>

## Using BuildContext to access the store

To access the store state inside of widgets, you can use the provided extension on `context`:

```dart
// Read state (will rebuild when the state changes) 
var myInfo = context.state.myInfo;

// Dispatch action
context.dispatch(MyAction());

// Use isWaiting to show a spinner 
var isWaiting = context.isWaiting(MyAction); 

// Use isFailed to show an error message
if (context.isFailed(MyAction)) return Text('Loading failed');
                                                                   
// Use exceptionFor to get the error message from the exception
if (context.isFailed(MyAction)) return Text(context.exceptionFor(MyAction).message);

// Use clearExceptionFor to clear the error
context.clearExceptionFor(MyAction);
```  

In more detail:

* `var state = context.getState<AppState>` - Reads the store state. Widgets that use this will
  rebuild whenever the state changes.

  It's recommended that you define this extension in your own code:

  ```dart
  extension BuildContextExtension on BuildContext {
     AppState get state => getState<AppState>();
  }
  ```

  This will allow you to write:
  ```dart
  var state = context.state;
  ```


* `context.dispatch()`, `.dispatchAndWait()` and `.dispatchSync()` - Dispatch an action.

* `context.dispatchAll()`, and `.dispatchAndWaitAll()` - Dispatch multiple actions.

* `context.isWaiting()` - Returns true if the given action type is currently being processed.

* `context.isFailed()` - Returns true if an action of the given type failed with an `UserException`.

* `context.exceptionFor()` - Returns the `UserException` of the action of the given type that
  failed.

* `context.clearExceptionFor()` - Removes the given type from the list of action types that failed.

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_conector_vs_provider.dart.dart">
Connector vs Provider Example</a>.

## Connector

In Redux, you generally have two widgets, one called the "dumb-widget", which knows nothing about
Redux and the store, and another one to "wire" the store with that dumb-widget.

While Vanilla Redux traditionally calls these wiring widgets "containers", Flutter's most common
widget is already called a `Container`, which can be confusing. So I prefer calling them "
connectors".

They do their magic by using a `StoreConnector` and a `ViewModel`.

A view-model is a helper object to a `StoreConnector` widget. It holds the part of the Store state
the corresponding dumb-widget needs, and may also convert this state part into a more convenient
format for the dumb-widget to work with.

In more detail: Each time some action reducer changes the store state, all `StoreConnector`s in the
screen will use that state to create a new view-model, and then compare it with the previous
view-model created with the previous state. Only if the view-model changed, the connector rebuilds.

For example:

```dart
class MyHomePageConnector extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
	return StoreConnector<AppState, ViewModel>(
	  vm: () => Factory(this),
	  builder: (BuildContext context, ViewModel vm) => MyHomePage(
		counter: vm.counter,
		description: vm.description,
		onIncrement: vm.onIncrement,
	  ));
  }}

class Factory extends VmFactory<AppState, MyHomePageConnector, ViewModel> {
  Factory(connector) : super(connector);
  @override
  ViewModel fromStore() => ViewModel(
      counter: state.counter,
      description: state.description,
      onIncrement: () => dispatch(IncrementAndGetDescriptionAction()),
      );
}

class ViewModel extends Vm {  
  final int counter;
  final String description;
  final VoidCallback onIncrement;
  ViewModel({
       required this.counter,
       required this.description,
       required this.onIncrement,
  }) : super(equals: [counter, description]);
}
```

For the view-model comparison to work, your ViewModel class must implement equals/hashcode.
Otherwise, the `StoreConnector` will think the view-model changes everytime, and thus will rebuild
everytime. This won't create any visible problems to your app, but is inefficient and may be slow.

The equals/hashcode can be done in three ways:

* By typing `ALT`+`INSERT` in IntelliJ IDEA and choosing `==() and hashcode`. You can't forget to
  update this whenever new parameters are added to the model.

* You can use the <a href="https://pub.dev/packages/built_value">built_value</a> package to ensure
  they are kept correct, without you having to update them manually.

* Just add all the fields you want (which are not callbacks) to the `equals` parameter to
  the `ViewModel`'s `build` constructor. This will allow the ViewModel to automatically create its
  own `operator ==` and `hashcode` implicitly. For example:

```dart
ViewModel({
  required this.field1,
  required this.field2,
}) : super(equals: [field1, field2]);
```      

Note: Each state passed in the `equals` parameter will, by default, be compared by equality (`==`).
However, you can provide your own comparison method, if you want. To that end, your state classes
may implement the `VmEquals` interface. As a default, objects of type `VmEquals` are compared by
their `VmEquals.vmEquals()` method, which by default is an identity comparison. You may then
override this method to provide your custom comparisons.

For example, here `description` will be compared by equality, while `myObj` will be compared by
its `info` length:

```dart
class ViewModel extends Vm {
  final String description;
  final MyObj myObj;  

  ViewModel({
    required this.description,
    required this.myObj,
  }) : super(equals: [description, myObj]);
}

...

class MyObj extends VmEquals<MyObj> {  
  String info; 
  bool operator ==(Object other) => info.length == other.info.length;
  int get hashCode => 0;
}
```

<br>

### How to provide the ViewModel to the StoreConnector

The `StoreConnector` actually accepts two parameters for the `ViewModel`, of which **only one**
should be provided in the `StoreConnector` constructor: `vm` or `converter`.

1. the `vm` parameter

   Most examples in the [example tab](https://pub.dartlang.org/packages/async_redux#-example-tab-)
   use the `vm` parameter.

   The `vm` parameter expects a function that creates a `Factory` object that extends
   `VmFactory`. This class should implement a method `fromStore` that returns a `ViewModel`
   that extends `Vm`:

   ```dart
   @override
     Widget build(BuildContext context) {
       return StoreConnector<int, ViewModel>(
         vm: () => Factory(),
         builder: (BuildContext context, ViewModel vm) => MyWidget(...),
       );
     }
   ```   

   AsyncRedux will automatically inject `state`, `currentState()` and `dispatch()` into your model
   instance, so that boilerplate is reduced in your `fromStore` method. For example:

   ```dart
   class Factory extends VmFactory<AppState, MyHomePageConnector, ViewModel> {
              
        @override
        ViewModel fromStore() => ViewModel(
            counter: state.counter,
            description: state.description,
            onIncrement: () => dispatch(IncrementAndGetDescriptionAction()),
            );
   }
   ```     

   **Note:**

    * `state` getter: The state the store was holding when the factory and the view-model were
      created. This state is final inside the factory.

    * `currentState()` method: The current (most recent) store state. This will return the current
      state the store holds at the time the method is called.

   <br>

   If you need it, you may pass the connector widget to the factory's constructor, like this:

   ```dart    
   vm: () => Factory(this),
   
   ...
   
   class Factory extends VmFactory<AppState, MyHomePageConnector, ViewModel> {
      Factory(connector) : super(connector);
   
      @override
      ViewModel fromStore() => ViewModel(
          name: state.names[widget.user],             
          );
      }
   ```

   The `vm` parameter's architecture lets you create separate methods for helping construct your
   model, without having to pass the `store` around. For example:

   ```dart
   @override
   ViewModel fromStore() => ViewModel(
       name: _name(),
       onSave: _onSave,
   );
    
   String _name() => state.user.name;
    
   VoidCallback _onSave: () => dispatch(SaveUserAction()),
   ```  

   You can reference the view-model inside the Factory methods, by using the `vm` getter.
   For example:

   ```dart
   @override
   ViewModel fromStore() => ViewModel(
       name: state.user.name,
       onSave: _onSaveName,
   );       
   
   // Use `vm.name` here.  
   VoidCallback _onSaveName: () => dispatch(SaveUserAction(vm.name)),
   ```            

   Please note, you can only use the `vm` getter after the `fromStore()` method returns, which
   means you cannot use it inside the `fromStore()` method itself. If you do that,
   you'll get a `StoreException`.

   <br>

   Another idea is to subclass `VmFactory` to:

    * Reduce boilerplate, and not having to pass the `AppState` type parameter whenever you
      create a Factory.

    * Provide additional features to your model. For example, you could add extra getters to help
      you access state.

   Example:

   ```dart       
   abstract class BaseFactory<T extends Widget?, Model extends Vm> 
     extends VmFactory<AppState, T, Model> {
          
     BaseFactory([T? connector]) : super(connector);
         
     User get user => state.user;        
   }
    
   class _Factory extends BaseFactory {
    
     @override
     ViewModel fromStore() => ViewModel(
        name: user.name, // Instead of `name: state.user.name`       
     );                                    
   }
   ```   

   <br>

**What if you can't generate the view-model?**

Note: Sometimes you don't have enough information to generate the view-model. For example, some
information may still be loading, or the state is inconsistent for some reason. In that case,
your Factory can return `null` instead of the `vm`, and the connector can return an alternative
placeholder widget.

To that end, declare the view-model as nullable (`ViewModel?`) in these 3 places:
the `StoreConnector`, the `builder`, and the `fromStore` method. Then, check for `null` in
the `builder`. For example:

   ```dart                               
   return StoreConnector<AppState, ViewModel?>( // 1. Use `ViewModel?` here!
   vm: () => Factory(this),       
   builder: (BuildContext context, ViewModel? vm) { // 2. Use `ViewModel?` here!
     return (vm == null) // 3. Check for null view-model here.
       ? Text("The user is not logged in")
       : MyHomePage(user: vm.user)
   
   ...                         
   
   class Factory extends VmFactory<AppState, MyHomePageConnector, ViewModel> {   
   ViewModel? fromStore() { // 4. Use `ViewModel?` here!
   return (store.state.user == null)
       ? null
       : ViewModel(user: store.state.user)
   
   ...
   
   class ViewModel extends Vm {
   final User user;  
   ViewModel({required this.user}) : super(equals: [user]);
   ```         

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_null_viewmodel.dart">
Null ViewModel Example</a>.

   <br>

2. The `converter` parameter

   If you are migrating from `flutter_redux` to `async_redux`, you can keep using `flutter_redux`'s
   good old `converter` parameter:

   ```dart
   @override
     Widget build(BuildContext context) {
       return StoreConnector<int, ViewModel>(
         converter: (store) => ViewModel.fromStore(store),
         builder: (BuildContext context, ViewModel vm) => MyWidget(...),
       );
     }
   ```

   It expects a static factory function that gets a `store` and returns the `ViewModel`.

    ```dart
    class ViewModel {
       final String name;
       final VoidCallback onSave;
    
       ViewModel({
          required this.name,
          required this.onSave,
       });
    
       static ViewModel fromStore(Store<AppState> store) {
          return ViewModel(
             name: store.state,
             onSave: () => store.dispatch(IncrementAction(amount: 1)),
          );
       }
    
       @override
       bool operator ==(Object other) =>
           identical(this, other) ||
           other is ViewModel && runtimeType == other.runtimeType && name == other.name;
    
       @override
       int get hashCode => name.hashCode;
    }
    ```                     

However, the `converter` parameter can also make use of the `Vm` class to avoid having to
create `operator ==` and `hashcode` manually:

    ```dart
    class ViewModel extends Vm {
       final String name;
       final VoidCallback onSave;
    
       ViewModel({
          required this.name,
          required this.onSave,
       }) : super(equals: [name]);
    
       static ViewModel fromStore(Store<AppState> store) {
          return ViewModel(
             name: store.state,
             onSave: () => store.dispatch(IncrementAction(amount: 1)),
          );
       }    
    }
    ```                     

When using the `converter` parameter, it's a bit more difficult to create separate methods for
helping construct your view-model:

    ```dart
    static ViewModel fromStore(Store<AppState> store) {
       return ViewModel(
          name: _name(store),
          onSave: _onSave(store),
       );
    }
    
    static String _name(Store<AppState>) => store.state.user.name;
    
    static VoidCallback _onSave(Store<AppState>) { 
       return () => store.dispatch(SaveUserAction());
    } 
    ```

To see the `converter` parameter in action, please run
<a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_static_view_model.dart">
this example</a>.

#### Will a state change always trigger the StoreConnectors?

Usually yes, but if you want you can order some action not to trigger the `StoreConnector`, by
providing a `notify: false` when dispatching:

```dart
dispatch(MyAction1(), notify: false); 
```

<br>

### Provider

Another good alternative to the `StoreConnector` is using
the <a href="https://pub.dev/packages/provider">Provider</a>
package.

Both the `StoreConnector` (from *async_redux*) and `ReduxSelector` (from *provider_for_redux*)
let you deal with widget rebuilds when the state changes.

You may use `StoreConnector` when you want to have two widgets, one to access the store and prepare
the state to use, and the second as a dumb widget. You may use `ReduxSelector` when you want less
boilerplate, and want to access the store directly from inside a single widget.

Please visit the <a href="https://pub.dev/packages/provider_for_redux">provider_for_redux</a>
package for in-depth explanation and examples on how to use AsyncRedux and Provider together.

<br>

## Processing errors thrown by Actions

AsyncRedux has special provisions for dealing with errors, including observing errors, showing
errors to users, and wrapping errors into more meaningful descriptions.

Let's see an example. Suppose a logout action that checks if there is an internet connection, and
then deletes the database and sets the store to its initial state:

```dart
class LogoutAction extends ReduxAction<AppState> {
  @override
  Future<AppState> reduce() async {
	await checkInternetConnection();
	await deleteDatabase();
	dispatch(NavigateToLoginScreenAction());
	return AppState.initialState();
  }
}
```

In the above code, the `checkInternetConnection()` function checks if there is an
<a href="https://pub.dev/packages/connectivity">internet connection</a>, and if there isn't it
throws an error:

```dart
Future<void> checkInternetConnection() async {
	if (await Connectivity().checkConnectivity() == ConnectivityResult.none)
		throw NoInternetConnectionException();
}
```

All errors thrown by action reducers are sent to the **ErrorObserver**, which you may define during
store creation. For example:

```dart
var store = Store<AppState>(
  initialState: AppState.initialState(),
  errorObserver: MyErrorObserver<AppState>(),
);

class MyErrorObserver<St> implements ErrorObserver<St> {
  @override
  bool observe(Object error, StackTrace stackTrace, ReduxAction<St> action, Store store) {
    print("Error thrown during $action: $error");
    return true;
  }
}                                                                                               
```

If your error observer returns `true`, the error will be rethrown after the `errorObserver`
finishes. If it returns `false`, the error is considered dealt with, and will be "swallowed" (not
rethrown).

<br>

### Giving better error messages

If your reducer throws some error you probably want to collect as much information as possible. In
the above code, if `checkInternetConnection()` throws an error, you want to know that you have a
connection problem, but you also want to know this happened during the logout action. In fact, you
want all errors thrown by this action to reflect that.

The solution is implementing the optional `wrapError(error)` method:

```dart
class LogoutAction extends ReduxAction<AppState> {

  @override
  Future<AppState> reduce() async { ... }

  @override
  Object wrapError(error)
	  => LogoutError("Logout failed.", cause: error);
}
```

Note the `LogoutError` above gets the original error as cause, so no information is lost.

In other words, the `wrapError(error)` method acts as the "catch" statement of the action.

<br>

### User exceptions

To show error messages to the user, make your actions throw an `UserException`, and then wrap your
home-page with `UserExceptionDialog`, below `StoreProvider` and `MaterialApp`:

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context)
	  => StoreProvider<AppState>(
		  store: store,
		  child: MaterialApp(
		    navigatorKey: navigatorKey,
			home: UserExceptionDialog<AppState>(
			  child: MyHomePage(),
			)));
}
```

Note: If you are not using the `home` parameter of the `MaterialApp` widget, you can also put the
`UserExceptionDialog` the `builder` parameter. Please note, if you do that you **must** define the
`NavigateAction.navigatorKey` of the Navigator. Please, see the documentation of the
`UserExceptionDialog.useLocalContext` parameter for more information.

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_show_error_dialog.dart">
Show Error Dialog Example</a>.

**In more detail:**

Sometimes, actions fail because the user provided invalid information. These failings don't
represent errors in the code, so you usually don't want to log them as errors. What you want,
instead, is just warn the user by opening a dialog with some corrective information. For example,
suppose you want to save the user's name, and you only accept names with at least 4 characters:

```dart
class SaveUserAction extends ReduxAction<AppState> {
   final String name;
   SaveUserAction(this.name);

   @override
   Future<AppState> reduce() async {
	 if (name.length < 4) dispatch(ShowDialogAction("Name must have at least 4 letters."));
	 else await saveUser(name);
	 return null;
   }
}
```

Clearly, there is no need to log as an error the user's attempt to save a 3-char name. The above
code dispatches a `ShowDialogAction`, which you would have to wire into a Flutter error dialog
somehow.

However, there's an easier approach. Just throw AsyncRedux's built-in `UserException`:

```dart
class SaveUserAction extends ReduxAction<AppState> {
   final String name;
   SaveUserAction(this.name);

   @override
   Future<AppState> reduce() async {
	 if (name.length < 4) throw UserException("Name must have at least 4 letters.");
	 await saveName(name);
	 return null;
   }
}
```

The special `UserException` error class represents "user errors" which are meant as warnings to the
user, and not as code errors to be logged. By default, if you don't define your own `errorObserver`,
only errors which are not `UserException` are thrown. And if you do define an `errorObserver`,
you'd probably want to replicate this behavior.

In any case, `UserException`s are put into a special error queue, from where they may be shown to
the user, one by one. You may use `UserException` as is, or subclass it, returning title and message
for the alert dialog shown to the user. _Note: In the `Store` constructor you can set the maximum
number of errors that queue can hold._

As explained in the beginning of this section, if you use the build-in error handling you must wrap
your home-page with `UserExceptionDialog`. There, you may pass the `onShowUserExceptionDialog`
parameter to change the default dialog, show a toast, or some other suitable widget:

```dart
UserExceptionDialog<AppState>(
	  child: MyHomePage(),
	  onShowUserExceptionDialog:
		  (BuildContext context, UserException userException) => showDialog(...),
);
``` 

> Note: The `UserExceptionDialog` can display any error widget you want in front of all the others
> on the screen. If this is not what you want, you can easily create your
> own `MyUserExceptionWidget` to intercept the errors and do whatever you want. Start by
> copying `user_exception_dialog.dart` (which contains `UserExceptionDialog` and its `_ViewModel`)
> into another file, and search for the `didUpdateWidget` method. This method will be called each
> time an error is available, and there you can record this information in the widget's own state.
> You can then change the screen in any way you want, according to that saved state, in this
> widget's `build` method.

<br>

### Converting third-party errors into UserExceptions

Third-party code may also throw errors which should not be considered bugs, but simply messages to
be displayed in a dialog to the user.

For example, Firebase my throw some `PlatformException`s in response to a bad connection to the
server. In this case, you can convert this error into a `UserException`, so that a dialog appears to
the user, as already explained above. There are two ways to do that.

The first is to do this conversion in the action itself by implementing the
optional `ReduxAction.wrapError(error)` method:

```dart
class MyAction extends ReduxAction<AppState> {

  @override
  Object? wrapError(error) {
     if ((error is PlatformException) && (error.code == "Error performing get") &&
               (error.message == "Failed to get document because the client is offline."))
        return UserException("Check your internet connection.").addCause(error);
     else 
        return error; 
  }    
```

However, then you'd have to add this to all actions that use Firebase. A better way is doing this
globally by passing a `GlobalWrapError` object to the store:

```dart              
var store = Store<AppState>(
  initialState: AppState.initialState(),
  globalWrapError: MyGlobalWrapError(),
);

class MyGlobalWrapError extends GlobalWrapError {
  @override
  Object? wrap(error, stackTrace, action) {
    if ((error is PlatformException) && (error.code == "Error performing get") &&
          (error.message == "Failed to get document because the client is offline.")) 
        return UserException("Check your internet connection.").addCause(error);
    else 
        return error;
  }
}    
```

The `GlobalWrapError` object will be given all errors. It may then return a `UserException` which
will be used instead of the original exception. Otherwise, it just returns the original `eerror`,
so that it will not be modified. It may also return `null` to disable (swallow) the error.

Note this wrapper is called **after** `ReduxAction.wrapError`, and **before** the `ErrorObserver`.

<br>

### UserExceptionAction

If you want the `UserExceptionDialog` to display some `UserException`, you must throw the exception
from inside an action's `before()` or `reduce()` methods.

However, sometimes you need to create some **callback** that throws an `UserException`. If this
callback is called **outside** an action, the dialog will **not** display the exception. To solve
this, the callback should not throw an exception, but instead call the
provided `UserExceptionAction`, which will then simply throw the exception in its own `reduce()`
method.

The `UserExceptionAction` is also useful even inside of actions, when you want to display an error
dialog to the user, but you don't want to interrupt the action by throwing an exception.

<br>

## Testing

Testing involves waiting for an action to complete its dispatch process,
or for the store state to meet a certain condition. After this, you can verify the current
state or action using the
methods `store.dispatchAndWait`, `store.dispatchAndWaitAll`, `store.waitCondition`,
`store.waitActionCondition`, `store.waitAllActions`, `store.waitActionType`,
`store.waitAllActionTypes`, and `store.waitAnyActionTypeFinishes`. For example:

```dart
// Wait for some action to dispatch and check the state. 
await store.dispatchAndWait(MyAction());
expect(store.state.name, 'John')

// Wait for some action to dispatch, and check for errors in the action status. 
var status = await dispatchAndWait(MyAction());
expect(status.originalError, isA<UserException>());

// Dispatches two actions in SERIES (one after the other).
await dispatchAndWait(SomeAsyncAction());
await dispatchAndWait(AnotherAsyncAction());

// Dispatches two actions in PARALLEL and wait for their TYPES.
expect(store.state.portfolio, ['TSLA']);
dispatch(BuyAction('IBM'));
dispatch(SellAction('TSLA'));
await store.waitAllActionTypes([BuyAction, SellAction]);
expect(store.state.portfolio, ['IBM']);                

// Dispatches two actions in PARALLEL and wait for them.
let action1 = BuyAction('IBM');
let action2 = BuyAction('TSLA');
dispatch(action1);
dispatch(action2);
await store.waitAllActions([action1, action2]);
expect(store.state.portfolio.containsAll('IBM', 'TSLA'), isFalse);

// Another way to dispatch two actions in PARALLEL and wait for them.
await store.dispatchAndWaitAll([BuyAction('IBM'), BuyAction('TSLA')]);
expect(store.state.portfolio.containsAll('IBM', 'TSLA'), isFalse);

// Wait until no actions are in progress.
dispatch(BuyStock('IBM'));
dispatch(BuyStock('TSLA'));  
await waitAllActions([]);                 
expect(state.stocks, ['IBM', 'TSLA']);
     
// Wait for some action of a given type.
dispatch(ChangeNameAction()); 
var action = store.waitActionType(ChangeNameAction);
expect(action, isA<ChangeNameAction>());
expect(action.status.isCompleteOk, isTrue);
expect(store.state.name, 'Bill'); 

// Wait until any action of the given types finishes dispatching.
dispatch(BuyOrSellAction());   
var action = store.waitAnyActionTypeFinishes([BuyAction, SellAction]);  
expect(store.state.portfolio.contains('IBM'), isTrue);

// Wait for some state condition.
expect(store.state.name, 'John')               
dispatch(ChangeNameAction("Bill"));
var action = await store.waitCondition((state) => state.name == "Bill");
expect(action, isA<ChangeNameAction>());
expect(store.state.name, 'Bill');  
```                         

## Testing with StoreTester (deprecated)

For **almost all tests** it's now recommended to use the `Store` directly, as shown in the previous
section.

Older/deprecated code may still use the old `StoreTester`.
Start by creating the store-tester from a store:

```dart
var store = Store<AppState>(initialState: AppState.initialState());
var storeTester = StoreTester.from(store);
```

Or else, creating it directly from `AppState`:

```dart
var storeTester = StoreTester<AppState>(initialState: AppState.initialState());
```

Then, dispatch some action, wait for it to finish, and check the resulting state:

```dart
storeTester.dispatch(SaveNameAction("Mark"));
TestInfo<AppState> info = await storeTester.wait(SaveNameAction);
expect(info.state.name, "Mark");
```

or

```dart
TestInfo<AppState> info = storeTester.dispatchAndWait(SaveNameAction("Mark")); 
expect(info.state.name, "Mark");
```

The variable `info` above will contain information about after the action reducer finishes
executing, **no matter if the reducer is sync or async**.

The `TestInfo` instance contains the following:

* `state`: The store state.
* `action`: The dispatched Action that resulted in that state.
* `ini`: A boolean which indicates true if this info represents the "initial" state right before the
  action is dispatched, or false it represents the "end" state right after the action finishes
  executing.
* `dispatchCount`: The number of dispatched actions so far.
* `reduceCount`: The number of reduced states so far.
* `errors`: The `UserException`s the store was holding when the information was gathered.

While the above example demonstrates the testing of a simple action, real-world apps have actions
that dispatch other actions. You may use different `StoreTester` methods to check if the expected
actions are dispatched, and test their intermediary states.

Let's see all the available methods of the `StoreTester`:

1. `Future<TestInfo> wait(Type actionType)`

   Expects **one action** of the given type to be dispatched, and waits until it finishes. Returns
   the info after the action finishes. Will fail with an exception if an unexpected action is seen.

2. `Future<TestInfo> waitUntil(Type actionType)`

   Runs until an action of the given type is dispatched, and then waits until it finishes. Returns
   the info after the action finishes.
   **Ignores other** actions types.

3. `Future<TestInfo> waitUntilAll(List<Type> actionTypes)`

   Runs until all actions of the given types are dispatched and finish, in any order. Returns a list
   with all info until the last action finishes. **Ignores other** actions types.

4. `Future<TestInfo> waitUntilAllGetLast(List<Type> actionTypes)`

   Runs until all actions of the given types are dispatched and finish, in any order. Returns the
   info after they all finish. **Ignores other** actions types.

5. `Future<TestInfo> waitUntilAction(ReduxAction action)`

   Runs until the exact given action is dispatched, and then waits until it finishes. Returns the
   info after the action finishes. **Ignores other** actions.

6. `Future<TestInfo> dispatchAndWait(ReduxAction action)`

   Dispatches the given action, then waits until it finishes. Returns the
   info after the action finishes. **Ignores other** actions.

7. `Future<TestInfo> waitAllGetLast(List<Type> actionTypes, {List<Type> ignore})`

   Runs until **all** given actions types are dispatched, **in order**. Waits until all of them are
   finished. Returns the info after all actions finish. Will fail with an exception if an unexpected
   action is seen, or if any of the expected actions are dispatched in the wrong order. To ignore
   some actions, pass them to the `ignore` list.

8. `Future<TestInfo> waitAllUnorderedGetLast(List<Type> actionTypes, {List<Type> ignore})`

   Runs until **all** given actions types are dispatched, in **any order**. Waits until all of them
   are finished. Returns the info after all actions finish. Will fail with an exception if an
   unexpected action is seen. To ignore some actions, pass them to the `ignore` list.

9. `Future<TestInfoList> waitAll(List<Type> actionTypes, {List<Type> ignore})`

   The same as `waitAllGetLast`, but instead of returning just the last info, it returns a list with
   the end info for each action. To ignore some actions, pass them to the `ignore` list.

10. `Future<TestInfoList> waitAllUnordered(List<Type> actionTypes, {List<Type> ignore})`

    The same as `waitAllUnorderedGetLast`, but instead of returning just the last info, it returns a
    list with the end info for each action. To ignore some actions, pass them to the `ignore` list.

11. `Future<TestInfoList<St>> waitCondition(StateCondition<St> condition, {bool testImmediately = true, bool ignoreIni = true})`

    Runs until the predicate function `condition` returns true. This function will receive each
    testInfo, from where it can access the state, action, errors etc. When `testImmediately` is
    true (the default), it will test the condition immediately when the method is called. If the
    condition is true, the method will return immediately, without waiting for any actions to be
    dispatched. When `testImmediately` is false, it will only test the condition once an action is
    dispatched. Only END states will be received, unless you pass `ignoreIni` as false. Returns a
    list with all info until the condition is met.

12. `Future<TestInfo<St>> waitConditionGetLast(StateCondition<St> condition, {bool testImmediately = true, bool ignoreIni = true})`

    Runs until the predicate function `condition` returns true. This function will receive each
    testInfo, from where it can access the state, action, errors etc. When `testImmediately` is
    true (the default), it will test the condition immediately when the method is called. If the
    condition is true, the method will return immediately, without waiting for any actions to be
    dispatched. When `testImmediately` is false, it will only test the condition once an action is
    dispatched. Only END states will be received, unless you pass `ignoreIni` as false. Returns the
    info after the condition is met.

13. `Future<TestInfoList<St>> waitUntilError({Object error, Object processedError})`

    Runs until after an action throws an error of this exact type, or this exact error (using
    equals). You can also, instead, define `processedError`, which is the error after wrapped by the
    action's `wrapError()` method. Returns a list with all info until the error condition is met.

14. `Future<TestInfo> waitUntilErrorGetLast({Object error, Object processedError})`

    Runs until after an action throws an error of this exact type, or this exact error (using
    equals). You can also, instead, define `processedError`, which is the error after wrapped by the
    action's `wrapError()` method. Returns the info after the condition is met.

15. `Future<TestInfo<St>> dispatchState(St state)`

    Dispatches an action that changes the current state to the one provided by you. Then, runs until
    that action is dispatched and finished (ignoring other actions). Returns the info after the
    action finishes, containing the given state.

Some of the methods above return a list of type `TestInfoList`, which contains the step by step
information of all the actions. You can then query for the actions you want to inspect. For example,
suppose an action named `IncrementAndGetDescriptionAction` calls another 3 actions. You can assert
that all actions are called in order, and then get the state after each one of them have finished,
all at once:

```dart
var storeTester = StoreTester<AppState>(initialState: AppState.initialState());
expect(storeTester.state.counter, 0);
expect(storeTester.state.description, isEmpty);

storeTester.dispatch(IncrementAndGetDescriptionAction());

TestInfoList<AppState> infos = await storeTester.waitAll([
   IncrementAndGetDescriptionAction,
   BarrierAction,
   IncrementAction,
   BarrierAction,
]);

// Modal barrier is turned on (first time BarrierAction is dispatched).
expect(infos.get(BarrierAction, 1).state.waiting, true);

// While the counter was incremented the barrier was on.
expect(infos[IncrementAction].waiting, true);

// Then the modal barrier is dismissed (second time BarrierAction is dispatched).
expect(infos.get(BarrierAction, 2).state.waiting, false);

// In the end, counter is incremented, description is created, and barrier is dismissed.
var info = infos[IncrementAndGetDescriptionAction];
expect(info.state.waiting, false);
expect(info.state.description, isNotEmpty);
expect(info.state.counter, 1);
```

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/test/main_before_and_after_STATE_test.dart">
Testing with the Store Listener</a>.

Also,
the <a href="https://github.com/marcglasberg/async_redux/blob/master/test/store_tester_test.dart">
tests of the StoreTester</a> can also serve as examples.

**Important:** The `StoreTester` has access to the current store state via `StoreTester.state`, but
you should not try to assert directly from this state. This would seem to work most of the time, but
by the time you do the assert, the state could already have been changed by some other action. To
avoid that, always assert from the `info` you get from the `StoreTester` methods, which is
guaranteed to be the one right after your *wait condition* is achieved. For example:

```dart
// This is right:
TestInfo<AppState> info = await storeTester.wait(SaveNameAction);
expect(info.state.name, "Mark");

// This is wrong:
await storeTester.wait(SaveNameAction);
expect(storeTester.state.name, "Mark");
```       

However, to help you further reduce your test boilerplate, the last `info`
obtained from the most recent wait condition is saved into a variable called `storeTester.lastInfo`:

```dart
// This:
TestInfo<AppState> info = await storeTester.wait(SaveNameAction);
expect(info.state.name, "Mark");

// Is the same as this:
await storeTester.wait(SaveNameAction);
expect(storeTester.lastInfo.state.name, "Mark");
```       

<br>

### Testing the StoreConnector's View-model

To test the view-model generated by a `VmFactory`, use `createFrom` and pass it the
`store` and the `factory`. Note this method must be called in a recently
created factory, as it can only be called once per factory instance.

The method will return the **view-model**, which you can use to:

* Inspect the view-model properties directly, or

* Call any of the view-model callbacks. If the callbacks dispatch actions,
  you use `await store.waitAllActions([])`,
  or `await store.waitActionType(MyAction)`,
  or `await store.waitAllActionTypes([MyAction, OtherAction])`,  
  or `await store.waitAnyActionTypeFinishes([MyAction, OtherAction])`,  
  or `await store.waitCondition((state) => ...)`,
  or if necessary you can even record all dispatched actions and state changes
  with `Store.record.start()` and `Store.record.stop()`.

Example:

```
var store = Store(initialState: User("Mary"));
var vm = Vm.createFrom(store, MyFactory());

// Checking a view-model property.
expect(vm.user.name, "Mary");

// Calling a view-model callback and waiting for the action to finish.
vm.onChangeNameTo("Bill"); // Dispatches SetNameAction("Bill").
await store.waitActionType(SetNameAction);
expect(store.state.name, "Bill");

// Calling a view-model callback and waiting for the state to change.
vm.onChangeNameTo("Bill"); // Dispatches SetNameAction("Bill").
await store.waitCondition((state) => state.name == "Bill");
expect(store.state.name, "Bill");
```

#### Testing the StoreConnector's onInit and onDispose

Suppose you want to start polling information when your user enters a particular screen, and stop
when the user leaves it. This could be your `StoreConnector`:

```dart
class MyScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) => StoreConnector<AppState, _Vm>(
        vm: () => _Factory(),
        onInit: _onInit,
        onDispose: _onDispose,
        builder: (context, vm) => MyWidget(...),
      );

  void _onInit(Store<AppState> store) => store.dispatch(PollInformationAction(true));
  void _onDispose(Store<AppState> store) => store.dispatch(PollInformationAction(false));
}
```

You want to test that `onInit` and `onDispose` above dispatch the correct action to start and stop
polling. You may achieve this by creating a widget test, entering and leaving the screen, and then
using the `StoreTester` to check that the actions were dispatched.

Instead, you may simply use the `ConnectorTester`, which you can access from the `StoreTester`:

```dart
var storeTester = StoreTester(...);
var connectorTester = storeTester.getConnectorTester(MyScreen());

connectorTester.runOnInit();
var info = await storeTester.waitUntil(PollInformationAction);
expect((info.action as PollInformationAction).start, true);

connectorTester.runOnDispose();
info = await storeTester.waitUntil(PollInformationAction);
expect((info.action as PollInformationAction).start, false);
```

<br>

### Mocking actions and reducers

To mock an action and its reducer, create a `MockStore` instead of a regular `Store`.

The `MockStore` has a `mocks` parameter which is a map where the keys are action types, and the
values are the mocks. For example:

```dart
var store = MockStore<AppState>(
  initialState: initialState,  
  mocks: {
     MyAction1 : ...
     MyAction2 : ...
     ...
  },
);
```       

There are 5 different ways to define mocks:

1. Use `null` to disable dispatching the action of a certain type:

    ```dart
    mocks: {
       MyAction : null
    }
    ```       

2. Use a `MockAction<St>` instance to dispatch this mock action instead, and provide the **original
   action** as a getter to the mock action.

    ```dart                        
    class MyAction extends ReduxAction<AppState> {
      String url;
      MyAction(this.url);
      Future<AppState> reduce() => get(url);
    }      

    class MyMockAction extends MockAction<AppState> {  
      Future<AppState> reduce() async {                  
        String url = (action as MyAction).url;
        if (url == 'https://example.com') return 123;
        else if (url == 'https://flutter.io') return 345;
        else return 678;
      }
    }
    ```

    ```dart    
    mocks: {
       MyAction : MyMockAction()
    }
    ```       

3. Use a `ReduxAction<St>` instance to dispatch this mock action instead.

    ```dart                        
    class MyAction extends ReduxAction<AppState> {
      String url;            
      MyAction(this.url);
      Future<AppState> reduce() => get(url);
    }
    
    class MyMockAction extends ReduxAction<AppState> {  
      Future<AppState> reduce() async => 123;
    }
    ```

    ```dart    
    mocks: {
       MyAction : MyMockAction()
    }
    ```       

4. Use a `ReduxAction<St> Function(ReduxAction<St>)` to create a mock from the original action.

    ```dart                        
    class MyAction extends ReduxAction<AppState> {
      String url;        
      MyAction(this.url);
      Future<AppState> reduce() => get(url);
    }
    
    class MyMockAction extends MockAction<AppState> {
      String url;
      MyMockAction(this.url);  
      Future<AppState> reduce() async {                     
        if (url == 'https://example.com') return 123;
        else if (url == 'https://flutter.io') return 345;
        else return 678;
      }
    }
    ```

    ```dart   
    mocks: {
       MyAction : (MyAction action) => MyMockAction(action.url)
    }
    ```       

5. Use a `St Function(ReduxAction<St>, St)`
   or `Future<St> Function(ReduxAction<St>, St)`
   to modify the state directly.

    ```dart                        
    class MyAction extends ReduxAction<AppState> {
      String url;        
      MyAction(this.url);
      Future<AppState> reduce() => get(url);
    }
    ```

    ```dart   
    mocks: {
       MyAction : (MyAction action, AppState state) async {
          if (action.url == 'https://example.com') return 123;
          else if (action.url == 'https://flutter.io') return 345;
          else return 678;
       }
    }
    ```       

You can also change the mocks after a store is created, by using the following methods of
the `MockStore` and `StoreTester` classes:

```dart
MockStore<St> addMock(Type actionType, dynamic mock);
MockStore<St> addMocks(Map<Type, dynamic> mocks);
MockStore<St> clearMocks();
```

<br>

### Testing UserExceptions

Since `UserException`s don't represent bugs in the code, AsyncRedux put them into the
store's `errors` queue, and then swallows them. This is usually what you want during production,
where errors from this queue are shown in a dialog to the user. But it may or may not be what you
want during tests.

In tests there are two possibilities:

1. You are testing that some `UserException` is thrown. For example, you want to test that users are
   warned if they typed letters in some field that only accepts numbers. To that end, your test
   would dispatch the appropriate action, and then check if the `errors` queue now contains
   an `UserException` with some specific error message.

2. You are testing some code that should **not** throw any exceptions. If the test has thrown an
   exception it means the test has failed, and the exception should show up in the console, for
   debugging. However, this won't work if when test throws an `UserException` it simply go to
   the `errors` queue. If this happens, the test will continue running, and may even pass. The only
   way to make sure no errors were thrown would be asserting that the `errors` queue is still empty
   at the end of the test. This is even more problematic if the unexpected `UserException` is thrown
   inside a `before()` method. In this case it will prevent the reducer to run, and the test will
   probably fail with wrong state but no errors in the console.

The solution is to use the `shouldThrowUserExceptions` parameter in the `StoreTester` constructor.

Pass `shouldThrowUserExceptions` as `true`, and all errors will be thrown and not swallowed,
including `UserException`s. Use this in all tests that should throw no errors:

```dart
var storeTester = StoreTester<AppState>(
                     initialState: AppState.initialState(), 
                     shouldThrowUserExceptions: true);
```

Pass `shouldThrowUserExceptions` as false (the default)
when you are testing code that should indeed throw `UserExceptions`. These exceptions will then
silently go to the `errors` queue, where you can assert they exist and have the right error
messages:

```dart
storeTester.dispatch(MyAction());
TestInfo info = await storeTester.waitAllGetLast([MyAction]);
expect(info.errors.removeFirst().msg, "You can't do this.");
```

<br>

### Test files

If you want your tests to be comprehensive you should probably have 3 different types of test for
each widget:

1. **State Tests** — Test the state of the app, including actions/reducers. This type of tests make
   use of the `StoreTester` described above.

2. **Connector Tests** — Test the connection between the store and the "dumb-widget". In other words
   it tests the "connector-widget" and the "view-model".

3. **Presentation Tests** — Test the UI. In other words it tests the "dumb-widget", making sure that
   the widget displays correctly depending on the parameters you use in its constructor. You pass in
   the data the widget requires in each test for rendering, and then writes assertions against the
   rendered output. Think of these tests as "pure function tests" of our UI. It also tests that the
   callbacks are called when necessary.

For example, suppose you have the counter app
shown <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_increment_async.dart">
here</a>. Then:

* The **state test** could create a store with count `0` and description empty, and then
  dispatch `IncrementAction` and expect the count to become `1`. Then it could test
  dispatching `IncrementAndGetDescriptionAction` alters the count to `2`
  and the description to some non-empty string.

* The **connector test** would create a store and a page with the `MyHomePageConnector` widget. It
  would then access the `MyHomePage` and make sure it gets the expected info from the store, and
  also that the expected `IncrementAndGetDescriptionAction` is dispatched when the "+" button is
  tapped.

* The **presentation test** would create the `MyHomePage` widget, pass `counter:0`
  and `description:"abc"` parameters in its constructor, and make sure they appear in the screen as
  expected. It would also test that the callback is called when the "+" button is tapped.

Since each widget will have a bunch of related files, you should have some consistent naming
convention. For example, if some dumb-widget is called `MyWidget`, its file could
be `my_widget.dart`. Then the corresponding connector-widget could be `MyWidgetConnector`
in `my_widget_CONNECTOR.dart`. The three corresponding test files could be
named `my_widget_STATE_test.dart`, `my_widget_CONNECTOR_test.dart`
and `my_widget_PRESENTATION_test.dart`. If you don't like this convention use your own,
but just choose one early and stick to it.

<br>

## Route Navigation

AsyncRedux comes with a `NavigateAction` which you can dispatch to navigate your Flutter app. For
this to work, during app initialization you must create a navigator key and then inject it into the
action:

```dart
final navigatorKey = GlobalKey<NavigatorState>();

void main() async {
  NavigateAction.setNavigatorKey(navigatorKey);
  ...
}
```

You must also use this same navigator key in your `MaterialApp`:

```dart
return StoreProvider<AppState>(
  store: store,
  child: MaterialApp(
	  ...
	  navigatorKey: navigatorKey,
	  initialRoute: '/',
	  onGenerateRoute: ...
	  ),
);
```

Then, use the action as needed:

```dart                
// Most Navigator methods are available. 
// For example pushNamed: 
dispatch(NavigateAction.pushNamed("myRoute"));
```

Note: Don't ever save the current route in the store. This will create all sorts of problems. If you
need to know the route you're in, you may use this static method provided by `NavigateAction`:

```dart
String routeName = NavigateAction.getCurrentNavigatorRouteName(context);
```     

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_navigate.dart">
Navigate Example</a>.

### Testing with `NavigateAction`

You can test navigation by asserting navigation types, route names etc. This is useful for verifying
app flow in unit tests, instead of widget or driver tests.

For example:

```dart         
var navigateAction = actions.get(NavigateAction).action as NavigateAction;
expect(navigateAction.type, NavigateType.pushNamed);
expect((navigateAction.details as NavigatorDetails_PushNamed).routeName, "myRoute");
```

<br>

## Events

In a real Flutter app it's not practical to assume that a Redux store can hold all the
application state. Widgets like `TextField` and `ListView` make use of controllers, which hold
state, and the store must be able to work alongside these. For example, in response to the
dispatching of some action you may want to clear the text-field, or you may want to scroll the
list-view to the top. Even when no controllers are involved, you may want to execute some one-off
processes, like opening a dialog or closing the keyboard, and it's not obvious how to do that in
vanilla Redux.

AsyncRedux solves these problems by introducing the concept of "events". The naming convention is
that Events are named with the `Evt` suffix.

Boolean events can be created like this:

```dart
var clearTextEvt = Event();
```

But you can have events with payloads of any other data type. For example:

```dart
var changeTextEvt = Event<String>("Hello");
var myEvt = Event<int>(42);
```

Events may be put into the store state in their "spent" state, by calling its `spent()` constructor.
For example, while creating the store initial-state:

```dart
static AppState initialState() {
  return AppState(
	clearTextEvt: Event.spent(),
	changeTextEvt: Event<String>.spent(),
}
```

And then events may be passed down by the `StoreConnector` to some `StatefulWidget`, just like any
other state:

```dart
class MyConnector extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
	return StoreConnector<AppState, ViewModel>(
	  model: ViewModel(),
	  builder: (BuildContext context, ViewModel vm) => MyWidget(
		initialText: vm.initialText,
		clearTextEvt: vm.clearTextEvt,
		changeTextEvt: vm.changeTextEvt,
		onClear: vm.onClear,
	  ));
  }
}

class ViewModel extends BaseModel<AppState> {
  ViewModel();

  String initialText;
  Event clearTextEvt;
  Event<String> changeTextEvt;

  ViewModel.build({
	required this.initialText,
	required this.clearTextEvt,
	required this.changeTextEvt,
  }) : super(equals: [initialText, clearTextEvt, changeTextEvt]);

  @override
  ViewModel fromStore() => ViewModel.build(
		initialText: state.initialText,
		clearTextEvt: state.clearTextEvt,
		changeTextEvt: state.changeTextEvt,
		onClear: () => dispatch(ClearTextAction()),
	  );
}

class ClearTextAction extends ReduxAction<AppState> {
  @override
  AppState reduce() => state.copy(changeTextEvt: Event());
}

class ChangeTextAction extends ReduxAction<AppState> {
  String newText;
  ChangeTextAction(this.newText);

  @override
  AppState reduce() => state.copy(changeTextEvt: Event<String>(newText));
}
```

This is how it differs: The dumb-widget will "consume" the event in its `didUpdateWidget`
method, and do something with the event payload:

```dart
@override
void didUpdateWidget(MyWidget oldWidget) {
  super.didUpdateWidget(oldWidget);
  consumeEvents();
}

void consumeEvents() {
  if (widget.clearTextEvt.consume()) { // Do something }

  var payload = widget.changeTextEvt.consume();
  if (payload != null) { // Do something }
}
```

The `evt.consume()` will return the payload once, and then that event is considered "spent".

In more detail, if the event **has no value and no generic type**, then it's a boolean event. This
means `evt.consume()` returns **true** once, and then **false** for subsequent calls. However, if
the event **has value or some generic type**, then `Event.consume()` returns the **value** once, and
then **null** for subsequent calls.

So, for example, if you use a `controller` to hold the text in a `TextField`:

```dart
void consumeEvents() {

	if (widget.clearTextEvt.consume())
	  WidgetsBinding.instance.addPostFrameCallback((_) {
		if (mounted) controller.clear();
	  });

	String newText = widget.changeTextEvt.consume();
	if (newText != null)
	  WidgetsBinding.instance.addPostFrameCallback((_) {
		if (mounted) controller.value = controller.value.copyWith(text: newText);
	  });
  }
```

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_event_redux.dart">
Event Example</a>.

<br>

### Can I put mutable events into the store state?

Events are mutable, and store state is supposed to be immutable. Won't this create problems? No!
Don't worry, events are used in a contained way, and were crafted to play well with the Redux
infrastructure. In special, their `equals()` and `hashcode()` methods make sure no unnecessary
widget rebuilds happen when they are used as prescribed.

You can think of events as piggybacking in the Redux infrastructure, and not belonging to the store
state. You should just remember **not to persist them** when you persist the store state.

<br>

### When should I use events?

The short answer is that you'll know it when you see it. When you want to do something, and it's not
obvious how to do it by changing regular store state, it's probably easy to solve it if you try
using events instead.

However, we can also give these guidelines:

1. You may use regular store state to pass constructor parameters to both stateless and stateful
   widgets.
2. You may use events to change the internal state of stateful widgets, after they are built.
3. You may use events to make one-off changes in controllers.
4. You may use events to make one-off changes in other implicit state like the open state of dialogs
   or the keyboard.

<br>

### Advanced event features

There are some advanced event features you may not need, but you should know they exist:

1. Methods `isSpent`, `isNotSpent` and `state`

   Methods `isSpent` and `isNotSpent` tell you if an event is spent or not, without consuming the
   event. Method `state` returns the event payload, without consuming the event.

2. Constructor `Event.map(Event<dynamic> evt, T Function(dynamic) mapFunction)`

   This is a convenience factory to create an event which is transformed by some function that,
   usually, needs the store state. You must provide the event and a map-function. The map-function
   must be able to deal with the spent state (`null` or `false`, accordingly).

   For example, if `state.indexEvt = Event<int>(5)` and you must get a user from it:

   ```dart
   var mapFunction = (index) => index == null ? null : state.users[index];
   Event<User> userEvt = MappedEvent<int, User>(state.indexEvt, mapFunction);
   ```  

3. Constructor `Event.from(Event<T> evt1, Event<T> evt2)`

   This is a convenience factory method to create `EventMultiple`, a special type of event which
   consumes from more than one event. If the first event is not spent, it will be consumed, and the
   second will not. If the first event is spent, the second one will be consumed. So, if both events
   are NOT spent, the method will have to be called twice to consume both. If both are spent,
   returns `null`.

4. Method `static T consumeFrom<T>(Event<T> evt1, Event<T> evt2)`

   This is a convenience static method to consume from more than one event. If the first event is
   not spent, it will be consumed, and the second will not. If the first event is spent, the second
   one will be consumed. So, if both events are NOT spent, the method will have to be called twice
   to consume both. If both are spent, returns `null`. For example:

    ```dart
    String getMessageEvt() => Event.consumeFrom(firstMsgEvt, secondMsgEvt);
    ```

<br>

## Progress indicators

A **progress indicator** is a visual indication that some important process is taking some time to
finish (and will hopefully finish soon). For example:

* A save button that displays a `CircularProgressIndicator` while some info is saving.

* A `Text("Please wait...")` that is displayed in the center of the screen while some info is being
  calculated.

* A <a href="https://pub.dev/packages?q=shimmer">shimmer</a> that is displayed as a placeholder
  while some widget info is being downloaded.

* A modal barrier that prevents the user from interacting with the screen while some info is loading
  or saving.

<br>

The easiest way to show a progress indicator is to use `store.isWaiting(MyAction)`,
where `MyAction` is the async action you are waiting for. This works well for the majority of cases.

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_show_spinner.dart">
Show Spinner Example</a>. When you press the "+" button, it dispatches an increment action that
takes 2 seconds to finish. Meanwhile, a spinner is shown in the button, and the counter text gets
grey.

In [Before and After the Reducer](#before-and-after-the-reducer) section I show how to manually
create a boolean flag that is used to add or remove a modal barrier in the screen (see the
code <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_before_and_after.dart">
here</a>). This will work in some rare complex cases where `store.isWaiting()` is not enough.

However, keeping track of many such boolean flags may be difficult to do.
If you need help with this problem, an option is using the built-in classes `WaitAction` and `Wait`.

For this to work, your store state must have a `Wait` field named `wait`, and then your state class
must have a `copy` or a `copyWith` method which copies this field as a named parameter. For example:

```dart
@immutable
class AppState {
  final Wait wait;
  ...  
  AppState({this.wait, ...});
  AppState copy({Wait wait, ...}) => AppState(wait: wait ?? this.wait, ...);
  }
```

Then, when you want to start waiting, simply dispatch a `WaitAction`
and pass it some immutable object to act as a flag. When you finish waiting, just remove the flag.
For example:

```dart
dispatch(WaitAction.add("my flag")); // To add a flag.   
dispatch(WaitAction.remove("my flag")); // To remove a flag.   
```            

When you are using the state:

* `state.wait.isWaitingAny` returns `true` if there's any waiting whatsoever.
* `state.wait.isWaiting(flag)` returns `true` if you are waiting for a specific `flag`
* `state.wait.isWaiting(flag, ref: reference)` returns `true` if you are waiting for a
  specific `reference` of the `flag`.
* `state.wait.isWaitingForType<T>()` returns `true` if you are waiting for any flag of type `T`.

<br>

The flag can be any convenient **immutable object**, like a URL, a user id, an index, an enum, a
String, a number, or other.

As an example, if we want to replace the `store.isWaiting()` method with the `Wait` object, we
could do this: Suppose that a button dispatches a `LoadAction` to load some text. You can make the
button show a progress indicator while the text is being loaded, and show the text when it's done:

```dart
class LoadAction extends ReduxAction<AppState> {

  Future<AppState> reduce() async {    
    var newText = await loadText(); 
    return state.copy(text: newText);
  }
  
  void before() => dispatch(WaitAction.add(this));
  void after() => dispatch(WaitAction.remove(this));
}
```

Then, in the button:

```dart
if (wait.isWaitingForType<LoadAction>()) { // Show the button as disabled }
else { // Show the button as enabled }
``` 

Note: You may also define a **mixin** to implement the waiting:

```dart
mixin WithWaitState implements ReduxAction<AppState> {
  void before() => dispatch(WaitAction.add(this));
  void after() => dispatch(WaitAction.remove(this));
}  

class LoadAction extends ReduxAction<AppState> with WithWaitState {
  Future<AppState> reduce() async {    
    var newText = await loadText(); 
    return state.copy(text: newText);
  }
}
```

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_wait_action_simple.dart">
Wait Action Simple Example</a>
(which is similar to the
<a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_before_and_after.dart">
Before and After example</a> but using the built-in `WaitAction`). It uses the action itself as the
flag, by passing `this`.

<br>

A more advanced example is
the <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_wait_action_advanced_1.dart">
Wait Action Advanced 1 Example</a>. Here, 10 buttons are shown. When a button is clicked it will be
replaced by a downloaded text description. Each button shows a progress indicator while its
description is downloading. Also, the screen title shows the text `"Downloading..."` if any of the
buttons is currently downloading.

![](https://github.com/marcglasberg/async_redux/blob/master/example/lib/images/waitAction.png)

The flag in this case is simply the index of the button, from `0` to `9`:

```dart                                                        
int index;
void before() => dispatch(WaitAction.add(index));
void after() => dispatch(WaitAction.remove(index));
```                            

In the `ViewModel`, just as before, if there's any waiting, then `state.wait.isWaitingAny` will
return `true`. However, now you can check each button wait flag separately by its index.
`state.wait.isWaiting(index)` will return `true` if that specific button is waiting.

Note: If necessary, you can clear all flags by doing `dispatch(WaitAction.clear())`.

<br> 

If you fear your flag may conflict with others, you can also add a "namespace", by further dividing
flags into references. This can be seen in
the <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_wait_action_advanced_2.dart">
Wait Action Advanced 2 Example</a>:

```dart
void before() => dispatch(WaitAction.add("button-download", ref: index));
void after() => dispatch(WaitAction.remove("button-download", ref: index));
```                            

Now, to check a button's wait flag, you must pass both the flag and the reference:
`state.wait.isWaiting("button-download", ref: index)`.

Note: If necessary, you can clear all references of that flag by
doing `dispatch(WaitAction.clear("button-download"))`.

You can also pass a delay to `WaitAction.add()` and `WaitAction.remove()` methods. Please refer to
their method documentation for more information.

### Using BuiltValue, Freezed, or other similar code generator packages

In case you use
<a href="https://pub.dev/packages/built_value">built_value</a>
or <a href="https://pub.dev/packages/freezed">freezed</a> packages, the `WaitAction` works
out-of-the-box with them. In both cases, you don't need to create the `copy` or `copyWith` methods
by hand. But you still need to add the `Wait` object to the store state as previously described.

If you want to further customize `WaitAction` to work with other packages, or to use the `Wait`
object in different ways, you can do so by injecting your custom reducer into `WaitAction.reducer`
during your app's initialization. Refer to
the `WaitAction` <a href="https://github.com/marcglasberg/async_redux/blob/master/lib/src/wait_action.dart">
documentation</a> for more information.

<br>

## Showing a RefreshIndicator while some action is running

In a real Flutter app it's also the case that some Widgets ask for futures that complete when some
async process is done.

The `dispatchAndWait()` function returns a future which completes as soon as the action is done.

This is an example using the `RefreshIndicator` widget:

```dart
Future<void> downloadStuff() => dispatchAndWait(DownloadStuffAction());

return RefreshIndicator(
    onRefresh: downloadStuff;
    child: ListView(...),
```                 

Or, if you have multiple actions you can use `dispatchAndWaitAll`.

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_dispatch_future.dart">
Dispatch Future Example</a>.

<br>

## Waiting until the state meets a certain condition

The `waitCondition` method from the `Store` class lets you create **futures** that complete when the
store state meets a certain condition:

```dart
/// Returns a future which will complete when the given condition is true.
/// The condition can access the state. You may also provide a
/// timeoutInSeconds, which by default is null (never times out).
Future<void> waitCondition(
   bool Function(St) condition, {
   int timeoutInSeconds
   })
```

For example:

```dart
class SaveAppointmentAction extends ReduxAction<AppState> {  
  final Appointment appointment;
  
  SaveAppointmentAction(this.appointment);      

  @override
  Future<AppState> reduce() {    
    dispatch(CreateCalendarIfNecessaryAction());    
    await store.waitCondition((state) => state.calendar != null);
    return state.copy(calendar: state.calendar.copyAdding(appointment));
  }
}
```         

The above action needs to add an appointment to a calendar, but it can only do that if the calendar
already exists. Maybe the calendar already exists, but if not, creating a calendar is a complex
async process, which may take some time to complete.

To that end, the action dispatches an action to create the calendar if necessary, and then use
the `store.waitCondition()` method to wait until a calendar is present in the state. Only then it
will add the appointment to the calendar.

<br>

## State Declaration

While your main state class, usually called `AppState`, may be simple and contain all the state
directly, in a real world application you will probably want to create many state classes and add
them to the main state class. For example, if you have some state for the login, some user related
state, and some *todos* in a To-Do app, you can organize it like this:

```dart
@immutable
class AppState {

  final LoginState loginState;
  final UserState userState;
  final TodoState todoState;

 AppState({
	this.loginState,
	this.userState,
	this.todoState,
  });

  AppState copy({
	LoginState loginState,
	UserState userState,
	TodoState todoState,
  }) {
	return AppState(
	  login: loginState ?? this.loginState,
	  user: userState ?? this.userState,
	  todo: todoState ?? this.todoState,
	);
  }

  static AppState initialState() =>
	AppState(
	  loginState: LoginState.initialState(),
	  userState: UserState.initialState(),
	  todoState: TodoState.initialState());

  @override
  bool operator ==(Object other) =>
	identical(this, other) || other is AppState && runtimeType == other.runtimeType &&
	  loginState == other.loginState && userState == other.userState && todoState == other.todoState;

  @override
  int get hashCode => loginState.hashCode ^ userState.hashCode ^ todoState.hashCode;
}
```

All of your state classes may follow this pattern. For example, the `TodoState` could be like this:

```dart  
import 'package:flutter/foundation.dart'; 
import 'package:collection/collection.dart';

class TodoState {    
  
  final List<Todo> todos;            

  TodoState({this.todos});

  TodoState copy({List<Todo> todos}) {
    return TodoState(          
      todos: todos ?? this.todos);
  }             
  
  static TodoState initialState() => TodoState(todos: const []);               
  
  @override
  bool operator ==(Object other) {          
    return identical(this, other) || other is TodoState && runtimeType == other.runtimeType && 
      listEquals(todos, other.todos);
  }    
  
  @override
  int get hashCode => const ListEquality.hash(todos);
}
```

<br>

### Selectors

Your connector-widgets usually have a view-model that goes into the store and selects the part of
the store the widget needs. If you have some "selecting logic" that you use in different places, you
may create a "selector". Selectors may be put in separate files, or they may be put into state
classes, as static methods. For example, the `TodoState` class above could contain a selector to
filter out some todos:

```dart
static List<Todo> selectTodosForUser(AppState state, User user)
   => state.todoState.todos.where((todo) => (todo.user == user)).toList();
```       

<br>

### Cache (Reselectors)

Suppose you use a `ListView.builder` to display user names as list items. In your `StoreConnector`,
you could create a `ViewModel` that, given the item index, returns a user name:

```dart
state.users[index].name;
```       

But now suppose you want to display only the users with names that start with the letter `A`. You
could filter the user list to remove all other names, like this:

```dart
state.users.where((user)=>user.name.startsWith("A")).toList()[index].name;
```                                                                                           

This works, but will filter the list repeatedly, once for each index. This is not a problem for
small lists, but will become slow if the list contains thousands of users.

The solution to this problem is caching the filtered list. To that end, you can use the "reselect"
functionality provided by AsyncRedux.

First, create a selector that returns the information you need:

```dart
static List<User> selectUsersWithNamesStartingWith(AppState state, {String text})
   => state.users.where((user)=>user.name.startsWith(text)).toList();
```    

And then use it in the ViewModel:

```dart
selectUsersWithNamesStartingWith(state, text: "A")[index].name;
```                                                                                           

Next, we have to modify the selector so that it caches the filtered list. AsyncRedux provides a few
global functions which you can use, depending on the number of states, and the number of parameters
your selector needs.

In this example, we have a single state and a single parameter, so we're going to use the `cache1_1`
method:

```dart                                                    
static List<User> selectUsersWithNamesStartingWith(AppState state, {String text})
   => _selectUsersWithNamesStartingWith(state)(text);

static final _selectUsersWithNamesStartingWith = cache1_1(
        (AppState state) 
           => (String text) 
              => state.users.where((user)=>user.name.startsWith(text)).toList());
```  

The above code will calculate the filtered list only once, and then return it when the selector is
called again with the same `state` and `text` parameters.

If the `state` changes, or the `text` changes (or both), it will recalculate and then cache again
the new result.

We can further improve this by noting that we only need to recalculate the result when `state.users`
changes. Since `state.users` is a subset of `state`, it will change less often. So a better selector
would be this:

```dart
static List<User> selectUsersWithNamesStartingWith(AppState state, {String text})
   => _selectUsersWithNamesStartingWith(state.users)(text);
 
static final _selectUsersWithNamesStartingWith = cache1_1(
        (List<User> users) 
           => (String text) 
              => users.where((user)=>user.name.startsWith(text)).toList());
```  

#### Cache syntax

For the moment, AsyncRedux provides these six methods that combine 1 or 2 states with 0, 1 or 2
parameters:

```dart
cache1((state) => () => ...);
cache1_1((state) => (parameter) => ...);
cache1_2((state) => (parameter1, parameter2) => ...);

cache2((state1, state2) => () => ...);
cache2_1((state1, state2) => (parameter) => ...);
cache2_2((state1, state2) => (parameter1, parameter2) => ...);
```    

I have created only those above, because for my own usage I never required more than that. Please,
open an <a href="https://github.com/marcglasberg/async_redux/issues">issue</a>
to ask for more variations in case you feel the need.

This syntax treats the states and the parameters differently. If you call some selector while
keeping the **same state** and changing only the parameter, the selector will cache all the results,
one for each parameter.

However, as soon as you call the selector with a **changed state**, it will delete all of its
previous cached information, since it understands that they are no longer useful. And even if you
don't call that selector ever again, it will delete the cached information if it detects that the
state is no longer used in other parts of the program. In other words, AsyncRedux keeps the cached
information in <a href="https://pub.dev/packages/weak_map">weak-map</a>, so that the cache will not
hold to old information and have a negative impact in memory usage.

#### The reselect package

The reselect functionality explained above is provided out-of-the-box with AsyncRedux. However,
AsyncRedux also works perfectly with the external <a href="https://pub.dev/packages/reselect">
reselect</a> package.

Then, why did I care to reimplement a similar functionality? What are the differences?

First, the AsyncRedux caches can keep any number of cached results for each selector, one for each
time the selector is called with the same states and different parameters. Meanwhile, the reselect
package keeps a single cached result per selector.

And second, the AsyncRedux reselector discards the cached information when the state changes or is
no longer used. Meanwhile, the reselect package will always keep the states and cached results in
memory.

<br>

## Action Subclassing

Suppose you have the following `AddTodoAction` for the To-Do app:

```dart
class AddTodoAction extends ReduxAction<AppState> {
  final Todo todo;
  AddTodoAction(this.todo);

  @override
  AppState reduce() {
	if (todo == null) return null;
	else return state.copy(todoState: List.of(state.todoState.todos)..add(todo));
  }
}

// You would use it like this:
store.dispatch(AddTodoAction(Todo("Buy some beer.")));
```

Since all actions extend `ReduxAction`, you may further use object oriented principles to reduce
boilerplate. Start by creating an **abstract** action base class to allow easier access to the
sub-states of your store. For example:

```dart
abstract class BaseAction extends ReduxAction<AppState> {
  LoginState get loginState => state.loginState;
  UserState get userState => state.userState;
  TodoState get todoState => state.todoState;
  List<Todo> get todos => todoState.todos;
}
```

And then your actions have an easier time accessing the store state:

```dart
class AddTodoAction extends BaseAction {
  final Todo todo;
  AddTodoAction(this.todo);

  @override
  AppState reduce() {
	if (todo == null) return null;
	else return state.copy(todoState: List.of(todos)..add(todo)));
  }
}
```

As you can see above, instead of writing `List.of(state.todoState.todos)` you can simply
write `List.of(todos)`. It may seem a small reduction of boilerplate, but it adds up.

<br>

### Abstract Before and After

Other useful abstract classes you may create provide already overridden `before()` and `after()`
methods. For example, this abstract class turns on a modal barrier when the action starts, and
removes it when the action finishes:

```dart
abstract class BarrierAction extends ReduxAction<AppState> {
  void before() => dispatch(BarrierAction(true));
  void after() => dispatch(BarrierAction(false));
}
```

Then you could use it like this:

```dart
class ChangeTextAction extends BarrierAction {

  @override
  Future<AppState> reduce() async {
	String newText = await read(Uri.http("numbersapi.com","${state.counter}");
	return state.copy(
	  counter: state.counter + 1,
	  changeTextEvt: Event<String>(newText));
  }
}
```

The above `BarrierAction` is demonstrated
in <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_event_redux.dart">
this example</a>.

<br>

## Dependency Injection

While you can always use <a href="https://pub.dev/packages/get_it">get_it</a> or any other
dependency injection solution, AsyncRedux lets you inject your dependencies directly in the
**store**, and then access them in your actions and view-model factories.

The dependency injection idea was contributed by <a href="https://github.com/craigomac">Craig
McMahon</a>.

To inject an environment object with the dependencies:

```dart
store = Store<AppState>(
   initialState: ...,
   environment: Environment(),
);
```

You can then extend both `ReduxAction` and `VmFactory` to provide typed access to your environment:

```dart
abstract class AppFactory<T extends Widget?, Model extends Vm> extends VmFactory<int, T, Model> {
  AppFactory([T? connector]) : super(connector);

  @override
  Environment get env => super.env as Environment;
}


abstract class Action extends ReduxAction<int> {

  @override
  Environment get env => super.env as Environment;
}
```

Then, use the environment when creating the view-model:

```dart
class Factory extends AppFactory<MyHomePageConnector> {
  Factory(connector) : super(connector);

  @override
  ViewModel fromStore() => ViewModel(
        counter: env.limit(state),
        onIncrement: () => dispatch(IncrementAction(amount: 1)),
      );
}

```

And also in your actions:

```dart
class IncrementAction extends Action {
  final int amount;
  IncrementAction({required this.amount});

  @override
  int reduce() => env.incrementer(state, amount);
}
```

Try running
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_environment.dart">
Dependency Injection Example</a>.

## IDE Navigation

How does AsyncRedux solve the IDE navigation problem?

During development, if you need to see what some action does, you just tell your IDE to navigate to
the action itself
(`CTRL+B` in IntelliJ/Windows, for example) and you have the reducer right there.

If you need to list all of your actions, you just go to the `ReduxAction` class declaration and ask
the IDE to list all of its subclasses.

<br>

## Persistence

Your store optionally accepts a `persistor`, which may be used for local persistence, i.e., keeping
the current app state saved to the local disk of the device.

You should create your own `MyPersistor` class which extends the `Persistor` abstract class. This is
the recommended way to use it:

```dart                       
var persistor = MyPersistor();          

var initialState = await persistor.readState();

if (initialState == null) {
      initialState = AppState.initialState();
      await persistor.saveInitialState(initialState);
    }

var store = Store<AppState>(
  initialState: initialState,  
  persistor: persistor,
);
```           

As you can see above, when the app starts you use the `readState` method to try and read the state
from the disk. If this method returns `null`, you must create an initial state and save it. You then
create the store with the `initialState` and the `persistor`.

This is the `Persistor` implementation:

```dart
abstract class Persistor<St> {
  Future<St?> readState();  
  Future<void> deleteState();  
  Future<void> persistDifference({required St? lastPersistedState, required St newState});  
  Future<void> saveInitialState(St state) => persistDifference(lastPersistedState: null, newState: state);    
  Duration get throttle => const Duration(seconds: 2);
}
```                   

The `persistDifference` method is the one you should implement to be notified whenever you must save
the state. It gets the `newState` and the `lastPersistedState`, so that you can compare them and
save the difference. Or, if your app state is simple, you can simply save the whole `newState` each
time the method is called.

The `persistDifference` method will be called by AsyncRedux whenever the state changes, but not more
than once each 2 seconds, which is the throttle period. All state changes within these 2 seconds
will be collected, and then a single call to the method will be made with all the changes after this
period.

Also, the `persistDifference` method won't be called while the previous save is not finished, even
if the throttle period is done. In this case, if a new state becomes available the method will be
called as soon as the current save finishes.

Note you can also override the `throttle` getter to define a different throttle period. In special,
if you define it as `null` there will be no throttle, and you'll be able to save the state as soon
as it changes.

Even if you have a non-zero throttle period, sometimes you may want to save the state immediately.
This is usually the case, for example, when the app is closing. You can do that by dispatching the
provided `PersistAction`. This action will ignore the throttle period and call
the `persistDifference` method right away to save the current state.

```dart
store.dispatch(PersistAction());
```  

You can use this code to help you start extending the abstract `Persistor` class:

```dart
class MyPersistor extends Persistor<AppState> {
  
  @override
  Future<AppState?> readState() async {
    // TODO: Put here the code to read the state from disk.
    return null;
  }

  @override
  Future<void> deleteState() async {
    // TODO: Put here the code to delete the state from disk.
  }

  @override
  Future<void> persistDifference({
    required AppState? lastPersistedState,
    required AppState newState,
  }) async {
    // TODO: Put here the code to save the state to disk.
  }

  @override
  Future<void> saveInitialState(AppState state) =>
      persistDifference(lastPersistedState: null, newState: state);

  @override
  Duration get throttle => const Duration(seconds: 2);
}

```

The persistor can also be paused and resumed, with methods `store.pausePersistor()`,
`store.persistAndPausePersistor()` and `store.resumePersistor()`. This may be used together with the
app lifecycle, to prevent a persistence process to start when the app is being shut down.

When logging out of the app, you can call `store.deletePersistedState()` to ask the persistor to
delete the state from disk.

First, add an `AppLifecycleManager` to your widget tree:

```dart
...
child: StoreProvider<AppState>(
  store: store,
    child: AppLifecycleManager(
      child: MaterialApp( ...
```

Then, create the `AppLifecycleManager` with a `WidgetsBindingObserver` that dispatches
a `ProcessLifecycleChange_Action`:

```dart
class AppLifecycleManager extends StatefulWidget {
  final Widget child;
  const AppLifecycleManager({Key? key, required this.child}) : super(key: key);  
  _AppLifecycleManagerState createState() => _AppLifecycleManagerState();
}

class _AppLifecycleManagerState extends State<AppLifecycleManager> with WidgetsBindingObserver {  

  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  void didChangeAppLifecycleState(AppLifecycleState lifecycle) {
    store.dispatch(ProcessLifecycleChange_Action(lifecycle));
  }
  
  Widget build(BuildContext context) => widget.child;
}
```

Finally, define your `ProcessLifecycleChange_Action` to pause and resume the persistor:

```dart
class ProcessLifecycleChangeAction extends ReduxAction<AppState> {
  final AppLifecycleState lifecycle;
  ProcessLifecycleChangeAction(this.lifecycle);

  @override
  Future<AppState?> reduce() async {
    if (lifecycle == AppLifecycleState.resumed || lifecycle == AppLifecycleState.inactive) {
      store.resumePersistor();  
    } else if (lifecycle == AppLifecycleState.paused || lifecycle == AppLifecycleState.detached) {
      store.persistAndPausePersistor();
    } else
      throw AssertionError(lifecycle);

    return null;
  }
}
```

Have a look at
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/test/persistence_test.dart">
Persistence tests</a>.

<br>

### Saving and Loading

You can choose any way you want to save the state difference to the local disk, but one way is using
the provided `LocalPersist` class, which is very easy to use.

Note: At the moment it only works for Android/iOS, not for the web.

First, import it:

```dart 
import 'package:async_redux/local_persist.dart';
```

You need to convert yourself your objects to a list of **simple objects**
composed only of numbers, booleans, strings, lists and maps (you can nest lists and maps).

For example, this is a list of simple objects:

```dart
List<Object> simpleObjs = [
  'Goodbye',
  '"Life is what happens\n\rwhen you\'re busy making other plans." -John Lennon',
  [100, 200, {"name": "João"}],
  true,
  42,
];
```

Then create a `LocalPersist` class to use the `/db/myFile.db` file:

```dart
var persist = LocalPersist("myFile");
```

You can save the list to the file:

```dart
await persist.save(simpleObjs);
```

And then later load these objects:

```dart                                       
List<Object> simpleObjs = await persistence.load();
```

Usually the `save` method rewrites the file. But it also lets you append more objects:

```dart
List<Object> moreObjs = ['Lets', 'append', 'more'];
await persist.save(simpleObjs, append: true);
```

You can also delete the file, get its length, see if it exists etc:

```dart                      
int length = await persist.length();
bool exists = await persist.exists();
await persist.delete();
```                            

Note: `LocalPersist` uses a file format similar to JSON, but not exactly JSON, because JSON cannot
be appended to. If you want to save and load a single object into standard JSON, use the `saveJson`
and `loadJson` methods:

```dart
await persist.saveJson(simpleObj);
Object? simpleObj = await persistence.loadJson();
```

Have a look at
the: <a href="https://github.com/marcglasberg/async_redux/blob/master/test/local_persist_test.dart">
Local Persist tests</a>.

<br>

## Logging

Your store optionally accepts lists of `actionObservers` and `stateObservers`, which may be used for
logging:

```dart
var store = Store<AppState>(
  initialState: state,
  actionObservers: [Log.printer(formatter: Log.verySimpleFormatter)],
  stateObservers: [StateLogger()],
);
```

The `ActionObserver` is an abstract class with an `observe` method which you can implement to be
notified of **action dispatching**:

```dart
abstract class ActionObserver<St> {

   void observe(
      ReduxAction<St> action, 
      int dispatchCount, {
      required bool ini,
      }
   );
}
```

The above observer will actually be called twice, one with `ini==true` for the INITIAL action
observation, and one with `ini==false` for the END action observation,

In more detail:

1. The INI action observation means the action was just dispatched and haven't changed anything yet.
   After that, it may do sync stuff, and it may or may not start async processes, depending on its
   reducer being sync or async.

2. The END action observation means the action reducer has just finished returning a new state, thus
   changing the store state. Only after getting END states you may see store changes.

3. The state observation is therefore called as soon as possible after the store change has taken
   place, i.e., right after the END action observation. However, it contains a copy of both the
   state **before the action INI** and the state **after the action END**, in case you need to
   compare them.

Please note, unless the action reducer is synchronous, getting an END action observation doesn't
mean that all the action effects have finished, because the action may have started async
processes that may well last into the future. And these processes may later dispatch other actions
that will change the store state. However, it does mean that the action can no longer change the
state **directly**.

Meanwhile, the `StateObserver` is an abstract class which you can implement to be notified of
**state changes**. This observer can be used for logging, but it can also be used for metrics.

<br>

## Metrics

The recommended place to add metrics is in the `stateObservers`:

```dart
var store = Store<AppState>(
  initialState: state,  
  stateObservers: [MetricsObserver()],
);
```

The `StateObserver` is an abstract class which you can implement to be notified of
**state changes**:

```dart
abstract class StateObserver<St> {

   void observe(
      ReduxAction<St> action, 
      St stateIni, 
      St stateEnd, 
      Object? error,
      int dispatchCount,
      );
}
```

One or more `StateObserver`s can be set during the `Store` creation. Those observers are
called for all dispatched actions, right after the reducer returns. That happens before the
`after()` method is called, and before the action's `wrapError()` and the
global `globalWrapError()` methods are called.

The parameters are:

* `action` = The action itself.

* `prevState` = The state right before the new state returned by the reducer is applied. Note this
  may be different from the state when the reducer was called.

* `newState` = The state returned by the reducer. Note: If you need to know if the state was
  changed or not by the reducer, you can compare both states:
  `bool ifStateChanged = !identical(prevState, newState);`

* `error` = Is `null` if the reducer completed with no error and returned. Otherwise, will be the
  error thrown by the reducer (before any `wrapError` is applied). Note that, in case of
  error, both `prevState` and `newState` will be the current store state when the error is
  thrown.

* `dispatchCount` = The sequential number of the dispatch.

Among other uses, the state-observer is a good place to add **metrics** to your application.
For example:

```dart
 abstract class AppAction extends ReduxAction<AppState> {
   void trackEvent(AppState stateIni, AppState stateEnd) { // Don't to anything }
 }

 class AppStateObserver implements StateObserver<AppState> {
   @override
   void observe(
     ReduxAction<AppState> action,
     AppState prevState,
     AppState newState,
     Object? error,
     int dispatchCount,
   ) {
     if (action is AppAction) action.trackEvent(prevState, newState, error);
   }
 }

 class MyAction extends AppAction {
    @override
    AppState? reduce() { // Do something }

    @override
    void trackEvent(AppState prevState, AppState newState, Object? error) =>
       MyMetrics().track(this, newState, error);
 }
```

<br>

## Printing actions to the console

If you use the provided `ConsoleActionObserver`, it will print all actions to the console, in
yellow, like so:

```dart
I/flutter (15304): | Action MyAction
```

This helps with development, so you probably don't want to use it in release mode:

```dart
store = Store<AppState>(
   ...
   actionObservers: kReleaseMode ? null : [ConsoleActionObserver()],
);
```

If you implement the action's `toString()`, you can display more information. For example, suppose a
`LoginAction` which has a `username` field:

```dart
class LoginAction extends ReduxAction {
  final String username;
  ...
  String toString() => super.toString() + '(username)';
}
```

The above code will print something like this:

```
I/flutter (15304): | Action MyLogin(user32)
```

## Observing rebuilds

Your store optionally accepts a `modelObserver`, which lets you visualize rebuilds.
It's rarely used, so feel free to skip this section.

The `ModelObserver` is an abstract class with an `observe` method which you can implement to be
notified, by each `StoreConnector` currently in the widget tree, whenever there is a state change.
You can create your own `ModelObserver`, but the provided `DefaultModelObserver` can be used out of
the box to print to the console and do basic testing:

```dart
var store = Store<AppState>(
  initialState: state,
  modelObserver: DefaultModelObserver(),  
);
```                                      

This is an example output to the console, showing how `MyWidgetConnector` responded to 3 state
changes:

    Model D:1 R:1 = Rebuid:true, Connector:MyWidgetConnector, Model:MyViewModel{B}.
    Model D:2 R:2 = Rebuid:false, Connector:MyWidgetConnector, Model:MyViewModel{B}.
    Model D:3 R:3 = Rebuid:true, Connector:MyWidgetConnector, Model:MyViewModel{C}.

You can see above that the first and third state changes resulted in a rebuild (`Rebuid:true`), but
the second one did not, probably because the part of the state that changed was not part
of `MyViewModel`.

<a href="https://github.com/marcglasberg/async_redux/blob/master/example/lib/main_dispatch_future.dart">
This example</a>
also shows the `ModelObserver` in action.

Note: You must pass `debug:this` as a `StoreConnector` constructor parameter, if you want
the `ModelObserver` to be able to print the `StoreConnector` type to the output. You can also
override your `ViewModel.toString()` to print out any extra info you need.

The `ModelObserver` is also useful when you want to create tests to assert that rebuilds happen
when and only when the appropriate parts of the state change. For an example, see
the <a href="https://github.com/marcglasberg/async_redux/blob/master/test/model_observer_test.dart">
Model Observer Test</a>.

<br>

## How to interact with the database

The following advice works for any Redux version, including AsyncRedux.

Pretend the user presses a button in the dumb-widget, running a callback which was passed in its
constructor. This callback, which was created by the Connector widget, will dispatch an action.

This action's async reducer will connect to the database and get the desired information. You can
**directly** connect to the database from the async reducer, or have a **DAO** to abstract the
database implementation details.

This would be your reducer:

```dart
@override
Future<AppState> reduce() async {
	var something = await myDao.loadSomething();
	return state.copy(something: something);
}
```

This rebuilds your widgets that depend on `something`, with its new value. The state now holds the
new `something`, and the local store persistor may persist this value to the local file system, if
that's what you want.

<br>

## How to deal with Streams and Timers

Follow this advice:

- Don't send the streams and timers down to the dumb-widget, and not even to the Connector.
  If you are declaring, subscribing to, or unsubscribing from streams inside of widgets, it means
  you are mixing Redux with some other architecture. You _can_ do that, but it's not recommended
  and not necessary.

- Don't put streams and timers into the store state. They are not app state, and they should not be
  persisted to the local filesystem. Instead, they are something that "generates state changes".

Let's pretend you want to listen to changes to the user name, in a Firestore database.
First, create an action to start listening, and another action to cancel. We could name
them `StartListenUserNameAction` and `CancelListenUserNameAction`.

- If the stream/timer should run all the time, you may dispatch the start action as soon as the app
  starts, right after you create the store, possibly in `main`. And cancel it when the app finishes.

- If the stream/timer should run only when the user is viewing some screen, you may dispatch the
  action from `initState` (of the screen widget) or `onInit` (StoreConnector), and cancel it from
  the `dispose` (of the screen widget) or `onDispose` (StoreConnector).

- If the stream/timer should run only when some action demands it, the action reducer may dispatch
  some other action to start and cancel them as needed.

While you should NOT put streams/timers in the store state, you can put them in the
store "properties". The `props` are a map that you can use to store any object you want,
and they are accessible from the reducers.

```dart
class StartTimerAction extends ReduxAction<AppState> {

  Future<AppState> reduce() async {
    props["my timer"] = Timer.periodic(Duration(seconds: 1), (timer) {
      dispatch(DoSomethingAction(timer.tick));
    });
    return null;	
  }
}

class StopTimerAction extends ReduxAction<AppState> {
    
  Future<AppState> reduce() async {
    Timer? timer = props["my timer"];
    if (timer != null) {
       timer.cancel();
       props.remove("my timer");
    }
    return null;	
  }
}  
```

If your stream/timer should only be removed when the app shuts down, you can
call `store.disposeProps();` when the app finishes. This will automatically close/cancel/ignore
all stream related objects, timers and futures in the props, and then also remove them from there.

<br>

### How do streams pass their information to the store and ultimately to the widgets?

When you create the stream, define its callback so that it dispatches an appropriate action. Each
time the stream gets some data it will pass it to this action's constructor. The action's reducer
will put the data into the store state, from where it will be automatically sent down to the widgets
that observe them.

For example:

```dart
Stream<QuerySnapshot> stream = query.snapshots();

streamSub = stream.listen((QuerySnapshot querySnapshot) {
  dispatch(DoSomethingAction(querySnapshot.documentChanges));
  }, onError: ...);
```

<br>

### To sum up:

1. Put your stream/timer where it can be accessed by the reducers, like in the store
   props or any other suitable place, but NOT inside the store state.

2. Don't use streams or timers directly in widgets (not in the Connector widget, and not in the
   dumb-widget).

3. Create actions to start and cancel streams and timers, and call them when necessary.

4. The stream/timer callback should dispatch actions to put the snapshot data into the store state.

<br>

## Undo and Redo

It's easy to create undo/redo features in AsyncRedux. When you create the store, add a
state-observer called `UndoRedoObserver`:

```dart
var store = Store<AppState>(
  initialState: state,  
  stateObservers: [UndoRedoObserver()],
);
```

That observer should add all the states it gets to a list. When you want to recover one of the
states, simple call an action called `RecoverStateAction`, like so:

```dart
class RecoverStateAction extends ReduxAction<AppState> {
   final AppState recoveredState;

   RecoverStateAction(this.recoveredState);

   @override
   AppState reduce() => recoveredState;
}
```

Note: This also works to undo/redo only part of the state. If you are only interested in undoing
part of the state, your observer can save only that part, and your action can revert only that part.

<br>

## Using Async Redux with the flutter_hooks package

For those who want to use `flutter_hooks` with Async Redux, add
the https://pub.dev/packages/flutter_hooks_async_redux package to your app.
This is a very lightweight package that provides a `useSelector` hook, which is similar to the
`StoreConnector` widget, but it's a hook.

Note: If your state is called `AppState`, you can define your own `useAppState` hook, like this:

```dart
T useAppState<T>(T Function(AppState state) converter, {bool distinct = true}) =>
   useSelector<T, AppState>(converter, distinct: distinct);
```

This will simplify the use of the hook, like this:

```dart
String username = useAppState((state) => state.username);
```

<br>

## Recommended Directory Structure

You probably have your own way of organizing your directory structure, but if you want some
recommendation, here it goes.

First, separate your directory structure by **client** and **business**. The **client** directory
holds Flutter stuff like widgets, including your connector and dumb widgets. The **business**
directory holds the business layer stuff, including the store, state, and code to access the
database and to persist the state to disk.

```
├── business
│  ├── lib
│  ├── test
│  └── pubspec.yaml
└── client
    ├── lib
    ├── test
    └── pubspec.yaml
```

Edit the `client/pubspec.yaml` file to contain this:

```
dependencies:
  business:
    path: ../business/
```

However, `business/pubspec.yaml` should contain no references to the **client**. This guarantees
the **client** code can use the **business** code, but the **business** code can't access the
**client** code.

In `business/lib` create separate directories for your main features, and only then create
directories like `actions`, `models`, `dao` or other.

Note that AsyncRedux has no separate reducers nor middleware, so this simplifies the directory
structure in relation to vanilla Redux.

Your final directory structure would then look something like this:

```
├── business
│   ├── lib
│   │   ├── login
│   │   │   ├── actions
│   │   │   │   ├── login_action.dart
│   │   │   │   ├── logout_action.dart
│   │   │   │   └── ...
│   │   │   └── models
│   │   │       └── login_state.dart
│   │   ├── todos
│   │   │   ├── actions
│   │   │   │   └── ...
│   │   │   └── models
│   │   │       ├── todos_state.dart
│   │   │       └── todo.dart
│   │   └── users
│   │       ├── actions
│   │       │   ├── create_user_action.dart
│   │       │   ├── change_user_action.dart
│   │       │   ├── delete_user_action.dart
│   │       │   └── ...
│   │       └── models
│   │           └── user.dart
│   ├── test
│   │   ├── login
│   │   │   ├── login_STATE_test.dart
│   │   │   ├── login_action_test.dart
│   │   │   ├── logout_action_test.dart
│   │   │   └── ...
│   │   ├── todos
│   │   │   ├── todos_STATE_test.dart
│   │   │   └── todo_test.dart
│   │   └── users
│   │       └── user_test.dart
│   ├── pubspec.yaml
│   └── ...
└── client
    ├── lib
    │   ├── login
    │   │   ├── login_connector_widget.dart
    │   │   └── login_widget.dart
    │   └── todos
    │       ├── todos_connector_widget.dart
    │       └── todos_widget.dart
    ├── test
    │   ├── login
    │   │   ├── login_CONNECTOR_test.dart
    │   │   └── login_PRESENTATION.dart
    │   └── todos
    │       ├── todos_CONNECTOR_test.dart
    │       └── todos_PRESENTATION.dart
    └── pubspec.yaml
```

<br>

## Where to put your business logic

Widgets, Connectors and ViewModels are part of the client code. If you use the recommended directory
structure, they should be in the client directory, which is **not** visible to the business code.

Actions, reducers and state classes are part of the business code. If you use the recommended
directory structure, they should be in the business directory, which is visible to the client code.

Rules of thumb:

* Don't put your business logic in the Widgets.
* Don't put your business logic in the Connectors.
* Don't put your business logic in the ViewModels of the Connectors.
* Put your business logic in the Action reducers.
* Put your business logic in the State classes.

<br>

## Architectural discussion

Reading the following text is not important for the practical use of AsyncRedux, and is meant only
for those interested in architectural discussions:

### Is AsyncRedux really Redux?

According to redux.js.org there are three principles to Redux:

1. **The state of your whole application is stored in an object tree within a single store.**

   That’s true for AsyncRedux.

2. **The only way to change the state is to emit an action, an object describing what happened.**

   That’s also true for AsyncRedux.

3. **To specify how the state tree is transformed by actions, you write pure reducers.**

   Ok, so how about middleware? It's not possible to create real world applications without async
   calls and external databases access. So, even in vanilla Redux, actions start async processes
   that yield results that only then will be put into the store state, through reducers. So it's not
   true that the state tree depends only on pure functions. You can't separate the pure part and
   call it a reducer, and then conveniently forget about the impure/async part. In other words, you
   have A and B. A is simple and pure, but we can't call it a reducer and say that's part of our
   principles, and then forget about B. Async Redux acknowledges that B is also part of the
   solution, and then creates tools to deal with it as easily as possible. The litmus test here, to
   prove that AsyncRedux is Redux, is that you can have a 1 to 1 mapping from vanilla Redux
   reducers+middleware code to the AsyncRedux sync+async reducers code. The same async code will
   call the same pure code. You just organize it differently to avoid boilerplate. Another way to
   look at it is that at first glance the AsyncRedux reducer doesn't appear to be a pure function.
   Pure function reducers are the wall of sanity against the side effects managed by middleware via
   thunks, sagas, observables, etc. But when you take a second look, `return state.copy(...)` is the
   pure reducer, and everything else in `reduce()` is essentially middleware.

### Besides the reduction of boilerplate, what are the main advantages of the AsyncRedux architecture?

In vanilla Redux it's easy to reason about the code at first, when it's just pure function reducers,
but it gets difficult to understand the whole picture as soon as you have to add complex Middleware.
When you see a side by side comparison of code written for vanilla Redux and for AsyncRedux, the
code is easier to understand with AsyncRedux.

Also, vanilla Redux makes it easy to test its pure functions reducers, but it doesn't help at all
with testing the middleware. In contrast, since AsyncRedux natively takes async code into
consideration, its testing capabilities (the `StoreTester`) make it easy to test the code as a
whole.

AsyncRedux also helps with code errors, by simply letting your reducers throw errors. AsyncRedux
will catch them and deal with them appropriately, while vanilla Redux forces your middleware to
catch errors and maybe even dispatch actions do deal with them.

### Is AsyncRedux a minimalist or lightweight Redux version?

No. AsyncRedux is concerned with being "easy to use", not with being lightweight. In terms of
library code size it's larger than the original Redux implementation. However, it's still very
small, and will make the total application code smaller than with the vanilla implementation,
because of the boilerplate reduction. In terms of speed/performance there should be no differences
in respect to the vanilla implementation.

### Is the AsyncRedux architecture useful for small projects?

It's usually said that you should not use Redux for small projects, because of the extra boilerplate
and limitations. Maybe it's not worth the effort. However, since AsyncRedux is easier than vanilla
Redux and has far less boilerplate, the limit of code complexity where a robust architecture starts
making sense is much lower.

***

*The AsyncRedux code is based upon packages <a href="https://pub.dev/packages/redux">redux</a> by
Brian Egan, and <a href="https://pub.dev/packages/flutter_redux">flutter_redux</a> by Brian Egan and
John Ryan. Also uses code from package <a href="https://pub.dev/packages/equatable">equatable</a> by
Felix Angelov. The dependency injection idea in AsyncRedux was contributed by Craig McMahon. Special
thanks: Eduardo Yamauchi and Hugo Passos helped me with the async code, checking the documentation,
testing everything and making suggestions. This work started after Thomas Burkhart explained to me
why he didn't like Redux. Reducers as methods of action classes were shown to me by Scott Stoll and
Simon Lightfoot.*

*The Flutter packages I've authored:*

* <a href="https://pub.dev/packages/async_redux">async_redux</a>
* <a href="https://pub.dev/packages/fast_immutable_collections">fast_immutable_collections</a>
* <a href="https://pub.dev/packages/provider_for_redux">provider_for_redux</a>
* <a href="https://pub.dev/packages/i18n_extension">i18n_extension</a>
* <a href="https://pub.dev/packages/align_positioned">align_positioned</a>
* <a href="https://pub.dev/packages/network_to_file_image">network_to_file_image</a>
* <a href="https://pub.dev/packages/image_pixels">image_pixels</a>
* <a href="https://pub.dev/packages/matrix4_transform">matrix4_transform</a>
* <a href="https://pub.dev/packages/back_button_interceptor">back_button_interceptor</a>
* <a href="https://pub.dev/packages/indexed_list_view">indexed_list_view</a>
* <a href="https://pub.dev/packages/animated_size_and_fade">animated_size_and_fade</a>
* <a href="https://pub.dev/packages/assorted_layout_widgets">assorted_layout_widgets</a>
* <a href="https://pub.dev/packages/weak_map">weak_map</a>
* <a href="https://pub.dev/packages/themed">themed</a>
* <a href="https://pub.dev/packages/bdd_framework">bdd_framework</a>

*My Medium Articles:*

* <a href="https://medium.com/flutter-community/https-medium-com-marcglasberg-async-redux-33ac5e27d5f6">
  Async Redux: Flutter’s non-boilerplate version of Redux</a> (
  versions: <a href="https://medium.com/flutterando/async-redux-pt-brasil-e783ceb13c43">
  Português</a>)
* <a href="https://medium.com/flutter-community/i18n-extension-flutter-b966f4c65df9">
  i18n_extension</a> (
  versions: <a href="https://medium.com/flutterando/qual-a-forma-f%C3%A1cil-de-traduzir-seu-app-flutter-para-outros-idiomas-ab5178cf0336">
  Português</a>)
* <a href="https://medium.com/flutter-community/flutter-the-advanced-layout-rule-even-beginners-must-know-edc9516d1a2">
  Flutter: The Advanced Layout Rule Even Beginners Must Know</a> (
  versions: <a href="https://habr.com/ru/post/500210/">русский</a>)
* <a href="https://medium.com/flutter-community/the-new-way-to-create-themes-in-your-flutter-app-7fdfc4f3df5f">
  The New Way to create Themes in your Flutter App</a>
* <a href="https://medium.com/@marcglasberg/a-new-bdd-tool-for-typescript-javascript-and-dart-673933b3b38e">
  A new BDD tool for TypeScript/React, and Flutter/Dart</a>

*My article in the official Flutter documentation*:

* <a href="https://flutter.dev/docs/development/ui/layout/constraints">Understanding constraints</a>

<br>_Marcelo Glasberg:_<br>
_https://github.com/marcglasberg_<br>
_https://linkedin.com/in/marcglasberg_<br>
_https://twitter.com/glasbergmarcelo_<br>
_https://stackoverflow.com/users/3411681/marcg_<br>
_https://medium.com/@marcglasberg_<br>
