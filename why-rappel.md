## Why Rappel?


### Composition


`Compose` is just a regular stage
```c++ [2|7]
auto FlattenOptional() {
  return rpl::Compose(rpl::Filter(), rpl::Deref());
}

rpl::Apply(
  std::vector<std::optional<int>>{1, std::nullopt, 3},
  FlattenOptional(),
  rpl::To<std::vector>());
```


`Compose` generators
```c++ [2-4|8]
auto PerfectSquares() {
  return rpl::Compose(
    rpl::Iota(1),
    rpl::Transform([](int i) { return i*i; }));
}

rpl::Apply(
  PerfectSquares(),
  rpl::Filter([](int i) { return i >= 100; }),
  rpl::Take(5),
  rpl::To<std::vector>());
```


`Compose` ranges
```c++ [5-7]
struct Transactions {
  std::vector<double> amounts;

  auto Credits() {
    return rpl::Compose(
      std::cref(amounts),
      rpl::Filter([](double d) { return d > 0; }));
  }
};
```


### Multi-Value Streams


Multi-value streams are passed as seperate arguments
```c++ [2|3|4|5]
std::map<int, std::vector<int>> values = {...};
rpl::Apply(values,              // pair<int, vector>
           rpl::ExpandTuple(),  // int, vector
           rpl::Flatten(),      // int, int
           rpl::ForEach([](int k, int v) { ... }));
```


First class support
```c++ [5-6]
std::map<std::string, int> name2age = {...};
auto age2name = rpl::Apply(
  std::move(name2age),
  rpl::ExpandTuple(),
  rpl::Swizzle<1, 0>(),
  rpl:ToMap<std::map>());
```


"Correct" Reference Semantics
```c++ [4-5|6-8]
std::vector<std::unique_ptr<int>> values = {...};
rpl::Apply(
  values,
  rpl::ZipResult([](const auto& ptr) {
    return std::make_unique<int>(*ptr); }),
  rpl::ForEach([](
    std::same_as<const std::unique_ptr<int>&> auto&& first,
    std::same_as<std::unique_ptr<int>> auto&& second) {}));
```


### Error Handling


Short-circuit on error and wrap results
```c++ [6|7|8|9|4-9]
std::optional<int> ParseInt(std::string_view);

std::vector<std::optional<std::string>> values = {...};
std::optional<int> sum = rpl::Apply(
  values,                     // optional<string>
  rpl::UnwrapOptional(),      // string
  rpl::Transform(&ParseInt),  // optional<int>
  rpl::UnwrapOptional(),      // int
  rpl::Accumulate());         // int
```


Monadic style
```c++ [6-8|10-14]
std::optional<int> ParseInt(std::string_view);
int Squared(int i);

std::optional<std::string> value = ...;

std::optional<int> cpp23 = value
  .and_then(&ParseInt)
  .transform(&Squared);

std::optional<int> rappel = rpl::Apply(
  value,
  rpl::UnwrapOptionalComplete(),
  rpl::TransformComplete(&ParseInt),
  rpl::TransformComplete(&Squared));
```


### Higher-Order Stages


`Tee` splits outputs to multiple stages
```c++ [8|9-12]
auto compute_avg = [](int sum, int count) {
  return sum / count;
};
std::vector<int> values = {...};
auto [min, max, count, avg] = rpl::Apply(
  values,
  rpl::Tee(
    rpl::Min(), rpl::Max(), rpl::Count(),
    rpl::Compose(
      rpl::Tee(rpl::Accumulate(), rpl::Count()),
      rpl::TransformComplete(rpl::ExpandTuple()),
      rpl::TransformComplete(compute_avg))));
```


Group elements
```c++ [4-6]
std::vector<int> values = {1, 2, 11, 12, 21, 22};
auto groups = rpl::Apply(
  values,
  rpl::MapGroupBy<std::map>(
    [](int i) { return i / 10; },
    rpl::To<std::vector>()),
  rpl::ToMap<std::map>());
// groups = {0: {1, 2},
//           1: {11, 12},
//           2: {21, 22}};
```


Reuse Transforms
```c++ [3]
int i = rpl::Apply(
  std::make_unique<int>(1),
  rpl::TransformComplete(rpl::Deref()));
```


### Reference Inputs


Const reference iteration by default
```c++
std::vector<std::unique_ptr<int>> values = {...};
rpl::Apply(
  values,  // values not copied!
  rpl::ForEach(
    [](std::same_as<const std::unique_ptr<int>&> auto&&) {
  }));
```


Mutable references with `std::ref`
```c++
rpl::Apply(
  std::ref(values),
  rpl::ForEach(
    [](std::same_as<std::unique_ptr<int>&> auto&&) {
  }));
```


R-Values when moved
```c++
rpl::Apply(
  std::move(values),
  rpl::ForEach(
    [](std::same_as<std::unique_ptr<int>> auto&&) {
  }));
```


Lazily moved
```c++ [8-13]
auto values = rpl::Apply(
  {1,2,3,4},
  rpl::Transform([](int i) {
    return std::make_unique<int>(i); }),
  rpl::To<std::vector>());

auto evens = rpl::Apply(
  std::move(values),
  rpl::Filter([](const auto& ptr) {
    return *ptr % 2 == 0; }),
  rpl::To<std::vector>());
// values = {*1, nullptr, *3, nullptr}
// evens = {*2, *4}
```
