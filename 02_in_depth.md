## Rappel background
--
### Terminology
* Pipeline 
* Stage
* Processing style
--
#### Processing Style
* How a stage processes elements
* Two kinds
    * Incremental
    * Complete
--

#### Incremental
* Processes elements one at a time
* `Filter`, `Transform` are all examples of *incremental*
* We can evaluate an element and output the result immediately without waiting for other elements
--
#### Complete
* Process a single element as a whole
* Sorting is an example of *complete* 
* You need all the elements to sort
* Surprising
    * `Max`
    * You take all the elements and reduce them to a single value.
    * You are treating that value as a whole, not part of something else.
--
#### Input and Output
* The input and output processing style don't have to be the same.
* Four possiblities


|Input|Output|
|---- | ----|
|Incremental | Incremental|
|Incremental | Complete|
|Complete | Complete|
|Complete | Incremental|

--
#### Incremental -> Incremental
* `Transform`
* `Filter`
* Take a single element, and output an element as part of something larger
--
#### Incremental -> Complete
* `To<std::vector>`
* `Max`
* `AnyOf`, `AllOf`, `NoneOf`
* Take values that are parts of a whole and collect them into new whole
* `Accumulate` is the generalization of transformation
--

#### Complete -> Complete
* `Sort` (based on `std::sort`)
* Transforms a collection, sorts it, and outputs that collection as a whole
* Others include `PartialSort`, `StableSort`, `NthElement`, `Unique`
--

#### Complete -> Incremental
* Iterating a range
* Implicit in Rappel

--
#### Rules for Incremental and Complete
* The input processing style of a stage has to match the output processing style of the previous stage
    * We can implicitly convert *complete* to *incremental*
* The final stage passed to `Apply` has to be *complete*
    * `Apply` needs to return a single value that stands on its own.

--
### Apply
```
template <typename Range, typename... Stages>
[[nodiscard]] auto Apply(Range&& range, Stages&&... stages);
```
* The first parameter is a `range`. It is actually more general and can be any complete value.
* The `stages` is one or more of the above stages.
* Note that the return type is `auto`. It will return a value and not a reference. This is for safety by default.

--
### Compose
* Creates new stages by composition
* It is just a thin wrapper over `std::tuple`.
* `Apply` treats passing in `Compose` as if you passed what is in the elements directly.

--

### Test
* `std::vector<int> v{...};`
* Assume we capture the return value

```c++[1|2|3|4|5]
Apply(v, Max());
Apply(v,Transform(triple));
Apply(v,Transform(triple),To<std::vector>());
Apply(v,Transform(triple),Sort());
Apply(v,Transform(triple),To<std::vector>(),Sort());
```
Note:
1,3,5 are correct
--
### Generators
* Input iterators can be difficult to write
* Want an easy way to make generators such as `Iota`

```c++
template <typename T, typename GeneratorInvocable>
auto Generator(GeneratorInvocable&& generator_invocable);
```
--
### Iota
```c++[|4]
template <typename T>
auto Iota(T begin) {
  return rpl::Generator<T>([begin](auto output) mutable {
    std::move(output)(begin);
    ++begin;
  });
};
--
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
--

#### Stages
--

#### Complete Transforms
* TransformComplete
* Sort
* StableSort
* PartialSort
* NthElement
* Unique
* UnwrapOptionalComplete
--
#### Incremental To Complete Accumulators
* Accumulate
* AccumulateInPlace
* ForEach
* To
* ToMap
* Into
--
#### Accumulators (continued)
* Front
* Count
* Min
* Max
* MinMax
* AnyOf
* AllOf
* NoneOf

--
#### Filters
* Filter
* Take
* TakeWhile
* Skip
* SkipWhile
* FilterDuplicates
--
#### Incremental Transforms
* Transform
* TransformArg
* Ref
* Move
* LRef
* Deref
* AddressOf
* Get
* Cast
* Construct
-- 
#### Incremental Argument Manipulation
* ExpandTuple
* Enumerate
* ZipWith
* ZipResult
* CartesianProductWith
* Swizzle
* MakeTuple
* MakePair
* Flatten
--
#### Flat Map
* FlatMap
* Can output zero to many results
--
#### Higher Order
* GroupBy
* MapGroupBy
* Chunk
* Tee
--
#### Sequence extension
* RepeatWhile
* RepeatN
* ChainBefore
* ChainAfter
--
#### Error Handling
* UnwrapOptional
* UnwrapOptionalArg
--
## Implementing Stages
 * Warning: Internals ahead
 * Simplifications and hand-waving ahead
--
### Stage Impl
```c++
template<typename InputType, typename... OtherParameters>
struct FooStageImpl{
    using OutputType = ;

    // Required for incremental 
    template<typename Next>
    void ProcessIncremental(InputType input, Next&& next);
    // Optional for incremental
    template<typename Next>
    decltype(auto) End(Next&& next);
    bool Done() const;

    // Required for complete
    decltype(auto) ProcessComplete(InputType input, Next&& next);
};
```
Note: InputType is always a reference type
--
### Complete Transform
```
template <typename InputType, typename F>
struct TransformCompleteImpl {
  F f;

  using OutputType = std::invoke_result_t<F,InputType>; 

  template <typename Next>
  decltype(auto) ProcessComplete(InputType input, Next&& next) {
    return next.ProcessComplete(std::invoke(f,
      static_cast<InputType>(input)));
  }
};

```
--
### Incremental Transform
```c++
template <typename InputType, typename F>
struct TransformImpl {
  using OutputType = std::invoke_result_t<F,InputType>; 

  template <typename Next>
  decltype(auto) ProcessIncremental(InputType input, Next&& next) {
    return next.ProcessIncremental(std::invoke(f,
      static_cast<InputType>(input)));
  }
};

```
--
### Filter
```c++
template <typename InputType, typename Predicate>
struct FilterImplImpl<InputType, Predicate> {
  using OutputType = InputType;
  Predicate f;

  template <typename Next>
  void ProcessIncremental(InputType input, Next&& next) {
    if(std::invoke(f,std::as_const(input))){
        next.ProcessIncremental(static_cast<InputType>(input));
    }
};

```
--
### Holding Impls
```c++
template <typename Chain, size_t I, typename InputTypeParam,
          typename Stage>
class StageChainComponent {
 public:
  using InputType = InputTypeParam;
  using OutputType = typename Stage::OutputType;
private:
  Stage stage_;
public:
  decltype(auto) Get(Index<I>) { return *this; }
  auto& GetChain() { return static_cast<Chain&>(*this); }
  decltype(auto) Next() { return GetChain().Get(Index<I + 1>()); }
```
--
```c++
  constexpr decltype(auto) End() {
    if constexpr (kHasEnd<Stage, decltype(Next())>) {
      return stage_.End(Next());
    } else {
      return Next().End();
    }
  }
  constexpr bool Done() const {
    if constexpr (kHasDone<Stage>) {
      return stage_.Done();
    } else {
      return false;
    }
  }
```
--
```c++
  void ProcessIncremental(InputType t) {
    stage_.ProcessIncremental(static_cast<InputType>(t), Next());
  }
  decltype(auto) ProcessComplete(InputType t) {
    if constexpr(is_incremental){
     for(auto&& v: t) // Handwavy
       stage_.ProcessIncremental(v,Next());
     return End();
    }else{
      return 
       stage_.ProcessComplete(static_cast<InputType>(t), Next());
     }
  }
};

```
--
### Chain
```c++
template<size_t... Is,typename... Components>
struct Chain<std::index_sequence<Is...>,Components...>
:ComponentT<Chain,Is,Components>...{
    using ComponentT<Chain,Is,Components>::Get...;

    template<typename T>
    auto ProcessComplete(T&& t){
        return std::forward<T>(t);
    }

    auto& Get(std::Index<sizeof...(Is)>){
        return *this;
    }
};

```
--
```c++
template<typename Range, typename... Stages>
auto Apply(Range&& range, Stages&&... stages){
    auto chain = MakeChain(std::forward<Stages>(stages)...);
    return chain.ProcessComplete(std::forward<Range>(range));
}

```


