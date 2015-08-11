title: How to build a retaining wall
date: 2015-06-24 08:59:32
tags: 
- Python
- Dynamic Programming
- Algorithms
- Recursion
---

When we moved into our first house we inherited a half finished retaining wall.

We had posts already built, 
eventually we where kindly gifted some wood planks represented as a list of lengths `wood_lengths`
to finish the retaining wall we had to cut the planks to get planks of all the lengths we require, 
so we had another list of lengths `required_lengths`.

We want to use as few cuts as possible because we are lazy programmers.

Task: write an algorithm to find the list of cuts you should make.

A cut can be described as:
 
 
 `wood_num` is the index of the length of wood to cut from in `wood_lengths` 
 `cut_amount` the length of wood to cut from this `wood_num`
 
 e.g.
 
 ```
 {
    'wood_num': 0,
    'cut_amount': 10,
 }
 ```
 describes cutting a plank of length 10 from `wood_lengths[0]`

test cases:

```
wood_lengths=[6] required_lengths=[2, 2, 2] -> [
    {
        'wood_num': 0,
        'cut_amount': 2,
    },
    {
        'wood_num': 0,
        'cut_amount': 2,
    },
]
wood_lengths=[2, 4, 6] required_lengths=[2, 2, 2] -> [
    {
        'wood_num': 1,
        'cut_amount': 2,
    },
]
```

Finding an optimal solution means generating all valid possibilities and taking the one with optimal output (with the smallest amount of cuts in this case).

One strategy for trying all possibilities is to imagine:
&nbsp;&nbsp; for `required_lengths[0]` we could take the cut from any length in `wood_lengths`
&nbsp;&nbsp;&nbsp;&nbsp; for `required_lengths[1]` we could take the cut from any length in `wood_lengths`
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ...


This recursive algorithm would run generate a recursive call tree with `len(wood_lengths)` branches at each step and `len(required_lengths)` deep
 so it would take atleast $O( len(wood\\_lengths) ^ {len(required\\_lengths)} )$ time.
 
There are some conditions allowing you to prune this recursive call tree: 
- You can't take a cut of size N from a wood_length less than N.
- Only consider non duplicate wood lengths (If you have planks A and B both of size N then you could take a cut from either, it would be equivalent)


    class RetainingWallSolver(object):
        def retaining_wall(self, wood_lengths, required_lengths):
            self.required_lengths = required_lengths
            return self.retaining_wall_recursive(wood_lengths, len(required_lengths) - 1)
    
        def retaining_wall_recursive(self, wood_lengths, required_length_idx):
            if required_length_idx <= -1:
                return {
                    'cuts': []
                }
    
            current_required_length = self.required_lengths[required_length_idx]
    
            possible_subsolutions = []
    
            # only consider non duplicate wood lengths
            seen_wood_lengths = set()
            for wood_length_idx in range(len(wood_lengths) - 1, -1, -1):
                if wood_lengths[wood_length_idx] in seen_wood_lengths:
                    continue
                seen_wood_lengths.add(wood_lengths[wood_length_idx])
    
                if wood_lengths[wood_length_idx] < current_required_length:
                    # cant cut from this length
                    continue
    
                # what if we chose to cut current_required_length out of this wood length
    
                wood_lengths[wood_length_idx] -= current_required_length
    
                subsolution = self.retaining_wall_recursive(wood_lengths, required_length_idx - 1)
    
                if not subsolution:
                    continue
    
                # don't need to cut if the wood length and required length are the same
                if wood_lengths[wood_length_idx] != 0:
                    subsolution['cuts'].append({
                        'wood_num': wood_length_idx,
                        'cut_amount': current_required_length
                    })
    
                #roll back the state of wood_lengths
                wood_lengths[wood_length_idx] += current_required_length
    
                possible_subsolutions.append(subsolution)
    
            if len(possible_subsolutions) == 0:
                return False
    
            # return the solution with the least number of cuts
            return min(possible_subsolutions, key=lambda s: len(s['cuts']))


Example tests https://github.com/lee101/retaining-wall/blob/master/retaining_wall_test.py

#### Whats wrong with this solution?

Overlapping subsolutions:

consider the call tree when we pass the following parameters


    wood_lengths=[6, 8, 8] required_lengths=[2, 2, 2]
    |--wood_lengths=[4, 8, 8] required_lengths=[2, 2]
    |  |---wood_lengths=[2, 8, 8] required_lengths=[2]
    |  └---wood_lengths=[4, 6, 8] required_lengths=[2] <---
    └--wood_lengths=[6, 6, 8] required_lengths=[2, 2]
       |---wood_lengths=[4, 6, 8] required_lengths=[2] <---
       ...

Notice how `wood_lengths=[4, 6, 8] required_lengths=[2]` is computed twice, 
when inputs get large this can mean the algorithm spends a massive amount of time solving problems that it has already solved.

If the lengths in `wood_lengths` (regardless of order) is the same and `required_length_idx` is the same then `retaining_wall_recursive` should return the same thing.

One way to improve a recursive algorithm to not compute the same thing twice is called memoization (caching the function call), 
We could store a hashtable (a dict) mapping the given arguments to the output of the function.

One way to implement this in python is with decorators, its quite nice because our caching logic is in one place.
    
    
    import copy
    import functools
    
    
    class unordered_memoized(object):
        '''Decorator. Caches a function's return value each time it is called.
        If called later with the same arguments, the cached value is returned
        (not reevaluated).
        list arguments are hashed irrespective of the order of items.
        a deepcopy is cached to avoid changing contents of the cache by operating on a functions return value.
        '''
    
        def __init__(self, func):
            self.func = func
            self.cache = {}
    
        def __call__(self, *args):
            cache_key = []
            for arg in args:
                if isinstance(arg, list):
                    cache_key.append(tuple(sorted(arg)))
                else:
                    cache_key.append(arg)
            cache_key = tuple(cache_key)
    
            if cache_key in self.cache:
                return self.cache[cache_key]
            else:
                value = self.func(*args)
                self.cache[cache_key] = copy.deepcopy(value)
                return value
    
        def __repr__(self):
            '''Return the function's docstring.'''
            return self.func.__doc__
    
        def __get__(self, obj, objtype):
            '''Support instance methods.'''
            return functools.partial(self.__call__, obj)

We can then use the decorator as follows:


    @unordered_memoized
    def retaining_wall_recursive(self, wood_lengths, required_length_idx):
        ...


This runs a lot faster, but does use more memory, could we be better about our memory usage by smartly evicting keys?


Extra follow on problems:

#### Minimise waste

What if there where multiple solutions with the fewest cuts, then how should we decide the best?

That is first and foremost we want to make the fewest cuts

Secondly we want to minimize waste (we want nice long offcuts we can use for other things).

How should we define waste?
Well we know we want to build another retaining wall which will have similar distances between posts (and thus similar `required_lengths`)
As a heuristic we know we will only be able to use a leftover plank if its at least the smallest length in `required_lengths`

This means we can define waste as the amount of times you can cut the smallest `required_length` out of your remaining planks (actually that's how much reuse you get waste is the inverse)

{% math_block %}{% endmath_block %}
