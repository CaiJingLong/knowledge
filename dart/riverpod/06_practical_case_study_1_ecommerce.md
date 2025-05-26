# Riverpod 源码分析：实践案例研究 1 - 电子商务应用

## 引言

在前几篇文章中，我们深入分析了 Riverpod 的核心概念、各类提供者、修饰符以及高级交互和测试功能。本文将通过一个电子商务应用的案例，展示如何在实际项目中应用 Riverpod 来管理产品列表、购物车和用户认证状态。

## 案例概述

我们将构建一个简化的电子商务应用，包含以下功能：

1. **产品列表**：从 API 获取产品数据并展示
2. **购物车**：添加、删除商品和计算总价
3. **用户认证**：登录、注册和会话管理

通过这个案例，我们将展示如何使用 Riverpod 的各种功能来管理不同类型的状态，以及如何组织代码以提高可维护性。

## 数据模型

首先，我们定义应用程序的数据模型：

```dart
// 产品模型
class Product {
  final String id;
  final String name;
  final String description;
  final double price;
  final String imageUrl;
  
  Product({
    required this.id,
    required this.name,
    required this.description,
    required this.price,
    required this.imageUrl,
  });
  
  // 从 JSON 创建产品
  factory Product.fromJson(Map<String, dynamic> json) {
    return Product(
      id: json['id'],
      name: json['name'],
      description: json['description'],
      price: json['price'].toDouble(),
      imageUrl: json['imageUrl'],
    );
  }
}

// 购物车项模型
class CartItem {
  final Product product;
  final int quantity;
  
  CartItem({
    required this.product,
    required this.quantity,
  });
  
  // 计算小计
  double get subtotal => product.price * quantity;
}

// 用户模型
class User {
  final String id;
  final String email;
  final String name;
  
  User({
    required this.id,
    required this.email,
    required this.name,
  });
  
  // 从 JSON 创建用户
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'],
      email: json['email'],
      name: json['name'],
    );
  }
}
```

## 产品列表管理

### API 服务

首先，我们创建一个 API 服务来获取产品数据：

```dart
// API 服务接口
abstract class ApiService {
  Future<List<Product>> getProducts();
  Future<Product> getProduct(String id);
}

// API 服务实现
class ApiServiceImpl implements ApiService {
  final Dio _dio;
  
  ApiServiceImpl(this._dio);
  
  @override
  Future<List<Product>> getProducts() async {
    final response = await _dio.get('/products');
    final List<dynamic> data = response.data;
    return data.map((json) => Product.fromJson(json)).toList();
  }
  
  @override
  Future<Product> getProduct(String id) async {
    final response = await _dio.get('/products/$id');
    return Product.fromJson(response.data);
  }
}

// API 服务提供者
final apiServiceProvider = Provider<ApiService>((ref) {
  final dio = Dio(BaseOptions(baseUrl: 'https://api.example.com'));
  return ApiServiceImpl(dio);
});
```

### 产品仓库

接下来，我们创建一个产品仓库来管理产品数据：

```dart
// 产品仓库接口
abstract class ProductRepository {
  Future<List<Product>> getProducts();
  Future<Product> getProduct(String id);
}

// 产品仓库实现
class ProductRepositoryImpl implements ProductRepository {
  final ApiService _apiService;
  
  ProductRepositoryImpl(this._apiService);
  
  @override
  Future<List<Product>> getProducts() {
    return _apiService.getProducts();
  }
  
  @override
  Future<Product> getProduct(String id) {
    return _apiService.getProduct(id);
  }
}

// 产品仓库提供者
final productRepositoryProvider = Provider<ProductRepository>((ref) {
  final apiService = ref.watch(apiServiceProvider);
  return ProductRepositoryImpl(apiService);
});
```

### 产品提供者

现在，我们可以创建产品相关的提供者：

```dart
// 所有产品的提供者
final productsProvider = FutureProvider<List<Product>>((ref) async {
  final repository = ref.watch(productRepositoryProvider);
  return repository.getProducts();
});

// 单个产品的提供者
final productProvider = FutureProvider.family<Product, String>((ref, id) async {
  final repository = ref.watch(productRepositoryProvider);
  return repository.getProduct(id);
});

// 产品搜索提供者
final productSearchProvider = StateProvider<String>((ref) => '');

// 过滤后的产品提供者
final filteredProductsProvider = Provider<AsyncValue<List<Product>>>((ref) {
  final productsAsync = ref.watch(productsProvider);
  final searchQuery = ref.watch(productSearchProvider).toLowerCase();
  
  return productsAsync.when(
    loading: () => const AsyncLoading(),
    error: (error, stackTrace) => AsyncError(error, stackTrace),
    data: (products) {
      if (searchQuery.isEmpty) {
        return AsyncData(products);
      }
      
      final filteredProducts = products.where((product) {
        return product.name.toLowerCase().contains(searchQuery) ||
               product.description.toLowerCase().contains(searchQuery);
      }).toList();
      
      return AsyncData(filteredProducts);
    },
  );
});
```

## 购物车管理

### 购物车 Notifier

我们使用 `Notifier` 来管理购物车状态：

```dart
// 购物车状态
class CartState {
  final List<CartItem> items;
  
  CartState({required this.items});
  
  // 计算总价
  double get total => items.fold(0, (sum, item) => sum + item.subtotal);
  
  // 获取商品数量
  int get itemCount => items.fold(0, (sum, item) => sum + item.quantity);
  
  // 创建新状态
  CartState copyWith({List<CartItem>? items}) {
    return CartState(items: items ?? this.items);
  }
}

// 购物车 Notifier
class CartNotifier extends Notifier<CartState> {
  @override
  CartState build() {
    // 初始状态
    return CartState(items: []);
  }
  
  // 添加商品到购物车
  void addItem(Product product, {int quantity = 1}) {
    final items = [...state.items];
    final index = items.indexWhere((item) => item.product.id == product.id);
    
    if (index >= 0) {
      // 商品已在购物车中，增加数量
      final item = items[index];
      items[index] = CartItem(
        product: item.product,
        quantity: item.quantity + quantity,
      );
    } else {
      // 商品不在购物车中，添加新项
      items.add(CartItem(
        product: product,
        quantity: quantity,
      ));
    }
    
    state = state.copyWith(items: items);
  }
  
  // 从购物车中移除商品
  void removeItem(String productId) {
    final items = state.items.where((item) => item.product.id != productId).toList();
    state = state.copyWith(items: items);
  }
  
  // 更新商品数量
  void updateQuantity(String productId, int quantity) {
    if (quantity <= 0) {
      removeItem(productId);
      return;
    }
    
    final items = [...state.items];
    final index = items.indexWhere((item) => item.product.id == productId);
    
    if (index >= 0) {
      items[index] = CartItem(
        product: items[index].product,
        quantity: quantity,
      );
      state = state.copyWith(items: items);
    }
  }
  
  // 清空购物车
  void clear() {
    state = CartState(items: []);
  }
}

// 购物车提供者
final cartProvider = NotifierProvider<CartNotifier, CartState>(() {
  return CartNotifier();
});
```

## 用户认证管理

### 认证服务

首先，我们创建一个认证服务来处理用户认证：

```dart
// 认证服务接口
abstract class AuthService {
  Future<User> login(String email, String password);
  Future<User> register(String email, String password, String name);
  Future<void> logout();
}

// 认证服务实现
class AuthServiceImpl implements AuthService {
  final Dio _dio;
  
  AuthServiceImpl(this._dio);
  
  @override
  Future<User> login(String email, String password) async {
    final response = await _dio.post('/auth/login', data: {
      'email': email,
      'password': password,
    });
    
    return User.fromJson(response.data['user']);
  }
  
  @override
  Future<User> register(String email, String password, String name) async {
    final response = await _dio.post('/auth/register', data: {
      'email': email,
      'password': password,
      'name': name,
    });
    
    return User.fromJson(response.data['user']);
  }
  
  @override
  Future<void> logout() async {
    await _dio.post('/auth/logout');
  }
}

// 认证服务提供者
final authServiceProvider = Provider<AuthService>((ref) {
  final dio = ref.watch(apiServiceProvider)._dio;
  return AuthServiceImpl(dio);
});
```

### 认证状态 Notifier

我们使用 `AsyncNotifier` 来管理认证状态：

```dart
// 认证状态
class AuthState {
  final User? user;
  final String? token;
  
  AuthState({this.user, this.token});
  
  bool get isAuthenticated => user != null && token != null;
  
  AuthState copyWith({User? user, String? token}) {
    return AuthState(
      user: user ?? this.user,
      token: token ?? this.token,
    );
  }
}

// 认证 Notifier
class AuthNotifier extends AsyncNotifier<AuthState> {
  @override
  Future<AuthState> build() async {
    // 从本地存储加载认证状态
    final prefs = await SharedPreferences.getInstance();
    final token = prefs.getString('auth_token');
    
    if (token == null) {
      return AuthState();
    }
    
    try {
      // 验证令牌并获取用户信息
      final dio = ref.read(apiServiceProvider)._dio;
      dio.options.headers['Authorization'] = 'Bearer $token';
      
      final response = await dio.get('/auth/me');
      final user = User.fromJson(response.data);
      
      return AuthState(user: user, token: token);
    } catch (e) {
      // 令牌无效，清除本地存储
      await prefs.remove('auth_token');
      return AuthState();
    }
  }
  
  // 登录
  Future<void> login(String email, String password) async {
    state = const AsyncLoading();
    
    try {
      final authService = ref.read(authServiceProvider);
      final user = await authService.login(email, password);
      
      // 保存令牌到本地存储
      final prefs = await SharedPreferences.getInstance();
      final token = 'generated_token'; // 实际应用中应从响应中获取
      await prefs.setString('auth_token', token);
      
      state = AsyncData(AuthState(user: user, token: token));
    } catch (e, stack) {
      state = AsyncError(e, stack);
    }
  }
  
  // 注册
  Future<void> register(String email, String password, String name) async {
    state = const AsyncLoading();
    
    try {
      final authService = ref.read(authServiceProvider);
      final user = await authService.register(email, password, name);
      
      // 保存令牌到本地存储
      final prefs = await SharedPreferences.getInstance();
      final token = 'generated_token'; // 实际应用中应从响应中获取
      await prefs.setString('auth_token', token);
      
      state = AsyncData(AuthState(user: user, token: token));
    } catch (e, stack) {
      state = AsyncError(e, stack);
    }
  }
  
  // 登出
  Future<void> logout() async {
    state = const AsyncLoading();
    
    try {
      final authService = ref.read(authServiceProvider);
      await authService.logout();
      
      // 清除本地存储
      final prefs = await SharedPreferences.getInstance();
      await prefs.remove('auth_token');
      
      state = AsyncData(AuthState());
    } catch (e, stack) {
      state = AsyncError(e, stack);
    }
  }
}

// 认证提供者
final authProvider = AsyncNotifierProvider<AuthNotifier, AuthState>(() {
  return AuthNotifier();
});
```

## UI 实现

现在，我们可以使用这些提供者来构建 UI：

### 产品列表页面

```dart
class ProductListPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final searchQuery = ref.watch(productSearchProvider);
    final productsAsync = ref.watch(filteredProductsProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: Text('Products'),
        actions: [
          IconButton(
            icon: Icon(Icons.shopping_cart),
            onPressed: () {
              Navigator.push(
                context,
                MaterialPageRoute(builder: (_) => CartPage()),
              );
            },
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: TextField(
              decoration: InputDecoration(
                labelText: 'Search',
                prefixIcon: Icon(Icons.search),
                border: OutlineInputBorder(),
              ),
              onChanged: (value) {
                ref.read(productSearchProvider.notifier).state = value;
              },
              value: searchQuery,
            ),
          ),
          Expanded(
            child: productsAsync.when(
              loading: () => Center(child: CircularProgressIndicator()),
              error: (error, stack) => Center(child: Text('Error: $error')),
              data: (products) {
                if (products.isEmpty) {
                  return Center(child: Text('No products found'));
                }
                
                return GridView.builder(
                  gridDelegate: SliverGridDelegateWithFixedCrossAxisCount(
                    crossAxisCount: 2,
                    childAspectRatio: 0.7,
                  ),
                  itemCount: products.length,
                  itemBuilder: (context, index) {
                    final product = products[index];
                    return ProductCard(product: product);
                  },
                );
              },
            ),
          ),
        ],
      ),
    );
  }
}

class ProductCard extends ConsumerWidget {
  final Product product;
  
  ProductCard({required this.product});
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Card(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Expanded(
            child: Image.network(
              product.imageUrl,
              fit: BoxFit.cover,
              width: double.infinity,
            ),
          ),
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  product.name,
                  style: TextStyle(fontWeight: FontWeight.bold),
                  maxLines: 1,
                  overflow: TextOverflow.ellipsis,
                ),
                SizedBox(height: 4),
                Text(
                  '\$${product.price.toStringAsFixed(2)}',
                  style: TextStyle(color: Colors.green),
                ),
                SizedBox(height: 8),
                ElevatedButton(
                  onPressed: () {
                    ref.read(cartProvider.notifier).addItem(product);
                    ScaffoldMessenger.of(context).showSnackBar(
                      SnackBar(content: Text('${product.name} added to cart')),
                    );
                  },
                  child: Text('Add to Cart'),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

### 购物车页面

```dart
class CartPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cart = ref.watch(cartProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: Text('Cart'),
      ),
      body: cart.items.isEmpty
          ? Center(child: Text('Your cart is empty'))
          : ListView.builder(
              itemCount: cart.items.length,
              itemBuilder: (context, index) {
                final item = cart.items[index];
                return CartItemWidget(item: item);
              },
            ),
      bottomNavigationBar: BottomAppBar(
        child: Padding(
          padding: const EdgeInsets.all(16.0),
          child: Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Text(
                'Total: \$${cart.total.toStringAsFixed(2)}',
                style: TextStyle(
                  fontWeight: FontWeight.bold,
                  fontSize: 18,
                ),
              ),
              ElevatedButton(
                onPressed: cart.items.isEmpty ? null : () {
                  // 处理结账逻辑
                },
                child: Text('Checkout'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class CartItemWidget extends ConsumerWidget {
  final CartItem item;
  
  CartItemWidget({required this.item});
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return ListTile(
      leading: Image.network(
        item.product.imageUrl,
        width: 50,
        height: 50,
        fit: BoxFit.cover,
      ),
      title: Text(item.product.name),
      subtitle: Text('\$${item.product.price.toStringAsFixed(2)} x ${item.quantity}'),
      trailing: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          IconButton(
            icon: Icon(Icons.remove),
            onPressed: () {
              ref.read(cartProvider.notifier).updateQuantity(
                item.product.id,
                item.quantity - 1,
              );
            },
          ),
          Text('${item.quantity}'),
          IconButton(
            icon: Icon(Icons.add),
            onPressed: () {
              ref.read(cartProvider.notifier).updateQuantity(
                item.product.id,
                item.quantity + 1,
              );
            },
          ),
          IconButton(
            icon: Icon(Icons.delete),
            onPressed: () {
              ref.read(cartProvider.notifier).removeItem(item.product.id);
            },
          ),
        ],
      ),
    );
  }
}
```

### 登录页面

```dart
class LoginPage extends ConsumerWidget {
  final _formKey = GlobalKey<FormState>();
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authAsync = ref.watch(authProvider);
    
    ref.listen<AsyncValue<AuthState>>(authProvider, (previous, next) {
      next.whenData((state) {
        if (state.isAuthenticated) {
          Navigator.pushReplacement(
            context,
            MaterialPageRoute(builder: (_) => HomePage()),
          );
        }
      });
    });
    
    return Scaffold(
      appBar: AppBar(
        title: Text('Login'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              TextFormField(
                controller: _emailController,
                decoration: InputDecoration(labelText: 'Email'),
                keyboardType: TextInputType.emailAddress,
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'Please enter your email';
                  }
                  return null;
                },
              ),
              SizedBox(height: 16),
              TextFormField(
                controller: _passwordController,
                decoration: InputDecoration(labelText: 'Password'),
                obscureText: true,
                validator: (value) {
                  if (value == null || value.isEmpty) {
                    return 'Please enter your password';
                  }
                  return null;
                },
              ),
              SizedBox(height: 24),
              ElevatedButton(
                onPressed: authAsync.isLoading
                    ? null
                    : () {
                        if (_formKey.currentState!.validate()) {
                          ref.read(authProvider.notifier).login(
                                _emailController.text,
                                _passwordController.text,
                              );
                        }
                      },
                child: authAsync.isLoading
                    ? CircularProgressIndicator()
                    : Text('Login'),
              ),
              SizedBox(height: 16),
              TextButton(
                onPressed: () {
                  Navigator.push(
                    context,
                    MaterialPageRoute(builder: (_) => RegisterPage()),
                  );
                },
                child: Text('Don\'t have an account? Register'),
              ),
              if (authAsync.hasError)
                Padding(
                  padding: const EdgeInsets.only(top: 16),
                  child: Text(
                    'Error: ${authAsync.error}',
                    style: TextStyle(color: Colors.red),
                  ),
                ),
            ],
          ),
        ),
      ),
    );
  }
}
```

## 状态管理的优势

通过这个案例，我们可以看到 Riverpod 在状态管理方面的几个优势：

1. **模块化**：每个功能模块（产品、购物车、认证）都有自己的状态管理，使代码更加模块化和可维护。

2. **依赖注入**：通过 Provider 可以轻松实现依赖注入，例如 `apiServiceProvider` 被注入到 `productRepositoryProvider` 中。

3. **状态派生**：可以通过组合多个 Provider 来派生新的状态，例如 `filteredProductsProvider` 依赖于 `productsProvider` 和 `productSearchProvider`。

4. **异步处理**：Riverpod 提供了优雅的异步状态处理机制，例如 `AsyncValue` 和 `AsyncNotifier`，使得处理加载状态和错误变得简单。

5. **状态隔离**：每个 Provider 都有自己的状态，不会相互干扰，避免了全局状态的问题。

## 小结

通过这个电子商务应用的案例，我们展示了如何使用 Riverpod 来管理不同类型的状态：

1. **产品列表**：使用 `FutureProvider` 和 `Provider` 来获取和过滤产品数据
2. **购物车**：使用 `NotifierProvider` 来管理购物车状态和操作
3. **用户认证**：使用 `AsyncNotifierProvider` 来处理异步认证状态

这个案例展示了 Riverpod 在实际应用中的强大功能和灵活性，以及如何组织代码以提高可维护性。

在下一篇文章中，我们将通过另一个案例研究，展示如何使用 Riverpod 处理登录和 HTTP 请求，包括表单、API 调用和会话管理。
