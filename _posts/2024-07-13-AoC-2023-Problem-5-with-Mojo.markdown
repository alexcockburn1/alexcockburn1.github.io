---
layout: post
title:  "Speeding up Advent of Code problems with Mojo"
date:   2024-07-13 12:13:23 +0100
tags: [Software engineering]
---

Mojo is a young programming language which styles itself as having "the flexibility of Python with the speed of C".
Jeremy Howard gives a nice introduction and motivation for it [here](https://www.fast.ai/posts/2023-05-03-mojo-launch.html).
I got curious about Mojo because I thought it
would be fun to do some programming where I really try to optimise for performance.  I'm a backend Python engineer in my
day job, and since Python is a slow language I'm generally not thinking hard about performance past a certain point.

I decided to work on Advent of Code 2023 problem 5b as an exercise in learning Mojo.  Why this problem?  I'd heard
somewhere that Advent of Code always includes at least one problem which you can solve in a naive brute-force
algorithm with a fast language but not in a slow one, and last year 5b was such a problem. I set myself the challenge of
solving 5b using only the naive algorithm.  I thought that this challenge would be simple enough as a first exercise in
learning Mojo, and also meaty enough that I could get some performance gains using Mojo's cool features. In this post
I'll go through a naive Python solution, then port the solution to Mojo and show how it can be sped up using
parallelization and vectorization. Disclaimer: I'm
very much still a Mojo & systems engineering noob, so my solution is probably not a perfect example of Mojo code.

[Here's](https://adventofcode.com/2023/day/5) the link to AoC 2023 problem 5, for the rest of this post I'll assume
you've read that.  Part 5b only unlocks after you've completed 5a, but the part 5b problem statement is a small
modification to 5a.  The list of seeds now represents *ranges* of seeds, so `seeds: 79 14 55 13` represents two ranges,
where the first range is `79, 80, ... 92` and the second is `55, 56, .. 67` , and you now need to consider all those 27
numbers when searching for the lowest location number.

## Python solution 🐍

I started by defining a simple `ShiftInterval` class to hold the data for each row of the mapping:

```python
class ShiftInterval:
    def __init__(self, destination_start: int, source_start: int, source_end: int):
        self.source_start = source_start
        self.source_end = source_end
        self.shift = destination_start - source_start
```

The next step is to create a class which assembles a list of `ShiftInterval` instances  together with the logic for applying them to a seed number[^1]:

```python
class Mapping:
    def __init__(self):
        self.intervals = []

    def apply_mapping(self, number: int) -> int:
        for i in range(len(self.intervals)):
            interval = self.intervals[i]
            if interval.source_start <= number < interval.source_end:
                return number + interval.shift
        return number
```

Finally I created a class for applying a list of `Mappings` sequentially:

```python
class MappingList:
    def __init__(self, mappings: List[Mapping]):
        self.mappings = mappings

    def apply_mappings(self, number: int) -> int:
        mapped_number = number
        for i in range(len(self.mappings)):
            mapping = self.mappings[i]
            mapped_number = mapping.apply_mapping(mapped_number)
        return mapped_number
```

and here's the rest of the solution which just parses the input `test_file.txt` and finds the lowest location number by looping over all possible seed numbers:

```python
def parse_list_of_ints(in_string: str) -> List[int]:
    str_list = in_string.split(" ")
    return [int(s) for s in str_list]


def part5b(overall_mapping: MappingList, seeds_int: List[int]):
    minimum = 1000000000000
    for seed_index in range(0, len(seeds_int), 2):
        for i in range(seeds_int[seed_index], seeds_int[seed_index] + seeds_int[seed_index + 1]):
            transformed_number = overall_mapping.apply_mappings(i)
            if transformed_number < minimum:
                minimum = transformed_number
        print("Finished seed_index " + str(seed_index))
    print(str(minimum))


def main():
    with open("test_file.txt", "r") as handle:
        text = handle.read()
        lines = text.split("\n")
        seeds_line = lines[0]
        seeds_str = seeds_line[7:]
        seeds_int = parse_list_of_ints(seeds_str)
        mappings = []
        for i in range(1, len(lines)):
            line = lines[i]
            if "map" in line:
                mappings.append(Mapping())
            elif len(line) > 0:
                row_vals = parse_list_of_ints(line)
                mappings[-1].intervals.append(ShiftInterval(source_start=row_vals[1], source_end=row_vals[1] + row_vals[2], destination_start=row_vals[0]))
        overall_mapping = MappingList(mappings)

        part5b(overall_mapping, seeds_int)

```

I didn't run this script to completion, but it took 5.91 seconds to iterate through the first 1,000,000 seed
numbers from [the test file](https://github.com/alexcockburn1/AoC_prob5_mojo/blob/main/test_file.txt) on my MacBook Air
M2.  There are a total of 2,482,221,626 seed numbers in my test file, so I estimate that this script would take about
4 hours to solve the problem on my machine.

## Mojo solution 🔥

### Structs

As a first step, I ported my Python classes to Mojo structs.  Mojo structs are less dynamic than Python classes and
enforce stricter type declarations to allow the compiler to perform more optimisations.  My `ShiftInterval` becomes:

```
@value
struct ShiftInterval:
    var source_start: Int
    var source_end: Int
    var shift: Int

    fn __init__(inout self, destination_start: Int, source_start: Int, source_end: Int):
        self.source_start = source_start
        self.source_end = source_end
        self.shift = destination_start - source_start
```

Differences from the Python class:
- `@value` annotation (TODO is "annotation" the right word?) which gets the Mojo compiler to create some utility methods
- Typed variable declarations using `var`.  Mojo requires these in structs.
- `inout` is related to Mojo's memory management system, which I won't be covering in this post.
- We are using `fn` instead of `def`.  `fn` methods must specify argument and return types, and are more performant than `def` methods.

The `Mapping` and `MappingList` classes become:

```
@value
struct Mapping:
    var intervals: List[ShiftInterval]

    fn __init__(inout self):
        self.intervals = List[ShiftInterval]()

    fn apply_mapping(self, number: Int) -> Int:
        for i in range(len(self.intervals)):
            var interval = self.intervals[i]
            if interval.source_start <= number < interval.source_end:
                return number + interval.shift
        return number


@value
struct MappingList:
    var mappings: List[Mapping]

    fn __init__(inout self, mappings: List[Mapping]):
        self.mappings = mappings

    fn apply_mappings(self, number: Int) -> Int:
        var mapped_number = number
        for i in range(len(self.mappings)):
            var mapping = self.mappings[i]
            mapped_number = mapping.apply_mapping(mapped_number)
        return mapped_number
```

And the code for parsing input and looping over seed numbers is essentially unchanged:

```
def part5b(overall_mapping: MappingList, seeds_int: List[Int]):
    minimum = Int(1000000000000)
    for seed_index in range(0, len(seeds_int), 2):
        for i in range(seeds_int[seed_index], seeds_int[seed_index] + seeds_int[seed_index + 1]):
            transformed_number = overall_mapping.apply_mappings(i)
            if transformed_number < minimum:
                minimum = transformed_number
        print("Finished seed_index " + str(seed_index))
    print("5b solutions is: " + str(minimum))


def main():
    with open("test_file_small.txt", "r") as handle:
        text = handle.read()
        lines = text.split("\n")
        seeds_line = lines[0]
        seeds_str = seeds_line[7:]
        seeds_int = parse_list_of_ints(seeds_str)
        var mappings = List[Mapping]()
        for i in range(1, len(lines)):
            line = lines[i]
            if "map" in line:
                mappings.append(Mapping())
            elif len(line) > 0:
                row_vals = parse_list_of_ints(line)
                mappings[-1].intervals.append(ShiftInterval(source_start=row_vals[1], source_end=row_vals[1] + row_vals[2], destination_start=row_vals[0]))
        overall_mapping = MappingList(mappings)
        part5b(overall_mapping, seeds_int)
```

This Mojo solution solves the problem in about 25 minutes on my machine - roughly a 10x improvement over Python!  The
Mojo docs state that Mojo gives a similar improvement for [matrix multiplication](https://docs.modular.com/mojo/notebooks/Matmul#importing-the-python-implementation-to-mojo),
so Mojo is performing as advertised here.

### Parallelization

CPython doesn't natively offer true parallelization because of the [GIL](https://en.wikipedia.org/wiki/Global_interpreter_lock),
but Mojo does and implementing it is straightforward. I added a new `apply_mappings_parallelized` function to the
`MappingList`:

```
@value
struct MappingList:
    var mappings: List[Mapping]

    fn __init__(inout self, mappings: List[Mapping]):
        self.mappings = mappings

    fn apply_mappings(self, number: Int) -> Int:
        var mapped_number = number
        for i in range(len(self.mappings)):
            var mapping = self.mappings[i]
            mapped_number = mapping.apply_mapping(mapped_number)
        return mapped_number

    fn apply_mappings_parallelized(self, range_start: Int, input_range: Int, inout overall_min: Int):

        @parameter
        fn apply_mappings_worker(i: Int):
            var mapped_number = self.apply_mappings(range_start + i)
            if mapped_number < overall_min:
                overall_min = mapped_number

        parallelize[apply_mappings_worker](input_range, num_workers=4)
```

I was really just pattern-matching
on the Modular doc's [matmul example](https://docs.modular.com/mojo/notebooks/Matmul#parallelizing-matmul) when I wrote
this, but for more details on the syntax you can see the docs on
[parallelize](https://docs.modular.com/mojo/stdlib/algorithm/functional/parallelize)
and [parametric closure](https://docs.modular.com/mojo/manual/decorators/parameter#parametric-closure).

The `parallelize` function requires a range to be parallelized over, so here it's effectively doing the work
of the `for i in range(seeds_int[seed_index], seeds_int[seed_index] + seeds_int[seed_index + 1])` loop in the
`part5b` function.  This means that I need to refactor `part5b` to take out that loop:

```
def part5b_parallelized(overall_mapping: OverallMapping, seeds_int: List[Int]):
    minimum = Int(1000000000000)
    for seed_index in range(0, len(seeds_int), 2):
        range_start = seeds_int[seed_index]
        input_range = seeds_int[seed_index + 1]
        overall_mapping.apply_mappings_parallelized(range_start, input_range, minimum)
        print("Finished seed_index " + str(seed_index) + " of " + str(len(seeds_int)))
    print("5b solutions is: " + str(minimum))
```

With 4 workers (the maximum available on my machine) the runtime drops roughly 4x as expected, down from ~25 minutes to
~7 minutes.

### Vectorization

The examples in the Mojo docs use vectorization as another way to improve performance.  I had never heard of
vectorization before and I found [this stack overflow answer](https://stackoverflow.com/a/1422181) pretty helpful.
In short, it means exploiting the ability of CPUs to apply the same instruction to multiple pieces of data at once.
These CPU instructions are called SIMD (Single Instruction Multiple Data), and Mojo provides the programmer with
direct access to these.

Like matrix multiplication, this advent of code problem is essentially 2-dimensional: I'm iterating over an outer loop
of seed numbers and an inner loop of mappings.  I already parallelized the outer loop, so the only loop I have available
to vectorize is the inner loop over mappings.  For each seed number I split the mappings into batches and perform
comparisons across the whole batch.

The [matrix multiplication example](https://stackoverflow.com/a/1422181) uses a `DTypePointer` to hold the matrix
values.  As far as I can tell, this helps in two ways: firstly data loading and writing is very fast for `DTypePointer`,
and secondly `DTypePointer.load` returns data of SIMD type, which is handy for performing SIMD instructions.  To make
use of `DTypePointer` I created a new `MappingVectorized` struct which takes a `Mapping` instance in its constructor:

```
alias type = DType.uint32

@value
struct MappingVectorized:
    var source_starts: DTypePointer[type]
    var source_ends: DTypePointer[type]
    var shifts: DTypePointer[type]
    var mapping_size: Int

    fn __init__(inout self, mapping: Mapping):
        self.mapping_size = len(mapping.intervals)
        self.source_starts = DTypePointer[type].alloc(self.mapping_size)
        self.source_ends = DTypePointer[type].alloc(self.mapping_size)
        self.shifts = DTypePointer[type].alloc(self.mapping_size)
        for i in range(self.mapping_size):
            self.source_starts[i] = mapping.intervals[i].source_start
            self.source_ends[i] = mapping.intervals[i].source_end
            self.shifts[i] = mapping.intervals[i].shift
```
The constructor extracts the starts, ends and shifts from all the `ShiftInterval` instances and moves the data into
3 `DTypePointer`s.  Here is the code for applying the mapping using vectorization:

```
alias type = DType.uint32
alias simdwidth = simdwidthof[type]()  # This is the size of the vectorization batch.  On my machine this is 4.

@value
struct MappingVectorized:
    ...

    fn apply_mapping(self, number: Int) -> Int:

        for i in range(self.mapping_size//simdwidth):
            var number_simd = SIMD[type, simdwidth](number)
            var source_starts_batch = self.source_starts.load[width=simdwidth](simdwidth*i)
            var source_ends_batch = self.source_ends.load[width=simdwidth](simdwidth*i)
            var shifts_batch = self.shifts.load[width=simdwidth](simdwidth*i)
            var above_lower_bound = number_simd >= source_starts_batch
            var below_upper_bound = number_simd < source_ends_batch
            var is_within_bounds = below_upper_bound.__and__(above_lower_bound)
            if is_within_bounds.reduce_or():
                for j in range(simdwidth):
                    if is_within_bounds[j]:
                        return int(number + shifts_batch[j])

        var number_simd_scalar = SIMD[type, 1](number)
        for i in range(simdwidth*(self.mapping_size//simdwidth), self.mapping_size):
            if self.source_starts[i] <= number_simd_scalar < self.source_ends[i]:
                return int(number_simd_scalar + self.shifts[i])
        return number
```

If I swap the `Mapping` struct for `MappingVectorized` in my parallelized solution, the runtime now drops from ~7
minutes down to 25 seconds!  It turns out that most of the gain is due to using `DTypePointer`.  If I set `simdwidth`
to `1` then the runtime is about 1 minute, so vectorizing the comparisons "only" provides 2x improvement, compared with the
7x improvement I get just from using `DTypePointer`.

## Conclusion

In this post I ported a simple Python advent of code solution to Mojo, and showed how to speed it up using vectorization
and parallelization. The runtime of the original Python solution was around 4 hours, and my final Mojo solution runs in
25 seconds, approximately a 500x improvement!  It was a lot of fun for me to dive into a new language, and very
satisfying that I was able to get such good performance improvements without too much effort.

## Footnotes

[^1]: I've tried to make this Python code portable to Mojo, which is why I've used
    ```python
    for i in range(len(self.intervals)):
        interval = self.intervals[i]
    ```
    rather than the much more preferable:
    ```python
    for interval in self.intervals:
    ```
    because this idiom doesn't seem to exist in Mojo at time of writing.
