---
layout: post
title:  "Speeding up Advent of code problems with Mojo"
date:   2024-07-13 12:13:23 +0100
categories: jekyll update
---

Mojo is a young programming language which styles itself as having "the flexibility of Python with the speed of C".
Jeremy Howard gives a nice introduction and motivation for it here. I got curious about Mojo because I thought it
would be fun to do some programming where I really try to optimise for performance.  I'm a backend Python engineer in my
day job, and since Python is such a slow language it's generally not worthwhile for me to think very hard about speeding
up my code.

I decided to work on Advent of Code 2023 problem 5b as an exercise in learning Mojo.  Why this problem?  I'd heard
somewhere that Advent of Code always includes at least one problem which you can solve in a naive brute-force
algorithm with a fast language but not in a slow one, and last year 5b was such a problem. I set myself the challenge of
solving 5b using only the naive algorithm.  I thought that this challenge would be simple enough as a first exercise in
learning Mojo, and also meaty enough that I could get some performance gains using Mojo's cool features.

[Here's](https://adventofcode.com/2023/day/5) the link to AoC 2023 problem 5, for the rest of this post I'll assume
you've read that.  Part 5b only unlocks after you've completed 5a, but the part 5b problem statement is a small
modification to 5a.  The list of seeds now represents *ranges* of seeds, so `seeds: 79 14 55 13` represents two ranges,
where the first range is `79, 80, ... 92` and the second is `55, 56, .. 67` , and you now need to consider all those 27
numbers when searching for the lowest location number.

## Python solution

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
    total_numbers_to_check = sum([seed for i, seed in enumerate(seeds_int) if i % 2 != 0])
    print(f"{total_numbers_to_check=}")

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
numbers on my MacBook Air M2.  There are a total of 2,482,221,626 seed numbers in my test file, so I estimate that this
script would take about 4 hours to solve the problem on my machine.

## Mojo solution

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

And the parsing code becomes

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
