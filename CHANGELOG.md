## 7.4.1

* fix cast issues around store changes in dart 2
* open analyzer version range

## 7.4.0

* add built_redux_test_utils library - exposes an expectDispatched function for expecting a given action gets dispatched asynchronously
* add combine to MiddlewareBuilder class - lets you combine middleware builders
* add toString to Action class
* add example
* add docs

## 7.3.3

* add changelog
* update readme to include link to todoMVC repo

## 7.3.2

* open dependency range on the build package

## 7.3.1

* open dependency range on the built_collection package

## 7.3.0

* dispose is now awaitable

## 7.2.0

* Added actionStream stream to the store api so one can listen to changes resulting from a specific action name.
* Fixed issues called out my adding implicit-dynamic and implicit-cast rules.

## 7.1.0

* Added reducer builders for all collection types in built_collection. This means you can now write reducers that rebuild just a collection. Examples: https://github.com/davidmarne/built_redux/blob/master/test/unit/collection_models.dart
* The action generator now supports inheritance, meaning one can now have action classes extend one another. AbstractReducerBuilder was added as means of merging a reducer builder defined to rebuild the abstract pieces of state. Examples: https://github.com/davidmarne/built_redux/blob/7.1.0/test/unit/inheritance_test_models.dart for examples

## 7.0.0

* **Breaking changes**:
  * A breaking change in behavior was made to action dispatchers. Before action dispatchers executed the middleware/reducers asynchronously, but they now execute synchronously.

For example you may have:

```dart
int getCount(store) {
  print(store.state.count); // prints 0
  store.actions.increment(1);
  print(store.state.count); // would print 1 if actions were sync, 0 if async
  return store.state.count; // would return 1 if actions were sync, 0 if async
}
```

Before this function would return 0 because the state update that occurs from actions.increment would be processed in a future task.
Now this function will return 1 because the state update that occurs from actions.increment is processed in the current task

## 6.0.0

Removes the BuiltReducer interface and the generated BuiltReducer implementation in favor of building a reducer function and passing it to the store at instantiation.

* **Breaking changes**:
  * Store now takes a reducer
  * Nested reducers need to be built with the NestedReducerBuilder and merged wth the main ReducerBuilder using ReducerBuilder.addNestedReducer
  * Models no longer implement BuiltReducer
    * Remove references to the reducer on your models
  * Renamed ActionDispatcher.syncWithStore to setDispatcher
  * Renamed SubStateChange to SubstateChange


Reasoning:

* Decouples your models and reducers.
* Requires less generated code.
* Reducing requires less map look ups. Previously N map look ups were required where N was the number of BuiltReducers in your state tree. Now only 1 map look up is required per action dispatched.


Examples:

* Example model change. Remove the generated BuiltReducer mixin and the reducer getter

Before:

```dart
  abstract class Counter extends Object
     with CounterReducer
     implements Built<Counter, CounterBuilder> {
   int get count;

   get reducer => _reducer;

   Counter._();
   factory BaseCounter() => new _$BaseCounter._(count: 1);
 }
```

After:

```dart
  abstract class Counter implements Built<Counter, CounterBuilder> {
   int get count;

   Counter._();
   factory BaseCounter() => new _$BaseCounter._(count: 1);
 }
```

* Example nested reducer change. Change NestedCounter's reducer builder to a NestedReducerBuilder. Pass the NestedReducerBuilder mapper functions from the main state/builder to the nested state/builder.

Before

```dart

// Built Reducer
abstract class BaseCounter extends Object
    with BaseCounterReducer
    implements Built<BaseCounter, BaseCounterBuilder> {
  int get count;

  BuiltList<int> get indexOutOfRangeList;

  NestedCounter get nestedCounter;

  @override
  get reducer => _baseReducer;

  // Built value constructor
  BaseCounter._();
  factory BaseCounter() => new _$BaseCounter._(
        count: 1,
        nestedCounter: new NestedCounter(),
      );
}

final _baseReducer = (new ReducerBuilder<BaseCounter, BaseCounterBuilder>()
      ..add(BaseCounterActionsNames.increment, _baseIncrement)
      ..add(BaseCounterActionsNames.decrement, _baseDecrement))
    .build();

abstract class NestedCounter extends Object
    with NestedCounterReducer
    implements Built<NestedCounter, NestedCounterBuilder> {
  int get count;

  get reducer => _nestedReducer;

  // Built value constructor
  NestedCounter._();
  factory NestedCounter() => new _$NestedCounter._(count: 1);
}

final _nestedReducer =  (new ReducerBuilder<NestedCounter, NestedCounterBuilder>()
          ..add(NestedCounterActionsNames.increment, _nestedIncrement)
          ..add(NestedCounterActionsNames.decrement, _nestedDecrement))
        .build();

```

After

```dart
abstract class BaseCounter implements Built<BaseCounter, BaseCounterBuilder> {
  int get count;

  NestedCounter get nestedCounter;

  BaseCounter._();
  factory BaseCounter() => new _$BaseCounter._(
        count: 1,
        nestedCounter: new NestedCounter(),
      );
}

// the reducer passed to the store
final reducer = (new ReducerBuilder<BaseCounter, BaseCounterBuilder>()
      ..add(BaseCounterActionsNames.increment, _baseIncrement)
      ..add(BaseCounterActionsNames.decrement, _baseDecrement)
      ..combineNested(_nestedReducer))
    .build();

abstract class NestedCounter implements Built<NestedCounter, NestedCounterBuilder> {
  int get count;

  // Built value constructor
  NestedCounter._();
  factory NestedCounter() => new _$NestedCounter._(count: 1);
}

final _nestedReducer = new NestedReducerBuilder<BaseCounter, BaseCounterBuilder,
    NestedCounter, NestedCounterBuilder>(
  (state) => state.nestedCounter,
  (builder) => builder.nestedCounter,
)
  ..add(NestedCounterActionsNames.increment, _nestedIncrement)
  ..add(NestedCounterActionsNames.decrement, _nestedDecrement);
```