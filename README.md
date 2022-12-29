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

# StateNotifierProvider

핵심이 되고 사용에 가장 추천되는 기능. 복잡한 스테이트와 로직처리에 적합하다.
StateNotifier에 대해 listen을 가능하게 해준다.
Immutable한 state를 보유하며 freezed에서 제공해주는 copyWith 메소드와 함께 쓰는것이 좋은 궁합이다. 안쓰면 힘들게 다 구현해야한다. vscode의 riverpod 익스텐션을 쓰는것도 한가지 방법이다.
비즈니스 로직들을 한군데로 모으는데에 적합하다.

Todo 모델

```

@immutable
class Todo {
  const Todo({required this.id, required this.description, required this.completed});

  // All properties should be `final` on our class.
  final String id;
  final String description;
  final bool completed;

  // Since Todo is immutable, we implement a method that allows cloning the
  // Todo with slightly different content.
  Todo copyWith({String? id, String? description, bool? completed}) {
    return Todo(
      id: id ?? this.id,
      description: description ?? this.description,
      completed: completed ?? this.completed,
    );
  }
}



```

Todo의 Notifier이자 ViewModel

```

// The StateNotifier class that will be passed to our StateNotifierProvider.
// This class should not expose state outside of its "state" property, which means
// no public getters/properties!
// The public methods on this class will be what allow the UI to modify the state.
class TodosNotifier extends StateNotifier<List<Todo>> {
  // We initialize the list of todos to an empty list
  TodosNotifier(): super([]);

  // Let's allow the UI to add todos.
  void addTodo(Todo todo) {
    // Since our state is immutable, we are not allowed to do `state.add(todo)`.
    // Instead, we should create a new list of todos which contains the previous
    // items and the new one.
    // Using Dart's spread operator here is helpful!
    state = [...state, todo];
    // No need to call "notifyListeners" or anything similar. Calling "state ="
    // will automatically rebuild the UI when necessary.
  }

  // Let's allow removing todos
  void removeTodo(String todoId) {
    // Again, our state is immutable. So we're making a new list instead of
    // changing the existing list.
    state = [
      for (final todo in state)
        if (todo.id != todoId) todo,
    ];
  }

  // Let's mark a todo as completed
  void toggle(String todoId) {
    state = [
      for (final todo in state)
        // we're marking only the matching todo as completed
        if (todo.id == todoId)
          // Once more, since our state is immutable, we need to make a copy
          // of the todo. We're using our `copyWith` method implemented before
          // to help with that.
          todo.copyWith(completed: !todo.completed)
        else
          // other todos are not modified
          todo,
    ];
  }
}

```

Provider로 리스너블하게 해준다.

```
// Finally, we are using StateNotifierProvider to allow the UI to interact with
// our TodosNotifier class.
final todosProvider = StateNotifierProvider<TodosNotifier, List<Todo>>((ref) {
  return TodosNotifier();
});

```

그리고 사용할때는
콜백함수에 사용할때는 read로, 나머지는 watch로 쓰는것이 적합하다.

```
class TodoListView extends ConsumerWidget {
  const TodoListView({Key? key}): super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // rebuild the widget when the todo list changes
    List<Todo> todos = ref.watch(todosProvider);

    // Let's render the todos in a scrollable list view
    return ListView(
      children: [
        for (final todo in todos)
          CheckboxListTile(
            value: todo.completed,
            // When tapping on the todo, change its completed status
            onChanged: (value) => ref.read(todosProvider.notifier).toggle(todo.id),
            title: Text(todo.description),
          ),
      ],
    );
  }
}

```

# FutureProvider

주사용처는

- 비동기 요청을 보낼때
- 로드/에러 처리 할때
- 여러 비동기 결과값들을 하나로 합칠때

ref.watch와 함께 쓸때 강력하다.
다만 간단한 use case에만 적합하다. 더 복잡한것들은 statenotifierprovider를 사용하자

### 예시: config file 읽기

이건 code generation을 써서 따로 futureprovider라고 명시 해주지 않은 것 같다.

```
@riverpod
Future<Configuration> fetchConfigration(FetchConfigrationRef ref) async {
  final content = json.decode(
    await rootBundle.loadString('assets/configurations.json'),
  ) as Map<String, Object?>;

  return Configuration.fromJson(content);
}

```

사용시에는

```
Widget build(BuildContext context, WidgetRef ref) {
  final config = ref.watch(fetchConfigrationProvider);

  return config.when(
    loading: () => const CircularProgressIndicator(),
    error: (err, stack) => Text('Error: $err'),
    data: (config) {
      return Text(config.host);
    },
  );
}
```

# StreamProvider

Future대신 Stream용.

- 파이어베이스 리스닝
- 프로바이더 리빌딩
  에 쓰인다.

StreamBuilder 보다 나은점이 있다.

- 다른 프로바이더들이 ref.watch를 이용해 접근가능하다.
- 로드/에러 처리를 하기 좋다.
- it removes the need for having to differentiate broadcast streams vs normal streams.
- 마지막 값이 캐싱이 된다.
- 테스트하기 좋다.

# StateProvider

StateNotifierProvider의 간단한 버전이며 별도의 StateNotifer를 작성하지 않아도 된다.
간단한 UI변경에 따른 값들의 연산에 적합하다. 대체로

- 필터종류 변경과 같은 enum
- 텍스트필드의 String
- 체크박스의 bool
- 페이지번호, 나이입력과 같은 int

이런상황에는 적합하지 않다.

- state에 validation이 필요하다
- state가 복잡하다(리스트, 맵)
- state변경이 `count++` 보다 복잡하다.

이런경우들에는 StateNotifierProvider가 더 적합하다.
결국 장기적으로는 StateNotifier를 별도로 가지고 있는것이 유리하다.

### 예시: 필터 변경

모델

```
class Product {
  Product({required this.name, required this.price});

  final String name;
  final double price;
}

```

더미 리스트

```
final _products = [
  Product(name: 'iPhone', price: 999),
  Product(name: 'cookie', price: 2),
  Product(name: 'ps5', price: 500),
];

```

프로바이더로 리스너블

```
final productsProvider = Provider<List<Product>>((ref) {
  return _products;
});
```

기본적인 리스트

```
Widget build(BuildContext context, WidgetRef ref) {
  final products = ref.watch(productsProvider);
  return Scaffold(
    body: ListView.builder(
      itemCount: products.length,
      itemBuilder: (context, index) {
        final product = products[index];
        return ListTile(
          title: Text(product.name),
          subtitle: Text('${product.price} \$'),
        );
      },
    ),
  );
}
```

필터 드롭다운 추가

```
// An enum representing the filter type
enum ProductSortType {
  name,
  price,
}

Widget build(BuildContext context, WidgetRef ref) {
  final products = ref.watch(productsProvider);
  return Scaffold(
    appBar: AppBar(
      title: const Text('Products'),
      actions: [
        DropdownButton<ProductSortType>(
          value: ProductSortType.price,
          onChanged: (value) {},
          items: const [
            DropdownMenuItem(
              value: ProductSortType.name,
              child: Icon(Icons.sort_by_alpha),
            ),
            DropdownMenuItem(
              value: ProductSortType.price,
              child: Icon(Icons.sort),
            ),
          ],
        ),
      ],
    ),
    body: ListView.builder(
      // ...
    ),
  );
}
```

필터 종류 프로바이더 추가

```
final productSortTypeProvider = StateProvider<ProductSortType>(
  // We return the default sort type, here name.
  (ref) => ProductSortType.name,
);

```

프로바이더와 드롭다운 연결

```
ropdownButton<ProductSortType>(
  // When the sort type changes, this will rebuild the dropdown
  // to update the icon shown.
  value: ref.watch(productSortTypeProvider),
  // When the user interacts with the dropdown, we update the provider state.
  onChanged: (value) =>
      ref.read(productSortTypeProvider.notifier).state = value!,
  items: [
    // ...
  ],
),
```

실제로 리스트에 필터를 반영시키기

```
final productsProvider = Provider<List<Product>>((ref) {
  final sortType = ref.watch(productSortTypeProvider);
  switch (sortType) {
    case ProductSortType.name:
      return _products.sorted((a, b) => a.name.compareTo(b.name));
    case ProductSortType.price:
      return _products.sorted((a, b) => a.price.compareTo(b.price));
  }
});
```

기존값을 편리하게 업데이트 하기

```
//문제는 없는데 두번이나 불러온다는 불편함이 있다.
ref.read(counterProvider.notifier).state = ref.read(counterProvider.notifier).state + 1;

//update를 써서 기존 state에 접근할 수 있다.
ref.read(counterProvider.notifier).update((state) => state + 1);

```

---

# Providers

프로바이더는 스테이트를 캡슐화하고 리슨을 가능하게 하는 오브젝트다.

- 다양한 곳에서 접근 가능하다. 싱글톤, 서비스 로케이터, Dependency Injection과 Inherited Widget을 대체 한다.
- 스테이트들의 combine을 간단히 한다. 오브젝트 몇개를 합치는 시나리오 자체를 예상했다.
- 성능에 좋다.
- 테스트하기 좋다.
- 로깅과 pull to refresh 같은 기능과도 잘 어울린다.

선언은 다음과 같이 한다.

```
final myProvider = Provider((ref) {
  return MyValue();
});
```

- final myProvider. the declaration of a variable. This variable is what we will use in the future to read the state of our provider. 프로바이더는 항상 final
- Provider(). It exposes an object that never changes. We could replace Provider with other providers like StreamProvider or StateNotifierProvider, to change how the value is interacted with.
- A function that creates the shared state. That function will always receive an object called ref as a parameter. This object allows us to read other providers, perform some operations when the state of our provider will be destroyed, and much more.
- function of a Provider can create any object. On the other hand, StreamProvider's callback will be expected to return a Stream.

ProviderScope를 앱에 제공해줘야 한다.

```
void main() {
  runApp(ProviderScope(child: MyApp()));
}
```

ref를 이용해 다른 프로바이더에 접근하는법

```
final counterProvider = StateNotifierProvider<Counter, int>((ref) {
  return Counter(ref);
});

class Counter extends StateNotifier<int> {
  Counter(this.ref): super(0);

  final Ref ref;

  void increment() {
    // Counter can use the "ref" to read other providers
    final repository = ref.read(repositoryProvider);
    repository.post('...');
  }
}

```
