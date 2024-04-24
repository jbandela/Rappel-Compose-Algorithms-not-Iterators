# Rappel
### Compose Algorithms, not Iterators
#### Google's Alternative to Ranges

Authors: Chris Philip and John Bandela  
Presented by: John Bandela at CppNow 2024

---
> __Iterators are the wrong abstraction for composing algorithms.__
---

---
## History
![Programming Pearls](knuth_mcilroy.png)

1986
---
### Challenge
> Given a text file and an integer k, print the k most
common words in the file (and the number of
their occurrences) in decreasing frequency.
--

### Don Knuth

Produces an absolute virtuoso work of art using literate programming and WEB.
* Elegant Pascal code
* Tries
* Beautifully typeset with cross-references

--

### Doug McIlroy 

```
tr -cs A-Za-z`
` |
tr A-Z a-z |
sort |
uniq -c |
sort -rn |
sed ${1} q 

```
--
## Pipeline Style
* By pipeline style, we mean writing a series of transformations as a single expression without nested parentheses, rather than the more standard style of using separate invocations. 
```c++
// Standard C++, non-pipelined
auto result1 = f1(input);
auto result2 = f2(result1);
auto result3 = f3(result2);

// Pipeline style
auto result = input | f1 | f2 | f3;

```
--
### std::ranges
```c++
auto even = [](int i) { return 0 == i % 2; };
auto square = [](int i) { return i * i; };
auto filter_squared = std::view::iota(i) 
  | std::views::filter(even) | std::views::transform(square);
for (int i : filtered_squared)
  std::cout << i << ' ';
```
* https://en.cppreference.com/w/cpp/ranges#Range_adaptors
--

### Find numbers whose square is greater than the hash of the string
```c++
 using IntAndString = std::pair<int, std::string>;
 auto make_int_and_string = [](int i) -> IntAndString {
     return {i*i, std::to_string(i)};
 };
 auto result = std::views::iota(1,1000001)                           //
   | std::views::transform(make_int_and_string)  
   | std::views::filter([](const auto& p) {
        return p.first >= std::hash<std::string>()(p.second);
    })
   | std::views::transform(&IntAndString::second)  
   | std::views::take(4);
 for (auto s : result) 
   std::cout << s << "\n";
```



--

# Crash

--
#### Why?
```c++[|6,10]
 using IntAndString = std::pair<int, std::string>;
 auto make_int_and_string = [](int i) -> IntAndString {
     return {i*i, std::to_string(i)};
 };
 auto result = std::views::iota(1,1000001)                           //
   | std::views::transform(make_int_and_string)  
   | std::views::filter([](const auto& p) {
        return p.first >= std::hash<std::string>()(p.second);
    })
   | std::views::transform(&IntAndString::second)  
   | std::views::take(4);
 for (auto s : result) 
   std::cout << s << "\n";
```
--
#### Reduced
```c++[]
 using IntAndString = std::pair<int, std::string>;
 auto make_int_and_string = [](int i) -> IntAndString {
     return {i*i, std::to_string(i)};
 };
 auto result = std::views::iota(1,1000001)                           //
   | std::views::transform(make_int_and_string)  
   | std::views::transform(&IntAndString::second)  
 for (auto s : result) 
   std::cout << s << "\n";
```


--
#### Transform Iterators

```c++[|10-11]
struct MakeIntAndStringIterator{
  IotaIterator iota_iter;
  IntAndString operator*(){
    return make_int_and_string(*iota_iter);
  }
};

struct SecondIterator{
  MakeIntAndStringIterator int_and_string_iter;
  std::string& operator*(){
    return (*int_and_string_iter).second;
  }
}


```

--
### Solution
* Range V3 `views::cache1`
* This in not in C++ ranges

--
### TPOIASI
```c++
int times2(int n){
   return n * 2;
}
bool isMultipleOf4(int n){return n % 4 == 0;}

int main() {
 std::vector<int> numbers = {1, 2, 3, 4, 5};
 auto results = numbers 
  | std::ranges::views::transform(times2) 
  | std::ranges::views::filter(isMultipleOf4);
 for (auto result : results) 
     std::cout << result << '\n';
}

```
* https://www.fluentcpp.com/2019/02/12/the-terrible-problem-of-incrementing-a-smart-iterator/
--
#### Output
```
4
8
```
--
### TPOIASI
```c++
int times2(int n){
   std::cout << "transforming " << n << "\n";
   return n * 2;
}
bool isMultipleOf4(int n){return n % 4 == 0;}

int main() {
 std::vector<int> numbers = {1, 2, 3, 4, 5};
 auto results = numbers 
  | std::ranges::views::transform(times2) 
  | std::ranges::views::filter(isMultipleOf4);
 for (auto result : results) 
     std::cout << result << '\n';
}

```
* https://www.fluentcpp.com/2019/02/12/the-terrible-problem-of-incrementing-a-smart-iterator/
--
#### Output
```
transforming 1
transforming 2
transforming 2
4
transforming 3
transforming 4
transforming 4
8
transforming 5

```
--
#### Stack size
```c++
__attribute__((noinline)) void Main7() {
  auto even = [](auto i) { return i % 2 == 0; };
  std::vector<int> r0 = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
  auto result = r0 | std::views::filter(even) //
                | std::views::filter(even)    //
                | std::views::filter(even)    //
                | std::views::filter(even)    //
                | std::views::filter(even)    //
                | std::views::filter(even)    //
                | std::views::filter(even);
  PrintStack("Main7");
}

```
* https://godbolt.org/z/6zhGaf588 
--

#### Stack size
```c++

int main() {
  Main0();
  Main1();
  Main2();
  Main3();
  Main4();
  Main5();
  Main6();
  Main7();
}
```

--
#### Output
```
Main0 192 144 32 0 
Main1 192 192 32 0 
Main2 192 240 32 0 
Main3 192 320 32 0 
Main4 192 448 32 0 
Main5 192 608 32 0 
Main6 192 832 32 0 
Main7 192 1120 32 0 


```
--
#### Cubic

* Richard Smith
* 144 + ((N+1)(N+2)(N+3)/6 + 2N) * 8 bytes 

Note:
(Except Main0 and Main4 which is off by 8)
We get one factor of N for:

* Making each range hold the previous one by value instead of by reference 
* Creating each range object in the same full expression 
* Having each range cache its begin() so that has an amortized constant-time begin() as (I think is) required by the view requirements 

--
#### Can we make it better
> Fundamentally, I don't think this cubic behavior can be fixed without giving up something that Ranges cares about. We get one factor of N for each of:

- Making each range hold the previous one by value instead of by reference (which seems necessary for safety / correctness in this example)
- Creating each range object in the same full expression (which seems important for the library's desired ergonomics)
- Having each range cache its begin() so that has an amortized constant-time begin() as (I think is) required by the view requirements (which seems important for performance in some cases, due to the use of iterators underlying the ranges library)
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
```c++[|2,14|3|4|5|6|7|8|9|10|11-13]
void OutputPythagoreanTriples() {
 rpl::Apply(
  rpl::Iota(1),
  rpl::ZipResult([](int c) {return rpl::Iota(1, c+1);}),
  rpl::Flatten(),
  rpl::ZipResult([](int c, int a){return rpl::Iota(a, c+1);}),
  rpl::Flatten(),
  rpl::Filter([](int c, int a, int b){return a*a + b*b == c*c;}),
  rpl::Swizzle<1, 2, 0>(),
  rpl::Take(10),
  rpl::ForEach([](int a, int b, int c){
   std::cout << a << " " << b << " " << c << "\n";
  })
 );
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
* Filter - Only passes to next stage iff predicate is true
* Swizzle - Rearranges the stream arguments
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
  rpl::Filter([](int c, int a, int b){return a*a + b*b == c*c;}),
  rpl::Swizzle<1, 2, 0>(),
  rpl::Take(10),
  rpl::ForEach([](int a, int b, int c){
   std::cout << a << " " << b << " " << c << "\n";
  })
 );
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
  rpl::Filter([](int c, int a, int b){return a*a + b*b == c*c;}),
  rpl::Swizzle<1, 2, 0>(),
  rpl::Take(10),
  rpl::ForEach([](int a, int b, int c){
   std::cout << a << " " << b << " " << c << "\n";
  })
 );
}

```
--
<!-- .slide: data-transition="slide-in none-out" -->
### Flexibility 
```c++[|10-13||1]
void OutputPythagoreanTriples() {
 auto zip_flat = [](auto f){
  return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
 rpl::Apply(
  rpl::Iota(1),
  zip_flat([](int c) {return rpl::Iota(1, c+1);}),
  zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
  rpl::Filter([](int c, int a, int b){return a*a + b*b == c*c;}),
  rpl::Swizzle<1, 2, 0>(),
  rpl::Take(10),
  rpl::ForEach([](int a, int b, int c){
      std::cout << a << " " << b << " " << c << "\n";
  })
 );
}

```
--
<!-- .slide: data-transition="none-out none-in" -->
### Flexibility 
```c++[1||4]
auto PythagoreanTriples() {
 auto zip_flat = [](auto f){
  return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
 rpl::Apply(
  rpl::Iota(1),
  zip_flat([](int c) {return rpl::Iota(1, c+1);}),
  zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
  rpl::Filter([](int c, int a, int b){return a*a + b*b == c*c;}),
  rpl::Swizzle<1, 2, 0>(),
  rpl::Take(10),
  rpl::ForEach([](int a, int b, int c){
      std::cout << a << " " << b << " " << c << "\n";
  })
 );
}

```
--
<!-- .slide: data-transition="none-out none-in" -->
### Flexibility 
```c++[4||10-13]
auto PythagoreanTriples() {
 auto zip_flat = [](auto f){
  return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
 return rpl::Compose(
  rpl::Iota(1),
  zip_flat([](int c) {return rpl::Iota(1, c+1);}),
  zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
  rpl::Filter([](int c, int a, int b){return a*a + b*b == c*c;}),
  rpl::Swizzle<1, 2, 0>(),
  rpl::Take(10),
  rpl::ForEach([](int a, int b, int c){
      std::cout << a << " " << b << " " << c << "\n";
  })
 );
}

```
--
<!-- .slide: data-transition=" none-in" -->
### Flexibility 
```c++[]
auto PythagoreanTriples() {
 auto zip_flat = [](auto f){
  return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
 return rpl::Compose(
  rpl::Iota(1),
  zip_flat([](int c) {return rpl::Iota(1, c+1);}),
  zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
  rpl::Filter([](int c, int a, int b){return a*a + b*b == c*c;}),
  rpl::Swizzle<1, 2, 0>()




 );
}

```
--
### Pythagorean Triples 
```c++[|6|7|8|9|10]
auto PythagoreanTriples() {
 auto zip_flat = [](auto f){
  return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};

 return rpl::Compose(
  rpl::Iota(1),
  zip_flat([](int c) {return rpl::Iota(1, c+1);}),
  zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
  rpl::Filter([](int c, int a, int b){return a*a + b*b == c*c;}),
  rpl::Swizzle<1, 2, 0>()
 );
}

```
Note:
* Hypotenuse length
* Long leg length
* Short leg length
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



 