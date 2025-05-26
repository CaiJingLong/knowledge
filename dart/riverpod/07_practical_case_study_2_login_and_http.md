# Riverpod 源码分析：实践案例研究 2 - 登录与 HTTP 请求

## 引言

在上一篇文章中，我们通过电子商务应用的案例展示了如何使用 Riverpod 管理产品列表、购物车和用户认证状态。本文将深入探讨另一个常见场景：登录表单和 HTTP 请求的处理，包括表单状态管理、API 调用和会话管理。

## 案例概述

我们将构建一个简化的登录系统，包含以下功能：

1. **登录表单**：表单验证和状态管理
2. **HTTP 请求**：处理 API 调用和响应
3. **会话管理**：存储和刷新认证令牌

通过这个案例，我们将展示如何使用 Riverpod 处理表单状态、异步操作和持久化数据。

## 数据模型

首先，我们定义应用程序的数据模型：

```dart
// 用户模型
class User {
  final String id;
  final String email;
  final String name;
  final String? avatarUrl;
  
  User({
    required this.id,
    required this.email,
    required this.name,
    this.avatarUrl,
  });
  
  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'],
      email: json['email'],
      name: json['name'],
      avatarUrl: json['avatar_url'],
    );
  }
}

// 认证令牌模型
class AuthToken {
  final String accessToken;
  final String refreshToken;
  final DateTime expiresAt;
  
  AuthToken({
    required this.accessToken,
    required this.refreshToken,
    required this.expiresAt,
  });
  
  bool get isExpired => DateTime.now().isAfter(expiresAt);
  
  factory AuthToken.fromJson(Map<String, dynamic> json) {
    return AuthToken(
      accessToken: json['access_token'],
      refreshToken: json['refresh_token'],
      expiresAt: DateTime.parse(json['expires_at']),
    );
  }
  
  Map<String, dynamic> toJson() {
    return {
      'access_token': accessToken,
      'refresh_token': refreshToken,
      'expires_at': expiresAt.toIso8601String(),
    };
  }
}
```

## HTTP 客户端

我们使用 Dio 作为 HTTP 客户端，并创建一个带有拦截器的客户端来处理认证令牌：

```dart
// HTTP 客户端提供者
final httpClientProvider = Provider<Dio>((ref) {
  final dio = Dio(BaseOptions(
    baseUrl: 'https://api.example.com',
    connectTimeout: Duration(seconds: 5),
    receiveTimeout: Duration(seconds: 3),
  ));
  
  // 添加日志拦截器
  dio.interceptors.add(LogInterceptor(
    requestBody: true,
    responseBody: true,
  ));
  
  return dio;
});

// 认证 HTTP 客户端提供者
final authHttpClientProvider = Provider<Dio>((ref) {
  final dio = ref.watch(httpClientProvider);
  final authState = ref.watch(authProvider);
  
  // 添加认证拦截器
  dio.interceptors.add(InterceptorsWrapper(
    onRequest: (options, handler) {
      // 如果有认证令牌，添加到请求头
      authState.whenData((state) {
        if (state.token != null && !state.token!.isExpired) {
          options.headers['Authorization'] = 'Bearer ${state.token!.accessToken}';
        }
      });
      return handler.next(options);
    },
    onError: (error, handler) async {
      // 如果是 401 错误，尝试刷新令牌
      if (error.response?.statusCode == 401) {
        try {
          // 刷新令牌
          await ref.read(authProvider.notifier).refreshToken();
          
          // 重试请求
          final response = await dio.fetch(error.requestOptions);
          return handler.resolve(response);
        } catch (e) {
          // 刷新令牌失败，登出用户
          await ref.read(authProvider.notifier).logout();
        }
      }
      return handler.next(error);
    },
  ));
  
  return dio;
});
```

## 认证服务

接下来，我们创建一个认证服务来处理登录、注册和令牌刷新：

```dart
// 认证服务接口
abstract class AuthService {
  Future<(User, AuthToken)> login(String email, String password);
  Future<(User, AuthToken)> register(String email, String password, String name);
  Future<AuthToken> refreshToken(String refreshToken);
  Future<void> logout(String accessToken);
}

// 认证服务实现
class AuthServiceImpl implements AuthService {
  final Dio _dio;
  
  AuthServiceImpl(this._dio);
  
  @override
  Future<(User, AuthToken)> login(String email, String password) async {
    final response = await _dio.post('/auth/login', data: {
      'email': email,
      'password': password,
    });
    
    final user = User.fromJson(response.data['user']);
    final token = AuthToken.fromJson(response.data['token']);
    
    return (user, token);
  }
  
  @override
  Future<(User, AuthToken)> register(String email, String password, String name) async {
    final response = await _dio.post('/auth/register', data: {
      'email': email,
      'password': password,
      'name': name,
    });
    
    final user = User.fromJson(response.data['user']);
    final token = AuthToken.fromJson(response.data['token']);
    
    return (user, token);
  }
  
  @override
  Future<AuthToken> refreshToken(String refreshToken) async {
    final response = await _dio.post('/auth/refresh', data: {
      'refresh_token': refreshToken,
    });
    
    return AuthToken.fromJson(response.data['token']);
  }
  
  @override
  Future<void> logout(String accessToken) async {
    await _dio.post('/auth/logout', data: {
      'access_token': accessToken,
    });
  }
}

// 认证服务提供者
final authServiceProvider = Provider<AuthService>((ref) {
  final dio = ref.watch(httpClientProvider);
  return AuthServiceImpl(dio);
});
```

## 本地存储服务

我们需要一个本地存储服务来保存和检索认证令牌：

```dart
// 本地存储服务接口
abstract class StorageService {
  Future<void> saveToken(AuthToken token);
  Future<AuthToken?> getToken();
  Future<void> deleteToken();
}

// 本地存储服务实现（使用 shared_preferences）
class StorageServiceImpl implements StorageService {
  static const String _tokenKey = 'auth_token';
  
  @override
  Future<void> saveToken(AuthToken token) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString(_tokenKey, jsonEncode(token.toJson()));
  }
  
  @override
  Future<AuthToken?> getToken() async {
    final prefs = await SharedPreferences.getInstance();
    final tokenJson = prefs.getString(_tokenKey);
    
    if (tokenJson == null) {
      return null;
    }
    
    try {
      return AuthToken.fromJson(jsonDecode(tokenJson));
    } catch (e) {
      await deleteToken();
      return null;
    }
  }
  
  @override
  Future<void> deleteToken() async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove(_tokenKey);
  }
}

// 本地存储服务提供者
final storageServiceProvider = Provider<StorageService>((ref) {
  return StorageServiceImpl();
});
```

## 认证状态管理

现在，我们可以创建一个 `AsyncNotifier` 来管理认证状态：

```dart
// 认证状态
class AuthState {
  final User? user;
  final AuthToken? token;
  
  AuthState({this.user, this.token});
  
  bool get isAuthenticated => user != null && token != null && !token!.isExpired;
  
  AuthState copyWith({User? user, AuthToken? token}) {
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
    // 从本地存储加载认证令牌
    final storageService = ref.read(storageServiceProvider);
    final token = await storageService.getToken();
    
    if (token == null || token.isExpired) {
      return AuthState();
    }
    
    try {
      // 验证令牌并获取用户信息
      final dio = ref.read(authHttpClientProvider);
      final response = await dio.get('/auth/me');
      final user = User.fromJson(response.data);
      
      return AuthState(user: user, token: token);
    } catch (e) {
      // 令牌无效，清除本地存储
      await storageService.deleteToken();
      return AuthState();
    }
  }
  
  // 登录
  Future<void> login(String email, String password) async {
    state = const AsyncLoading();
    
    try {
      final authService = ref.read(authServiceProvider);
      final (user, token) = await authService.login(email, password);
      
      // 保存令牌到本地存储
      final storageService = ref.read(storageServiceProvider);
      await storageService.saveToken(token);
      
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
      final (user, token) = await authService.register(email, password, name);
      
      // 保存令牌到本地存储
      final storageService = ref.read(storageServiceProvider);
      await storageService.saveToken(token);
      
      state = AsyncData(AuthState(user: user, token: token));
    } catch (e, stack) {
      state = AsyncError(e, stack);
    }
  }
  
  // 刷新令牌
  Future<void> refreshToken() async {
    final currentState = state.value;
    if (currentState == null || currentState.token == null) {
      throw Exception('No token to refresh');
    }
    
    try {
      final authService = ref.read(authServiceProvider);
      final newToken = await authService.refreshToken(currentState.token!.refreshToken);
      
      // 保存新令牌到本地存储
      final storageService = ref.read(storageServiceProvider);
      await storageService.saveToken(newToken);
      
      state = AsyncData(currentState.copyWith(token: newToken));
    } catch (e, stack) {
      // 刷新令牌失败，清除状态
      await logout();
      throw e;
    }
  }
  
  // 登出
  Future<void> logout() async {
    final currentState = state.value;
    if (currentState == null || currentState.token == null) {
      state = AsyncData(AuthState());
      return;
    }
    
    state = const AsyncLoading();
    
    try {
      final authService = ref.read(authServiceProvider);
      await authService.logout(currentState.token!.accessToken);
      
      // 清除本地存储
      final storageService = ref.read(storageServiceProvider);
      await storageService.deleteToken();
      
      state = AsyncData(AuthState());
    } catch (e, stack) {
      // 即使登出 API 调用失败，也清除本地状态
      final storageService = ref.read(storageServiceProvider);
      await storageService.deleteToken();
      
      state = AsyncData(AuthState());
    }
  }
}

// 认证提供者
final authProvider = AsyncNotifierProvider<AuthNotifier, AuthState>(() {
  return AuthNotifier();
});
```

## 表单状态管理

我们使用 `StateNotifier` 来管理登录表单的状态：

```dart
// 表单状态
class LoginFormState {
  final String email;
  final String password;
  final bool isEmailValid;
  final bool isPasswordValid;
  final bool isSubmitting;
  
  LoginFormState({
    this.email = '',
    this.password = '',
    this.isEmailValid = true,
    this.isPasswordValid = true,
    this.isSubmitting = false,
  });
  
  bool get isValid => isEmailValid && isPasswordValid && email.isNotEmpty && password.isNotEmpty;
  
  LoginFormState copyWith({
    String? email,
    String? password,
    bool? isEmailValid,
    bool? isPasswordValid,
    bool? isSubmitting,
  }) {
    return LoginFormState(
      email: email ?? this.email,
      password: password ?? this.password,
      isEmailValid: isEmailValid ?? this.isEmailValid,
      isPasswordValid: isPasswordValid ?? this.isPasswordValid,
      isSubmitting: isSubmitting ?? this.isSubmitting,
    );
  }
}

// 表单 Notifier
class LoginFormNotifier extends Notifier<LoginFormState> {
  @override
  LoginFormState build() {
    return LoginFormState();
  }
  
  // 更新邮箱
  void updateEmail(String email) {
    final isValid = RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(email);
    state = state.copyWith(
      email: email,
      isEmailValid: isValid,
    );
  }
  
  // 更新密码
  void updatePassword(String password) {
    final isValid = password.length >= 6;
    state = state.copyWith(
      password: password,
      isPasswordValid: isValid,
    );
  }
  
  // 提交表单
  Future<void> submit() async {
    if (!state.isValid || state.isSubmitting) {
      return;
    }
    
    state = state.copyWith(isSubmitting: true);
    
    try {
      await ref.read(authProvider.notifier).login(
        state.email,
        state.password,
      );
      
      // 重置表单
      state = LoginFormState();
    } catch (e) {
      // 提交失败，重置提交状态
      state = state.copyWith(isSubmitting: false);
      rethrow;
    }
  }
  
  // 重置表单
  void reset() {
    state = LoginFormState();
  }
}

// 登录表单提供者
final loginFormProvider = NotifierProvider<LoginFormNotifier, LoginFormState>(() {
  return LoginFormNotifier();
});
```

## UI 实现

现在，我们可以使用这些提供者来构建 UI：

### 登录页面

```dart
class LoginPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final formState = ref.watch(loginFormProvider);
    final authState = ref.watch(authProvider);
    
    // 监听认证状态变化
    ref.listen<AsyncValue<AuthState>>(authProvider, (previous, next) {
      next.whenData((state) {
        if (state.isAuthenticated) {
          // 登录成功，导航到主页
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
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            // 邮箱输入
            TextField(
              decoration: InputDecoration(
                labelText: 'Email',
                errorText: !formState.isEmailValid ? 'Invalid email' : null,
              ),
              keyboardType: TextInputType.emailAddress,
              onChanged: (value) {
                ref.read(loginFormProvider.notifier).updateEmail(value);
              },
            ),
            SizedBox(height: 16),
            
            // 密码输入
            TextField(
              decoration: InputDecoration(
                labelText: 'Password',
                errorText: !formState.isPasswordValid ? 'Password too short' : null,
              ),
              obscureText: true,
              onChanged: (value) {
                ref.read(loginFormProvider.notifier).updatePassword(value);
              },
            ),
            SizedBox(height: 24),
            
            // 登录按钮
            ElevatedButton(
              onPressed: formState.isValid && !formState.isSubmitting && !authState.isLoading
                  ? () {
                      ref.read(loginFormProvider.notifier).submit();
                    }
                  : null,
              child: authState.isLoading || formState.isSubmitting
                  ? CircularProgressIndicator()
                  : Text('Login'),
            ),
            SizedBox(height: 16),
            
            // 注册链接
            TextButton(
              onPressed: () {
                Navigator.push(
                  context,
                  MaterialPageRoute(builder: (_) => RegisterPage()),
                );
              },
              child: Text('Don\'t have an account? Register'),
            ),
            
            // 错误信息
            if (authState.hasError)
              Padding(
                padding: const EdgeInsets.only(top: 16),
                child: Text(
                  'Error: ${authState.error}',
                  style: TextStyle(color: Colors.red),
                ),
              ),
          ],
        ),
      ),
    );
  }
}
```

### 用户资料页面

```dart
class ProfilePage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authState = ref.watch(authProvider);
    
    return Scaffold(
      appBar: AppBar(
        title: Text('Profile'),
        actions: [
          IconButton(
            icon: Icon(Icons.logout),
            onPressed: () {
              ref.read(authProvider.notifier).logout();
            },
          ),
        ],
      ),
      body: authState.when(
        loading: () => Center(child: CircularProgressIndicator()),
        error: (error, stack) => Center(child: Text('Error: $error')),
        data: (state) {
          if (!state.isAuthenticated) {
            // 未认证，导航到登录页面
            WidgetsBinding.instance.addPostFrameCallback((_) {
              Navigator.pushReplacement(
                context,
                MaterialPageRoute(builder: (_) => LoginPage()),
              );
            });
            return Container();
          }
          
          final user = state.user!;
          
          return Padding(
            padding: const EdgeInsets.all(16.0),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.center,
              children: [
                // 头像
                CircleAvatar(
                  radius: 50,
                  backgroundImage: user.avatarUrl != null
                      ? NetworkImage(user.avatarUrl!)
                      : null,
                  child: user.avatarUrl == null
                      ? Text(user.name[0].toUpperCase())
                      : null,
                ),
                SizedBox(height: 16),
                
                // 用户名
                Text(
                  user.name,
                  style: TextStyle(
                    fontSize: 24,
                    fontWeight: FontWeight.bold,
                  ),
                ),
                SizedBox(height: 8),
                
                // 邮箱
                Text(
                  user.email,
                  style: TextStyle(
                    fontSize: 16,
                    color: Colors.grey,
                  ),
                ),
                SizedBox(height: 24),
                
                // 编辑资料按钮
                ElevatedButton(
                  onPressed: () {
                    // 导航到编辑资料页面
                  },
                  child: Text('Edit Profile'),
                ),
              ],
            ),
          );
        },
      ),
    );
  }
}
```

## 路由管理

为了根据认证状态控制路由，我们可以创建一个包装器组件：

```dart
class AuthWrapper extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final authState = ref.watch(authProvider);
    
    return authState.when(
      loading: () => Scaffold(
        body: Center(child: CircularProgressIndicator()),
      ),
      error: (error, stack) => Scaffold(
        body: Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Error: $error'),
              ElevatedButton(
                onPressed: () {
                  ref.refresh(authProvider);
                },
                child: Text('Retry'),
              ),
            ],
          ),
        ),
      ),
      data: (state) {
        if (state.isAuthenticated) {
          return HomePage();
        } else {
          return LoginPage();
        }
      },
    );
  }
}

// 主应用
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ProviderScope(
      child: MaterialApp(
        title: 'Login Demo',
        theme: ThemeData(
          primarySwatch: Colors.blue,
        ),
        home: AuthWrapper(),
      ),
    );
  }
}
```

## 状态管理的优势

通过这个案例，我们可以看到 Riverpod 在处理登录和 HTTP 请求方面的几个优势：

1. **表单状态管理**：使用 `Notifier` 可以轻松管理表单状态，包括验证和提交。

2. **HTTP 请求处理**：通过 `AsyncNotifier` 可以优雅地处理异步 HTTP 请求，包括加载状态和错误处理。

3. **令牌管理**：可以使用 Provider 依赖关系来管理认证令牌的存储、刷新和使用。

4. **拦截器**：可以使用 Dio 拦截器和 Riverpod 结合，实现自动令牌刷新和认证头添加。

5. **路由控制**：可以根据认证状态控制路由，确保用户只能访问他们有权限的页面。

## 小结

通过这个登录和 HTTP 请求的案例，我们展示了如何使用 Riverpod 来处理：

1. **表单状态**：使用 `Notifier` 管理表单输入和验证
2. **HTTP 请求**：使用 `AsyncNotifier` 处理 API 调用和响应
3. **会话管理**：存储、刷新和使用认证令牌

这个案例展示了 Riverpod 在处理用户认证和 API 交互方面的强大功能和灵活性，以及如何组织代码以提高可维护性和可测试性。

在下一篇文章中，我们将探讨 Riverpod 的性能调优和最佳实践，包括 `select` 的使用、状态结构化和常见陷阱的避免。
