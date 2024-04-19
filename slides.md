# Rappel
### Compose Algorithms, not Iterators
#### Google's Alternative to Ranges

Authors: Chris Philip and John Bandela  
Presented by: John Bandela at CppNow 2024

---

# Pythagorean Triples 
```c++[|2,14|3|4|5|6|7|8|9-10|11|12-13]
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
   rpl::ForEach([](auto a, auto b, auto c){
       std::cout << a << " " << b << " " << c << "\n";
   }));
}

```
--
# Repetition 
```c++ [|4-5|6-7]
void OutputPythagoreanTriples() {
  rpl::Apply(
   rpl::Iota(1),
   rpl::ZipResult([](auto c) {return rpl::Iota(1, c+1);}),
   rpl::Flatten(),
   rpl::ZipResult([](int c, int a) {return rpl::Iota(a, c+1);}),
   rpl::Flatten(),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}),
   rpl::Take(10),
   rpl::ForEach([](auto a, auto b, auto c){
       std::cout << a << " " << b << " " << c << "\n";
   }));
}

```
--
# Compose 
```c++[2-3|6-7|]
void OutputPythagoreanTriples() {
  auto zip_flat = [](auto f){
   return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
  rpl::Apply(
   rpl::Iota(1),
   zip_flat([](auto c) {return rpl::Iota(1, c+1);}),
   zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}),
   rpl::Take(10),
   rpl::ForEach([](auto a, auto b, auto c){
       std::cout << a << " " << b << " " << c << "\n";
   }));
}

```
--
# Flexibility 
```c++[|11-13]
void OutputPythagoreanTriples() {
  auto zip_flat = [](auto f){
   return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
  rpl::Apply(
   rpl::Iota(1),
   zip_flat([](auto c) {return rpl::Iota(1, c+1);}),
   zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}),
   rpl::Take(10),
   rpl::ForEach([](auto a, auto b, auto c){
       std::cout << a << " " << b << " " << c << "\n";
   }));
}

```
--

# Pythagorean Triples 
```c++[1|4|]
auto PythagoreanTriples() {
  auto zip_flat = [](auto f){
   return rpl::Compose(rpl::ZipResult(f), rpl::Flatten());};
  return rpl::Compose(
   rpl::Iota(1),
   zip_flat([](auto c) {return rpl::Iota(1, c+1);}),
   zip_flat([](int c, int a) {return rpl::Iota(a, c+1);}),
   rpl::Swizzle<1, 2, 0>(),
   rpl::Filter(
       [](int a, int b, int c) {return a*a + b*b == c*c;}));
}

```

--

# Pythagorean Triples 
```c++[]
vector<tuple<int,int,int>> triples = 
  rpl::Apply(
    PythagoreanTriples(),
    rpl::Take(100),
    rpl::MakeTuple(), 
    rpl::To<std::vector>());

```

--

# Compose

```c++
auto PythagoreanTriples() {
  auto FlatZip = [](auto f){
    return rpl::Compose(
        rpl::ZipResult([f](auto... args){return f(args...);}),
        rpl::Flatten()
        );
  };

  return rpl::Compose(
      rpl::Iota(1),                                                        
      FlatZip(                                                      
          [](auto z) { return rpl::Iota(1, z + 1); }                       
          ),                                                               
      FlatZip([](auto z, auto x) { return rpl::Iota(x, z + 1); }),  
      rpl::Swizzle<1, 2, 0>(),                                             
      rpl::Filter(                                                         
        [](auto x, auto y, auto z) { return x * x + y * y == z * z; })  
  );


```

---

# Next Slide


 