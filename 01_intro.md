# Rappel
### Compose Algorithms, not Iterators
#### Google's Alternative to Ranges

Authors: Chris Philip and John Bandela  
Presented by: John Bandela at CppNow 2024

---
## A C++ Tradition  
* In December 2018 after ranges were merged into C++20 Eric Niebler wrote a blog post showing how ranges can implement Pythagorean triples
  * https://ericniebler.com/2018/12/05/standard-ranges/
* We will honor that tradition today
* Pythagorean Triple: `a*a + b*b == c*c`

--
### Helper range
```c++
template<Semiregular T>
struct maybe_view : view_interface<maybe_view<T>> {
  maybe_view() = default;
  maybe_view(T t) : data_(std::move(t)) {
  }
  T const *begin() const noexcept {
    return data_ ? &*data_ : nullptr;
  }
  T const *end() const noexcept {
    return data_ ? &*data_ + 1 : nullptr;
  }
private:
  optional<T> data_{};
};
```
--
### Helper lambdas
```c++
inline constexpr auto for_each =
  []<Range R,
     Iterator I = iterator_t<R>,
     IndirectUnaryInvocable<I> Fun>(R&& r, Fun fun)
        requires Range<indirect_result_t<Fun, I>> {
      return std::forward<R>(r)
        | view::transform(std::move(fun))
        | view::join;
  };

inline constexpr auto yield_if =
  []<Semiregular T>(bool b, T x) {
    return b ? maybe_view{std::move(x)}
             : maybe_view<T>{};
  };
```
--

### Lazy Triples
```c++
using view::iota;
auto triples =
  for_each(iota(1), [](int z) {
    return for_each(iota(1, z+1), [=](int x) {
      return for_each(iota(x, z+1), [=](int y) {
        return yield_if(x*x + y*y == z*z,
          make_tuple(x, y, z));
      });
    });
  });
```
--
### Output
```c++
for(auto triple : triples | view::take(10)) {
  cout << '('
       << get<0>(triple) << ','
       << get<1>(triple) << ','
       << get<2>(triple) << ')' << '\n';
}
```
--
## Rappel
* Google's alterative to std::ranges for algorithm composition
* Makes a different set of tradeoffs than std::ranges
* Let's continue the tradition
--
### Pythagorean Triples 
```c++[|2,14|3|4|5|6|7|8|9-10|11|12-13]
void OutputPythagoreanTriples() {
  rpl::Apply(
   rpl::Iota(1),
   rpl::ZipResult([](int c) {return rpl::Iota(1, c+1);}),
   rpl::Flatten(),
   rpl::ZipResult([](int c, int a){return rpl::Iota(a, c+1);}),
   rpl::Flatten(),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}),
   rpl::Take(10),
   rpl::ForEach([](int a, int b, int c){
       std::cout << a << " " << b << " " << c << "\n";
   }));
}
```
Note:
* Apply 
  * Input
  * One or more *stage*
  * Input + Stages = Pipeline
  * Eager execution
* Iota - Generator
* ZipResult - Calls a function object with the stream values, and zips it
* Flatten - Flattens the last stream argument - outputting the previous args and every sub-element of the last arg
* Swizzle - Rearranges the stream arguments
* Filter - Only passes to next stage iff predicate is true
* Take
* ForEach - Notice now we get multiple arguments
--
### Repetition 
```c++ [|4-5|6-7]
void OutputPythagoreanTriples() {
  rpl::Apply(
   rpl::Iota(1),
   rpl::ZipResult([](int c) {return rpl::Iota(1, c+1);}),
   rpl::Flatten(),
   rpl::ZipResult([](int c, int a) {return rpl::Iota(a, c+1);}),
   rpl::Flatten(),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}),
   rpl::Take(10),
   rpl::ForEach([](int a, int b, int c){
       std::cout << a << " " << b << " " << c << "\n";
   }));
}

```
--
### Compose 
```c++[2-3|6-7|]
void OutputPythagoreanTriples() {
  auto zip_flat = [](auto f){
   return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
  rpl::Apply(
   rpl::Iota(1),
   zip_flat([](int c) {return rpl::Iota(1, c+1);}),
   zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}),
   rpl::Take(10),
   rpl::ForEach([](int a, int b, int c){
       std::cout << a << " " << b << " " << c << "\n";
   }));
}

```
--
### Flexibility 
```c++[|11-13]
void OutputPythagoreanTriples() {
  auto zip_flat = [](auto f){
   return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
  rpl::Apply(
   rpl::Iota(1),
   zip_flat([](int c) {return rpl::Iota(1, c+1);}),
   zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}),
   rpl::Take(10),
   rpl::ForEach([](int a, int b, int c){
       std::cout << a << " " << b << " " << c << "\n";
   }));
}

```
--

### Pythagorean Triples 
```c++[1|4||5|6|7|8|9-10]
auto PythagoreanTriples() {
  auto zip_flat = [](auto f){
   return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
  return rpl::Compose(
   rpl::Iota(1),
   zip_flat([](int c) {return rpl::Iota(1, c+1);}),
   zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}));
}

```

--

### Pythagorean Triples 
```c++[|2,7|3|4|5|6]
vector<tuple<int,int,int>> triples = 
  rpl::Apply(
    PythagoreanTriples(),
    rpl::Take(100),
    rpl::MakeTuple(), 
    rpl::To<std::vector>()
  );

```



 