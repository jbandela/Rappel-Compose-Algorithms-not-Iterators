## Using Rappel
---
### Aim for default safety
* By design, less likely to have dangling references
* `Apply` returns by value
* Stages have safe defaults
  * `Max` return `optional<T>`
  * `PartialSort(n)` is safe even if `n` > number of elements
  * `NthElement(n)` is safe even if `n` > number of elements
---
### Composition
---
`Compose` is a first-class stage
```c++ [2|7]
auto FlattenOptional() {
  return rpl::Compose(rpl::Filter(), rpl::Deref());
}

rpl::Apply(
  std::vector<std::optional<int>>{1, std::nullopt, 3},
  FlattenOptional(),
  rpl::To<std::vector>());
```
---
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
---
`Compose` ranges
```c++ [5-7]
struct Transactions {
  std::vector<double> amounts;

  auto Credits() const {
    return rpl::Compose(
      std::cref(amounts),
      rpl::Filter([](double d) { return d > 0; }));
  }
};
```
---
### Multi-Argument Streams
---
Multi-argument streams are passed as seperate arguments
```c++ [2|3|4|5]
std::map<int, std::vector<int>> values = {...};
rpl::Apply(values,              // pair<int, vector>
           rpl::ExpandTuple(),  // int, vector
           rpl::Flatten(),      // int, int
           rpl::ForEach([](int k, int v) { ... }));
```
---
First class support
```c++ [5-6]
std::map<std::string, int> name2id = {...};
auto id2name = rpl::Apply(
  std::move(name2id),
  rpl::ExpandTuple(),
  rpl::Swizzle<1, 0>(),
  rpl:ToMap<std::map>());
```
---
TransformArg
```c++ [5-6]
std::map<std::string, Person> name2person = {...};
auto id2name = rpl::Apply(
  std::move(name2id),
  rpl::ExpandTuple(),
  rpl::TransformArg<1>(&Person::id),
  rpl::Swizzle<1, 0>(),
  rpl:ToMap<std::map>());
```
---
"Correct" Reference Semantics
```c++ [4-5|6-8]
std::vector<std::unique_ptr<int>> values = {...};
rpl::Apply(
  values,
  rpl::ZipResult([](const auto& ptr) {
    return std::make_unique<int>(*ptr); }),
  rpl::ForEach([](auto&& first, auto&& second) {
      static_assert(std::is_same_v<
        decltype(first),
        const std::unique_ptr<int>&
      >);
      static_assert(std::is_same_v<
        decltype(second),
        std::unique_ptr<int>&&
      >);
    }));
```
---
### Error Handling
---
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
---
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
---
Monadic StatusOr
```c++ [6-8|10-14]
absl::StatusOr<int> ParseInt(std::string_view);
int Squared(int i);

absl::StatusOr<std::string> value = ...;

absl::StatusOr<int> rappel = rpl::Apply(
  value,
  rpl::UnwrapStatusOrComplete(),
  rpl::TransformComplete(&ParseInt),
  rpl::TransformComplete(&Squared));
```
---
### Higher-Order Stages
---
`Tee` splits outputs to multiple stages
```c++ [10|11-14]
auto compute_avg = [](int sum, int count) {
  return count > 0
    ? std::make_optional(sum / count)
    : std::nullopt;
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
---
Group elements
```c++ [9-12]
struct Employee {
  int id;
  bool is_fulltime;
  std::string org;
};
std::vector<Employee> employees = {...};
auto counts = rpl::Apply(
  employees,
  rpl::MapGroupBy<std::unordered_map>(
    &Employee::org
    rpl::Filter(&Employee::is_fulltime),
    rpl::Count()),
  rpl::MakePair(),
  rpl::To<std::vector>());
```
---
### Reference Inputs
---
Const reference iteration by default
```c++
std::vector<std::unique_ptr<int>> values = {...};
rpl::Apply(
  values,  // values not copied!
  rpl::ForEach(
    [](auto&& ptr) {
    static_assert(std::is_same_v<
      decltype(ptr),
      const std::unique_ptr<int>&
    >);
  }));
```
---
Mutable references with `std::ref`
```c++
rpl::Apply(
  std::ref(values),
  rpl::ForEach(
    [](auto&& ptr) {
      static_assert(std::is_same_v<
        decltype(ptr),
        std::unique_ptr<int>&
      >);
  }));
```
---
R-Values when moved
```c++
rpl::Apply(
  std::move(values),
  rpl::ForEach(
    [](auto&& ptr) {
      static_assert(std::is_same_v<
        decltype(first),
        std::unique_ptr<int>&&
      >);
  }));
```
---
Lazily moved
```c++ [8-13]
auto values = rpl::Apply(
  {std::make_unique<int>(1), std::make_unique<int>(2),
   std::make_unique<int>(3), std::make_unique<int>(4)},
  rpl::To<std::vector>());

auto evens = rpl::Apply(
  std::move(values),
  rpl::Filter([](const auto& ptr) {
    return *ptr % 2 == 0; }),
  rpl::To<std::vector>());
// values = {*1, nullptr, *3, nullptr}
// evens = {*2, *4}
```
---
### Conveniences
---
Initializing `std::vector` with `unique_ptr`

```c++
std::vector v{make_unique<int>(0), make_unique<int>(1)}; 
```

```c++
auto v = rpl::Apply(
  {std::make_unique<int>(0), std::make_unique<int>(1)},
  rpl::To<std::vector>());

```
---
Moving keys out of map or set
```c++
std::vector<std::string> v(std::move(s).begin(),
  std::move(s).end());

std::vector<std::string> v2(std::move_move_iterator(s.begin()),
  std::make_move_iterator(m.end()));

```

```c++
auto v = rpl::Apply(std::move(s), rpl::To<std::vector>());

```