---
# Extra Slides

---
## Error Handling

--

```c++
// Wrap different types in a `std::optional` if necessary.
struct AsOptional {
  std::nullopt_t operator()(std::nullopt_t) const { return std::nullopt; }
  template <typename T>
  std::optional<T> operator()(std::optional<T> opt) const {
    return opt;
  }
  template <typename T>
  std::optional<T> operator()(T t) const {
    return std::make_optional(std::move(t));
  }
};

```
--

```c++
struct UnwrapOptionalImpl {
  bool has_error = false;
  template <typename T>
  void Process(const std::optional<T>& t) {
    has_error = !t.has_value();
  }

  bool HasError() const { return has_error; }

  template <typename F>
  auto ToResult(F&& f) {
    auto as_optional = [&] { return AsOptional()(f()); };
    if (has_error) {
      return decltype(as_optional())(std::nullopt);
    } else {
      return as_optional();
    }
  }
};
```
--

```c++
template <typename InputType, typename ErrorImpl>
struct BreakIfErrorImpl {
  using OutputType = InputType;
  ErrorImpl error_impl{};

  template <typename Next>
  void ProcessIncremental(InputType input, Next& next) {
    error_impl.Process(input);
    if (!error_impl.HasError()) {
      next.ProcessIncremental(static_cast<InputType>(input));
    }
  }

  template <typename Next>
  auto End(Next& next) {
      return error_impl.ToResult([&] { return next.End(); });
  }

  bool Done() const { return error_impl.HasError(); }
};



```
--
```c++
inline auto UnwrapOptional() {
  return rpl::Compose(
      BreakIfError<UnwrapOptionalImpl>(), 
      Deref());
}


```