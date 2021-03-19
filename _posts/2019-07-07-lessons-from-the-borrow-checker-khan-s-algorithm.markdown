---
layout: post
title:  "Lessons From the Borrow Checker: Khan's Algorithm"
date:   2019-07-07
categories: Lessons From the Borrow Checker
---
During a coding interview I was asked to write an algorithm for the problem of [topologically ordering][topo-sort] the nodes of a certain [directed acyclic graph][dag] that the interviewer drew on the whiteboard (see Figure).
Since in the last year I learned Rust and I have been using it for all my personal projects, I decided to go for it.
Needless to say that the borrow checker is not your friend during an interview. In any case I came home with a few lessons.

Regarding the interview, if using Rust and especially if the interviewer is not familiar with it, make clear that you will rush towards a first solution borrowing or even cloning your data wherever the borrow checker complains. After you reached this first solution discuss every clone and come up with possible solutions to avoid them. Finally, always remember to take the time to clean up your solution, no matter how long it took you to devise it.

![The graph on the whiteboard](/blog/assets/images/dag.png)

Maybe the most intuitive algorithm has been given by [A. B. Khan (1962)][khan-algo]. It works as following: first, find all the nodes with no incoming arcs, the sources, and start listing them in the sorting (in any order). Then, take every one of them in turn and remove all its outgoing arcs from the graph. Check if this process creates new sources, add them to the ordering, and keep going.
When this process stops, if the graph has no more arcs, you have a topological sort. Otherwise the graph is not directed acyclic.

The algorithm is pretty straightforward but the data on which it operates, the graph, is complex enough for the borrow checker to give us a small lesson.

## The Code

First of all, we need a way to represent our graph. Notice that Khan's algorithm makes use of the following operations:

- Check if a node has no incoming arcs
- Find all the outgoing neighbors of a node
- Remove an incoming arc from a node

An adjacency list containing both incoming and outgoing arcs, seems to work well enough for all the above operations. A quick way to implement this, using only the standard library, is an hash map and a custom `struct` as follows:

``` rust
struct Neighbors {
    inc: Vec<char>,
    outg: Vec<char>,
}

fn main() {
  let mut graph: HashMap<char, Neighbors> = HashMap::new();
}
```

Notice that we are using chars only because the interviewer example made use of this kind of labels, but at the same time they implement the copy semantic of Rust (instead of the move semantic) thus it is cheap to work with them.

With such a kind of data structure, in order to build the graph in Figure we need a series of insertions like the following one:

``` rust
graph.insert(
    'a',
    Neighbors {
        inc: vec![],
        outg: vec!['c', 'b'],
    },
);
```

We are now ready for the first step of Khan's algorithm, that is finding the sources nodes and adding them to a queue in order to use them afterwords.

``` rust
let mut source_nodes: VecDeque<char> = VecDeque::new();

for (&node, neighbors) in &graph {
    // check if the node has no incoming arcs
    if neighbors.inc.is_empty() {
        source_nodes.push_back(node);
    }
}
```

Not much to say about the previous snippet and , up to this point, no big deal with the borrow checker. Let's enter the crux of the algorithm.

``` rust
while let Some(source) = source_nodes.pop_front() {
    // add the node to the topological sort
    topo_sort.push(source);

    for neighbor in graph.get(&source).unwrap().outg.clone() {
        let neighbor_inc_list = &mut graph.get_mut(&neighbor).unwrap().inc;

        // remove the arc from the current source to the neighbor
        neighbor_inc_list
            .iter()
            .position(|&item| item == source)
            .map(|i| neighbor_inc_list.remove(i));

        // if the neighbor becomes a new source add it to the queue
        if graph.get(&neighbor).unwrap().inc.is_empty() {
            source_nodes.push_back(neighbor);
        }
    }
}
```

As you can see we use our graph in three different lines. First we cycle on the outgoing neighbors of our current source

```rust
graph.get(&source).unwrap().outg.clone()
```

secondly we get the incoming arcs of this neighbor in order to remove the one from the source

```rust
&mut graph.get_mut(&neighbor).unwrap().inc
```

thirdly if the removal of such an arc makes the neighbor a new source we add it to our list of source nodes

```rust
graph.get(&neighbor).unwrap().inc.is_empty()
```

In the third case [non-lexical lifetimes][nll] come to the rescue. In the second one a mutable reference is what we need since we are gonna modify our graph. In the first case we can see a `clone()`. What's going on here? If we try to use a reference here the borrow checker will complain because we are borrowing our graph both mutably and immutably.
But this does not feel right, if you think of it, the first occurrence is considering only outgoing neighbors while the second one only incoming ones, and since we are using chars as labels and copying them around, these information are logically separated and it is safe to work on both of them.
This could remind us of the case in which we want to access at the same time two different indices of a slice and the borrow checker does not allow us to, thus forcing us to use some `unsafe` code and the `split_at` function. But in this case the workaround looks harder to implement (if possible).

Here it comes the *lesson from the borrow checker*. Unexpectedly, the lesson I got from this exercise is not about `unsafe` Rust but about [data-oriented design][dod].
Think of where we started, that is of our definition of a graph.

``` rust
struct Neighbors {
    inc: Vec<char>,
    outg: Vec<char>,
}

fn main() {
  let mut graph: HashMap<char, Neighbors> = HashMap::new();
}
```

It looked simply reasonable to have a struct for the `Neighbors` of a certain node, but this ended up bundling together outgoing and incoming neighbors, raising the need for cloning data.

What if instead we opted for something like the following?

``` rust
let mut inc_graph: HashMap<char, Vec<char>> = HashMap::new();
let mut outg_graph: HashMap<char, Vec<char>> = HashMap::new();
```

Two different and well separated graphs representing incoming and outgoing arcs. For which inserting a node from our example looks like this

``` rust
inc_graph.insert('a', vec![]);
outg_graph.insert('a', vec!['c', 'b']);
```

The first part of the algorithm is very similar

``` rust
let mut topo_sort: Vec<char> = vec![];
let mut source_nodes: VecDeque<char> = VecDeque::new();

for (&node, inc_neighbors) in &inc_graph {
    if inc_neighbors.is_empty() {
        source_nodes.push_back(node);
    }
}
```

The second part though can be modified as following:

``` rust
while let Some(source) = source_nodes.pop_front() {
    topo_sort.push(source);

    for neighbor in outg_graph.get(&source).unwrap() {
        let neighbor_inc_list = inc_graph.get_mut(&neighbor).unwrap();

        neighbor_inc_list
            .iter()
            .position(|&item| item == source)
            .map(|i| neighbor_inc_list.remove(i));

        if inc_graph.get(&neighbor).unwrap().is_empty() {
            source_nodes.push_back(*neighbor);
        }
    }
}
```

As you can see, thanks to the new structure of our graph, we are finally able to get rid of the `clone()`, iterating on the graph of outgoing arcs while operating (mutably) on the graph of incoming arcs.
Furthermore the new structure could be even friendlier on the cache (but this statement would definitely need a more thorough analysis). That is all for this lesson from the borrow-checker, but probably many more will come in the future.

Source code: [clone() version][first-version] and [final version][final-version].

[topo-sort]: https://en.wikipedia.org/wiki/Topological_sorting
[dag]: https://en.wikipedia.org/wiki/Directed_acyclic_graph
[khan-algo]: https://dl.acm.org/citation.cfm?doid=368996.369025
[nll]: https://doc.rust-lang.org/stable/edition-guide/rust-2018/ownership-and-lifetimes/non-lexical-lifetimes.html
[dod]: https://en.wikipedia.org/wiki/Data-oriented_design
[first-version]: https://gist.github.com/manuelmauro/be8c74c6949bebb96cc3e293a3bd4d2f
[final-version]: https://gist.github.com/manuelmauro/43735f70247e659199061310dd4481d7
