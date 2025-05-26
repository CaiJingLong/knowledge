# Signals.dart 核心 API 详解

## 1. 基础信号 (Signal)

信号是 Signals.dart 的基础构建块，它是一个可以随时间变化的值的容器。

### 1.1 创建信号

```dart
import 'package:signals/signals.dart';

// 创建一个基本信号
final counter = signal(0);

// 创建一个带有调试标签的信号
final name = signal("张三", debugLabel: "用户名");

// 创建一个自动释放的信号
final temporaryData = signal([], autoDispose: true);

// 创建一个懒加载信号
final lazyConnection = lazySignal<DatabaseConnection>();
// 稍后设置值
lazyConnection.value = DatabaseConnection();
```

### 1.2 读取和修改信号值

```dart
// 读取信号值
print(counter.value); // 0

// 修改信号值
counter.value = 1;
print(counter.value); // 1

// 使用 set 方法修改值（可以强制更新）
counter.set(2, force: true); // 即使值相同也会触发更新
```

### 1.3 信号的特殊方法

#### peek() 方法

`peek()` 方法允许读取信号的当前值，而不会建立依赖关系：

```dart
final counter = signal(0);
final effectCount = signal(0);

effect(() {
  print(counter.value); // 建立依赖关系

  // 读取值但不建立依赖关系
  effectCount.value = effectCount.peek() + 1;
});
```

#### 信号的销毁

```dart
final counter = signal(0);

// 检查信号是否已销毁
print(counter.disposed); // false

// 手动销毁信号
counter.dispose();
print(counter.disposed); // true

// 尝试读取已销毁的信号会产生警告
print(counter.value); // 会产生警告，但仍返回最后的值

// 尝试修改已销毁的信号会抛出异常
try {
  counter.value = 1; // 抛出 SignalsWriteAfterDisposeError
} catch (e) {
  print(e);
}
```

#### 销毁回调

```dart
final counter = signal(0);

// 添加销毁回调
final cleanup = counter.onDispose(() {
  print("信号已销毁");
});

// 移除销毁回调
cleanup();

// 销毁信号
counter.dispose(); // 不会打印 "信号已销毁"，因为回调已被移除
```

## 2. 计算信号 (Computed)

计算信号是基于其他信号计算得出的信号，当依赖的信号发生变化时，计算信号会自动重新计算。

### 2.1 创建计算信号

```dart
import 'package:signals/signals.dart';

final firstName = signal("张");
final lastName = signal("三");

// 创建一个计算信号
final fullName = computed(() => "${lastName.value}${firstName.value}");

// 创建一个带有调试标签的计算信号
final greeting = computed(
  () => "你好，${fullName.value}！",
  debugLabel: "问候语",
);

// 创建一个自动释放的计算信号
final isAdult = computed(
  () => age.value >= 18,
  autoDispose: true,
);
```

### 2.2 读取计算信号

```dart
print(fullName.value); // 张三

// 修改依赖的信号会导致计算信号更新
firstName.value = "四";
print(fullName.value); // 张四
```

### 2.3 计算信号的特殊方法

#### 强制重新计算

```dart
// 强制重新计算，即使依赖的信号没有变化
fullName.recompute();
```

#### 计算信号的销毁

```dart
// 检查计算信号是否已销毁
print(fullName.disposed); // false

// 手动销毁计算信号
fullName.dispose();
print(fullName.disposed); // true

// 销毁后，计算信号不再响应依赖的变化
firstName.value = "五"; // fullName 不会更新
```

## 3. 效果 (Effect)

效果是响应式系统的最后一个关键部分，它允许你在信号变化时执行副作用。

### 3.1 创建效果

```dart
import 'package:signals/signals.dart';

final counter = signal(0);

// 创建一个基本效果
final dispose = effect(() {
  print("计数器的值是: ${counter.value}");
});

// 创建一个带有调试标签的效果
final disposeWithLabel = effect(
  () {
    print("计数器的值是: ${counter.value}");
  },
  debugLabel: "计数器监视器",
);

// 创建一个带有销毁回调的效果
final disposeWithCleanup = effect(
  () {
    print("计数器的值是: ${counter.value}");
  },
  onDispose: () {
    print("效果已销毁");
  },
);
```

### 3.2 效果的清理函数

效果的回调函数可以返回一个清理函数，该函数将在效果重新执行前或销毁时调用：

```dart
final timer = signal(0);

final dispose = effect(() {
  print("定时器开始: ${timer.value}");
  
  // 返回一个清理函数
  return () {
    print("清理上一次的效果");
  };
});

// 修改信号会触发效果，并在重新执行前调用清理函数
timer.value = 1;
// 输出:
// 清理上一次的效果
// 定时器开始: 1

// 销毁效果也会调用清理函数
dispose();
// 输出: 清理上一次的效果
```

### 3.3 避免循环依赖

在效果内修改它依赖的信号会导致无限循环。为了避免这种情况，可以使用 `untracked` 函数：

```dart
import 'package:signals/signals.dart';

final counter = signal(0);

// 错误示例：会导致无限循环
// effect(() {
//   print(counter.value);
//   counter.value++; // 这会导致循环
// });

// 正确示例：使用 untracked
effect(() {
  print(counter.value);
  
  // 使用 untracked 读取值，不建立依赖关系
  final currentValue = untracked(() => counter.value);
  counter.value = currentValue + 1; // 不会导致循环
});
```

## 4. 批处理 (Batch)

批处理允许你一次性更新多个信号，而只触发一次更新：

```dart
import 'package:signals/signals.dart';

final firstName = signal("张");
final lastName = signal("三");
final age = signal(25);

// 创建一个依赖于多个信号的计算信号
final profile = computed(() => 
  "${lastName.value}${firstName.value}, ${age.value}岁"
);

// 监听 profile 的变化
effect(() {
  print("个人资料更新: ${profile.value}");
});

// 不使用批处理，会触发多次更新
firstName.value = "四"; // 触发一次更新
lastName.value = "李"; // 触发一次更新
age.value = 30; // 触发一次更新

// 使用批处理，只触发一次更新
batch(() {
  firstName.value = "五";
  lastName.value = "王";
  age.value = 35;
}); // 只触发一次更新
```

## 5. 取消追踪 (Untracked)

`untracked` 函数允许你在不建立依赖关系的情况下读取信号的值：

```dart
import 'package:signals/signals.dart';

final counter = signal(0);

effect(() {
  // 正常读取，建立依赖关系
  print("计数器 (tracked): ${counter.value}");
  
  // 使用 untracked 读取，不建立依赖关系
  print("计数器 (untracked): ${untracked(() => counter.value)}");
});

// 修改信号会触发效果，但只是因为第一次读取建立了依赖关系
counter.value = 1;
```

## 6. 只读信号 (ReadonlySignal)

只读信号允许你暴露一个信号的值，但不允许修改它：

```dart
import 'package:signals/signals.dart';

class Counter {
  final _count = signal(0);
  
  // 暴露一个只读信号
  ReadonlySignal<int> get count => _count.readonly();
  
  void increment() {
    _count.value++;
  }
}

final counter = Counter();
print(counter.count.value); // 0
counter.increment();
print(counter.count.value); // 1

// 尝试直接修改会导致编译错误
// counter.count.value = 10; // 错误：只读信号不能被修改
```

## 7. 信号容器 (SignalContainer)

信号容器是一种特殊的类，它可以包含多个信号，并提供一些便捷的方法来管理它们：

```dart
import 'package:signals/signals.dart';

// 创建一个信号容器
class UserState extends SignalContainer {
  final name = signal("张三");
  final age = signal(25);
  final isLoggedIn = signal(false);
  
  // 自定义方法
  void login() {
    isLoggedIn.value = true;
  }
  
  void logout() {
    isLoggedIn.value = false;
  }
  
  // 重写 dispose 方法
  @override
  void dispose() {
    print("用户状态被销毁");
    super.dispose(); // 会自动销毁所有信号
  }
}

final user = UserState();
print(user.name.value); // 张三
user.login();
print(user.isLoggedIn.value); // true

// 销毁容器会自动销毁所有信号
user.dispose();
```

## 8. 异步信号

Signals.dart 提供了多种处理异步操作的信号类型。

### 8.1 Future 信号

```dart
import 'package:signals/signals.dart';

// 创建一个 Future 信号
final userFuture = futureSignal<User>(fetchUser());

// 监听状态变化
effect(() {
  final state = userFuture.state;
  if (state.isLoading) {
    print("加载中...");
  } else if (state.hasError) {
    print("错误: ${state.error}");
  } else if (state.hasValue) {
    print("用户: ${state.value.name}");
  }
});

// 重新加载
userFuture.reload(fetchUser());
```

### 8.2 Stream 信号

```dart
import 'package:signals/signals.dart';

// 创建一个 Stream 信号
final counterStream = streamSignal<int>(counterUpdates());

// 监听状态变化
effect(() {
  final state = counterStream.state;
  if (state.isLoading) {
    print("等待数据...");
  } else if (state.hasError) {
    print("错误: ${state.error}");
  } else if (state.hasValue) {
    print("计数器: ${state.value}");
  }
});

// 重新连接
counterStream.reconnect(newCounterUpdates());
```

## 9. 集合信号

Signals.dart 提供了专门用于处理集合的信号类型。

### 9.1 列表信号

```dart
import 'package:signals/signals.dart';

// 创建一个列表信号
final todos = listSignal<String>(['学习 Dart', '学习 Flutter']);

// 添加项
todos.add('学习 Signals');
print(todos.value); // ['学习 Dart', '学习 Flutter', '学习 Signals']

// 删除项
todos.removeAt(0);
print(todos.value); // ['学习 Flutter', '学习 Signals']

// 更新项
todos[0] = '深入学习 Flutter';
print(todos.value); // ['深入学习 Flutter', '学习 Signals']

// 批量操作
todos.update((list) {
  list.clear();
  list.addAll(['新任务 1', '新任务 2']);
});
print(todos.value); // ['新任务 1', '新任务 2']
```

### 9.2 映射信号

```dart
import 'package:signals/signals.dart';

// 创建一个映射信号
final userInfo = mapSignal<String, dynamic>({
  'name': '张三',
  'age': 25,
});

// 添加或更新键值对
userInfo['email'] = 'zhangsan@example.com';
print(userInfo.value); // {'name': '张三', 'age': 25, 'email': 'zhangsan@example.com'}

// 删除键值对
userInfo.remove('age');
print(userInfo.value); // {'name': '张三', 'email': 'zhangsan@example.com'}

// 批量操作
userInfo.update((map) {
  map.clear();
  map.addAll({'id': 1, 'username': 'zhangsan'});
});
print(userInfo.value); // {'id': 1, 'username': 'zhangsan'}
```

### 9.3 集合信号

```dart
import 'package:signals/signals.dart';

// 创建一个集合信号
final uniqueTags = setSignal<String>({'dart', 'flutter'});

// 添加项
uniqueTags.add('signals');
print(uniqueTags.value); // {'dart', 'flutter', 'signals'}

// 删除项
uniqueTags.remove('dart');
print(uniqueTags.value); // {'flutter', 'signals'}

// 批量操作
uniqueTags.update((set) {
  set.clear();
  set.addAll({'mobile', 'web'});
});
print(uniqueTags.value); // {'mobile', 'web'}
```

## 10. 总结

Signals.dart 提供了丰富的 API 来处理各种响应式编程场景。通过组合使用这些 API，你可以构建出复杂的响应式应用程序，同时保持代码的简洁和可维护性。在下一篇文章中，我们将探讨如何在 Flutter 应用程序中使用 Signals.dart。
