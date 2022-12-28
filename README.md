```
name: my_app_name
environment:
  sdk: ">=2.17.0 <3.0.0"
  flutter: ">=3.0.0"

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.1.1
  riverpod_annotation: ^1.0.6

dev_dependencies:
  build_runner:
  riverpod_generator: ^1.0.6
```

Finally, run the code-generator with

`flutter pub run build_runner watch`

돌리면 지속적으로 build를 해준다.

Riverpod does not rely on generic types. Instead it relies on the variable created using provider definition.

The rules for context.watch vs context.read applies to Riverpod too:
Inside the build method, use "watch". Inside click handlers and other events, use "read".

# Combining Providers

provider

```
class UserIdNotifier extends ChangeNotifier {
  String? userId;
}

// ...

ChangeNotifierProvider<UserIdNotifier>(create: (context) => UserIdNotifier()),

ProxyProvider<UserIdNotifier, String>(
  update: (context, userIdNotifier, _) {
    return 'The user ID of the the user is ${userIdNotifier.userId}';
  }
)

```

riverpod

```
class UserIdNotifier extends ChangeNotifier {
  String? userId;
}

// ...

final userIdNotifierProvider = ChangeNotifierProvider<UserIdNotifier>(
  (ref) => UserIdNotifier(),
),

final labelProvider = Provider<String>((ref) {
  UserIdNotifier userIdNotifier = ref.watch(userIdNotifierProvider);
  return 'The user ID of the the user is ${userIdNotifier.userId}';
});
```

This ref.watch line should feel similar. This pattern was covered previously when explaining how to read providers inside widgets. Indeed, providers are now able to listen to other providers in the same way that widgets do.

# Mutable?

State is simply information about something held in memory.

As a simple exercise in object orientation, think of a class as a cookie cutter, and cookies as objects. You can create a cookie (instantiate an object) using the cookie cutter (class). Let's say one of the properties of the cookie is its color (which can be changed by using food coloring). The color of that cookie is part of its state, as are the other properties.

Mutable state is state that can be changed after you make the object (cookie). Immutable state is state that cannot be changed.

Immutable objects (for which none of the state can be changed) become important when you are dealing with concurrency, the ability for more than one processor in your computer to operate on that object at the same time. Immutability guarantees that you can rely on the state to be stable and valid for the object's lifetime.

# State Notifier

An observable class that stores a single immutable state.

It can be used as a drop-in replacement to ChangeNotifier or other equivalent objects like Bloc. Its particularity is that it tries to be simple, yet promote immutable data.

By using immutable state, it becomes a lot simpler to:

compare previous and new state
implement undo-redo mechanism
debug the application state

StateNotifier is designed to be subclassed. We first need to pass an initial value to the super constructor, to define the initial state of our object.

```
class Counter extends StateNotifier<int> {
  Counter(): super(0);
}
We can then expose methods on our StateNotifier to allow other objects to modify the counter:

class Counter extends StateNotifier<int> {
  Counter(): super(0);

  void increment() => state++;
  void decrement() => state--;
}
```

assigning state to a new value will automatically notify the listeners and update the UI.

Then, the object can either be listened like with StateNotifierBuilder/StateNotifierProvider using package:flutter_state_notifier or package:riverpod.

# Provider

## cache computation

we ideally do not want to filter our list of todos whenever our application re-renders. In this situation, we could use Provider to do the filtering for us.

```
# 모델이자 스테이트
class Todo {
  Todo(this.description, this.isCompleted);
  final bool isCompleted;
  final String description;
}

# 노티파이어이자 뷰모델
@riverpod
class Todos extends _$Todos {
  @override
  List<Todo> build() {
    return [];
  }

  void addTodo(Todo todo) {
    state = [...state, todo];
  }
}

# 프로바이더
@riverpod
List<Todo> completedTodos(CompletedTodosRef ref) {
  final todos = ref.watch(todosProvider);

  // we return only the completed todos
  return todos.where((todo) => todo.isCompleted).toList();
}

# 컨슈머
Consumer(builder: (context, ref, child) {
  final completedTodos = ref.watch(completedTodosProvider);
  // TODO show the todos using a ListView/GridView/...
});

```

list of completed todos will not be recomputed until todos are added/removed/updated, even if we are reading the list of completed todos multiple times.

# Reducing build

bad example

```
@riverpod
class PageIndex extends _$PageIndex {
 @override
 int build() {
   return 0;
 }

 void goToPreviousPage() {
   state = state - 1;
 }
}

class PreviousButton extends ConsumerWidget {
 const PreviousButton({Key? key}) : super(key: key);

 @override
 Widget build(BuildContext context, WidgetRef ref) {
   // if not on first page, the previous button is active
   final canGoToPreviousPage = ref.watch(pageIndexProvider) != 0;

   void goToPreviousPage() {
     ref.read(pageIndexProvider.notifier).goToPreviousPage();
   }

   return ElevatedButton(
     onPressed: canGoToPreviousPage ? goToPreviousPage : null,
     child: const Text('previous'),
   );
 }
}


```

이렇게 되면 페이지를 이동할때마다 PreviousButton을 빌드 해줘야 한다. watch가 pageIndex에 걸려있기 때문.

```
@riverpod
class PageIndex extends _$PageIndex {
 @override
 int build() {
   return 0;
 }

 void goToPreviousPage() {
   state = state - 1;
 }
}

// A provider which computes whether the user is allowed to go to the previous page
@riverpod
bool canGoToPreviousPage(CanGoToPreviousPageRef ref) {
 return ref.watch(pageIndexProvider) != 0;
}

class PreviousButton extends ConsumerWidget {
 const PreviousButton({Key? key}) : super(key: key);

 @override
 Widget build(BuildContext context, WidgetRef ref) {
   // We are now watching our new Provider
   // Our widget is no longer calculating whether we can go to the previous page.
   final canGoToPreviousPage = ref.watch(canGoToPreviousPageProvider);

   void goToPreviousPage() {
     ref.read(pageIndexProvider.notifier).goToPreviousPage();
   }

   return ElevatedButton(
     onPressed: canGoToPreviousPage ? goToPreviousPage : null,
     child: const Text('previous'),
   );
 }
}
```

연산 자체를 프로바이더에서 하면 값이 바뀔때만 버튼을 빌드하게 할 수 있다.
