## Rappel in depth
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
* `Apply` treats passing in `Compose` as if you passed what is in compose


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


