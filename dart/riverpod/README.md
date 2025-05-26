# Riverpod 源码分析系列

本系列文章深入分析 Riverpod 的源码实现、工作原理和最佳实践。通过阅读这些文章，你将全面理解 Riverpod 的内部机制，并能够更有效地使用它来管理 Flutter 应用的状态。

## 文章列表

1. [核心概念与基本原理](01_core_concepts_and_fundamentals.md)
   - Provider、Ref 和 ProviderContainer 的实现原理
   - 状态管理的基本流程
   - 依赖注入机制

2. [异步和简单状态提供者](02_async_and_simple_state_providers.md)
   - AsyncValue 的实现
   - FutureProvider 和 StreamProvider 的源码分析
   - StateProvider 的使用场景和实现

3. [复杂状态提供者](03_complex_state_providers.md)
   - NotifierProvider 系列分析
   - Notifier、AsyncNotifier 和 StreamNotifier 的实现
   - 状态更新和通知机制

4. [修饰符](04_modifiers.md)
   - autoDispose 修饰符的实现
   - family 修饰符的源码分析
   - 自定义修饰符的方法

5. [高级交互与测试](05_advanced_interaction_and_testing.md)
   - invalidate 和 refresh 方法分析
   - overrides 机制
   - 测试 Provider 的最佳实践

6. [实践案例研究 1 - 电子商务应用](06_practical_case_study_1_ecommerce.md)
   - 产品列表管理
   - 购物车状态管理
   - 用户认证流程

7. [实践案例研究 2 - 登录与 HTTP 请求](07_practical_case_study_2_login_and_http.md)
   - 表单状态管理
   - HTTP 请求处理
   - 会话管理和令牌刷新

8. [性能调优与最佳实践](08_performance_tuning_and_best_practices.md)
   - 使用 select 精确订阅
   - 状态结构化方法
   - 常见陷阱和解决方案

9. [注解与代码生成](09_riverpod_annotations_and_code_generation.md)
   - @riverpod 注解的实现
   - 代码生成原理
   - 注解系统的最佳实践

## 如何使用本系列文章

1. 按顺序阅读以获得对 Riverpod 的全面理解
2. 参考特定主题直接跳转到相关章节
3. 结合源码阅读以获得更深入的理解

## 运行示例代码

本系列文章中的代码示例可以直接在支持 Riverpod 的 Flutter 项目中运行。确保在 `pubspec.yaml` 中添加了最新版本的 Riverpod 依赖。

```yaml
dependencies:
  flutter_riverpod: ^2.0.0
  riverpod_annotation: ^2.0.0

dev_dependencies:
  build_runner: ^2.0.0
  riverpod_generator: ^2.0.0
  riverpod_lint: ^2.0.0
```

## 贡献

如果你发现任何错误或有改进建议，欢迎提交 issue 或 pull request。

## 许可证

本系列文章采用 [MIT 许可证](LICENSE)。

---

Happy coding with Riverpod! 🚀
