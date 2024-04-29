## Using Rappel
---

### Composition
---
`Compose` is a first-class stage
```c++ [|2|7]
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
```c++ [|2-4|8]
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
```c++ [|5-7]
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
### Generators
* Input iterators can be difficult to write
* Want an easy way to make generators such as `Iota`

```c++
template <typename T, typename GeneratorInvocable>
auto Generator(GeneratorInvocable&& generator_invocable);
```
---
### Iota
```c++[|4]
template <typename T>
auto Iota(T begin) {
  return rpl::Generator<T>([begin](auto output) mutable {
    std::move(output)(begin);
    ++begin;
  });
};
---
#### Iota with end
```c++[|4]
template <typename T, End>
auto Iota(T begin, End end) {
  return rpl::Generator<T>([begin,end](auto output) mutable {
    if(begin != end){
        std::move(output)(begin);
        ++begin;
    }
  });
};

```
---

### Multi-Argument Streams
---
Multi-argument streams are passed as seperate arguments
```c++ [|2|3|4|5]
std::map<int, std::vector<int>> values = {...};
rpl::Apply(values,              // pair<int, vector>
           rpl::ExpandTuple(),  // int, vector
           rpl::Flatten(),      // int, int
           rpl::ForEach([](int k, int v) { ... }));
```
---
First class support
```c++ [|5-6]
std::map<std::string, int> name2id = {...};
auto id2name = rpl::Apply(
  std::move(name2id),
  rpl::ExpandTuple(),
  rpl::Swizzle<1, 0>(),
  rpl:ToMap<std::map>());
```
---
TransformArg
```c++ [|5-6]
std::map<std::string, Person> name2person = {...};
auto id2name = rpl::Apply(
  std::move(name2id),
  rpl::ExpandTuple(),
  rpl::TransformArg<1>(&Person::id),
  rpl::Swizzle<1, 0>(),
  rpl:ToMap<std::map>());
```
---
Preserve Reference Semantics
```c++ [|9|13]
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
Note:
We can use temporaries safely because everything contained inside `Apply`
---
### Error Handling
---
Short-circuit on error and wrap results
```c++ [|6|7|8|9|4,10]
std::optional<int> ParseInt(std::string_view);

std::vector<std::optional<std::string>> values = {...};
std::optional<int> sum = rpl::Apply(
  values,                     // optional<string>
  rpl::UnwrapOptional(),      // string
  rpl::Transform(&ParseInt),  // optional<int>
  rpl::UnwrapOptional(),      // int
  rpl::Accumulate()           // int
);                            // optional<int>
```
---
Monadic style
```c++ [|6-8|10-14]
std::optional<int> ParseInt(std::string_view);
int Squared(int i);

std::optional<std::string> value = ...;

std::optional<int> cpp23 = value
  .and_then(&ParseInt)
  .transform(&Squared);

std::optional<int> result = rpl::Apply(
  value,
  rpl::UnwrapOptionalComplete(),
  rpl::TransformComplete(&ParseInt),
  rpl::TransformComplete(&Squared));
```
---
Monadic StatusOr
```c++ [|7|8|9|10|11|12|6,13]
absl::StatusOr<int> ParseInt(std::string_view);
int Squared(int i);

absl::StatusOr<std::vector<std::string>> values = ...;

absl::StatusOr<std::vector<int>> result = rpl::Apply(
  values,                        // StatusOr<vector<string>>
  rpl::UnwrapStatusOrComplete(), // std::vector
  rpl::Transform(&ParseInt),     // StatusOr<int>
  rpl::UnwrapStatusOr(),         // int
  rpl::Transform(&Squared),      // int
  rpl::To<std::vector>()         // vector<int>
);                               // StatusOr<vector<int>>
```
---
### Higher-Order Stages
---
`Tee` splits outputs to multiple stages
```c++ [|2,5-6]
std::vector<int> values = {...};
auto [min, max, count] = rpl::Apply(
  values,
  rpl::Tee(
    rpl::Min(), rpl::Max(), rpl::Count()));
```
---
Group elements
```c++ [|9-12]
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
```c++[|3|8]
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
```c++[|2|7]
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
```c++[|2|7]
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
### Conveniences
---
Initializing `std::vector` with `unique_ptr`

```c++
// Doesn't work
std::vector v{make_unique<int>(0), make_unique<int>(1)}; 

// Works
auto v = rpl::Apply(
  {std::make_unique<int>(0), std::make_unique<int>(1)},
  rpl::To<std::vector>());

```
---
Moving keys out of map or set
```c++

// Doesn't work
std::vector<std::string> v(std::move(s).begin(),
  std::move(s).end());

// Doesn't work
std::vector<std::string> v2(std::make_move_iterator(s.begin()),
  std::make_move_iterator(m.end()));

// Works
auto v = rpl::Apply(std::move(s), rpl::To<std::vector>());

```
Note:
Uses extract (or extract_and_get_next)