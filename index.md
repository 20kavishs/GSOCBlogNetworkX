---
title: "GSoC '23: Parallelization and nx-guides"
date: 2022-08-10
draft: false
description: "
Blog post for GSoC '23 Project
"
tags: ["gsoc", "networkx", "nx-guides"]
displayInList: true
author: ["Kavish Senthilkumar"]

---

## BLOG POST 3: August 10th, 2023

Time for another blog post. 

At this point I have good news: I have essentially finished up all the work in my proposal! I've parallelized betweeness_centrality, vitality, tournament, and isolates. I've also completed the traversal notebook and made finishing touches to the contribution guide.

In total, in the time since my blog post:
- I've been cleaning up the parallelized implementations and trying to use some shortcuts that were discussed by my mentors, 2)
- I added the traversal notebook and edited the contribution guide
- I've been taking a look through a new PR in nx_parallel
- I decided to parallelize another function beyond my proposal and parallelized efficiency_measures. 

**Parallelization and new PR**

I looked into using the __wrapped__ decorator but ran into some issues: it did not work for vitality and tournament...kept saying that certain functions were 'not implemented by parallel'. For consistency, I kept the functions the same. I made a small edit to betweenness because it no longer needs the convert helper. All this work is on PR #2, the PR I have been working in thus far. 

I looked through the code in the new PR by dPys (PR #7)  and feel the changes add value...it's a good step in the direction of a more expansive parallel backend and has good ideas for abstraction. It shortens a lot of function implementations with fundamental abstractions, and has good directory structure. I hope to build on it in the future as dPys makes more changes.


**Traversal Notebook and Contribution Guide**

I created a traversal notebook draft and sent it in to my previous improve consistency PR (#98). I aimed to not use static images and create all graphics/animations with networkx. I explained the foundations of BFS and DFS. I took a casual yet informative approach and explained all traversal functions (i.e. edge_bfs, predecessors, etc) in NetworkX. I made sure to adhere to our format guidelines throughout the notebook. I added to PR #98 because I wanted to build on top of my existing consistency changes from what Ross (one of my mentors) mentioned

My goal was to make the notebook engaging and accessible to everyone, striking a balance between being informative and enjoyable to read. I am considering adding some more practical examples for an analysis. Some of my animations were pretty cool!

I also added some finished touches to the format guidelines and contribution guide. 

**Parallelization and efficiency_measures**

I looked beyond my proposal to see what else I could add to. I decided to work on parallelizing more functions. I parallelized efficiency measures and added the corresponding tests. I parallelized efficiency_measures with a similar chunking and process-based parallelism approach I've used when chunking vertices into groups. This entailed creating a "subset" helper function and using joblib Parallel, the gist of which I have below. 

```python
    total_cores = cpu_count()
    num_chunks = max(len(G.nodes) // total_cores, 1)
    node_chunks = list(chunks(G.nodes, num_chunks))
    efficiencies = Parallel(n_jobs=total_cores)(
        delayed(local_efficiency_node_subset)(G, chunk) for chunk in node_chunks
    )
    return sum(efficiencies) / len(G)
    

def local_efficiency_node_subset(G, nodes):
    return sum(global_efficiency(G.subgraph(G[v])) for v in nodes)
```


**Next Steps**

Next, I will continue seeing what else I can add (I'm considering 1) updates to nx_parallel documentation and 2) parallelizing more functions). As I wrap up my contributions, I'll start working on my final submission deliverable for Google Summer of Code. It's been an incredible experience, and I'm grateful for the opportunity to contribute to this fantastic project.

## BLOG POST 2: July 9th, 2023
At last it is time for another blog post. A good deal of new changes here. 

I having been spending A LOT of time on this. It makes me want to give some kudos to the other contributors...I thought implementing some new parallel functions would be quick based on my work for isolates, but it most definitely was not. Figuring out testing, making the implementations work with PR #6688, and choosing how to parallelize the functions took quite a bit of time. 

Overall, I changed up the interface and added new ParallelGraph types, cloned the networkx tests to the repo to test my functions/got around the testing issue before, and added parallel implementations for closeness_vitality and tournament (a bit ahead of schedule). 

To be able to implement functions without running into dispatch decorator issues, I switched to the version of network in PR 6688 and made nx_parallel work with that. I found that the PR marked all the functions I wanted to parallelize with the dispatch decorator, so there was no need to make my own version of the PR. 

**Class Additions and Interface**

To pass some texts and allow full networkx functionality with nx_parallel, I added ParallelDiGraph, ParallelMultiDiGraph, and ParallelMultiGraph as simple wrappers that convert back to their namesake networkx Graph type with to_networkx(). I did this after seeing something similar with GraphBLAS...it helped me pass a few more tests. 

I also changed the interface a bit to make things work with PR 6688. I looked at dispatch_interface.py under /classes/tests for PR 6688 as a model, and used that function's version of the Dispatcher (along with a convert method).

**Testing**

I tried many ways to be able to use pytest and the networkx testing suite, but I was unable to. Eventually, to get around the "not implemented by parallel" errors I added my own version of networkx tests following the same structure as the networkx directory. 

In these tests, I simply converted from the graphs to the proper type of ParallelGraph. 

In my function implementations, I added all functions in every given file I was working in and used the to_networkx() methods to convert between networkx graph types and nx_parallel graph types. This allowed me to avoid reimplementation. 

**Parallel Implementations**

The most interesting part!

I am a bit ahead of schedule and wrapped up parallel implementations for tournament.py and vitality.py that pass all tests. I am still a bit tied up on making all the betweenness_centrality tests pass, so I will continue working on that in a bit. I think the issue there is that betweenness_centrality_subset does not properly take endpoints into account...so I will need to make another version of the accumulate_subset helper based on betweenness_centrality's accumulate_endpoints. 

First note I added a chunks helper to algorithms/utils/chunk.py. Also used a convert function to interchange between types.

Isolates
- Used chunks helper function instead of thread-level...as you'll see later I switched to mainly creating chunks and then using process-based parallelism instead of threads...I found it reduced overhead from allocation

Betweenness Centrality
- Mentioned above

Closeness Vitality
- Tried chunking here, (note to self relook at this later). Ended up sticking to process-based parallelism without chunking due to runtime. 
- Passes all tests and has significant speedup
- Didn't use helper function because of recursion

```python
def closeness_vitality(G, node=None, weight=None, wiener_index=None):
    if isinstance(G, ParallelMultiDiGraph):
        I = ParallelMultiDiGraph.to_networkx(G)
    if isinstance(G, ParallelMultiGraph):
        I = ParallelMultiGraph.to_networkx(G)
    if isinstance(G, ParallelDiGraph):
        I = ParallelDiGraph.to_networkx(G)
    if isinstance(G, ParallelGraph):
        I = ParallelGraph.to_networkx(G)
    if wiener_index is None:
        wiener_index = nx.wiener_index(I, weight=weight)
    if node is not None:
        after = nx.wiener_index(G.subgraph(set(G) - {node}), weight=weight)
        return wiener_index - after
    vitality = partial(closeness_vitality, I, weight=weight, wiener_index=wiener_index)
    result = Parallel(n_jobs=-1)(
        delayed(lambda v: (v, vitality(v)))(v) for v in I
    )
    return dict(result)
```

Tournament
- This implementation was quite interesting! I found four areas where I could potentially apply parallelism, as indicated by the TODOs. After some experimentation, I discovered that breaking the tasks into chunks and using process-based parallelism yielded the best results in terms of performance.

- I messed around a bit and again found that breaking up into chunks and using process-based parallelism was faster. 

- I didn't actually parallelize is_closed itself...I found that the run time of is_closed was so short it didn't justify the overhead of either creating threads or allocating to processors. I thought about it like this: assuming p is the processing time for all tasks, for parallel processing to be faster than sequential we need p/num_cores + o << p, where o is overhead from joblib. we need o << ~p, which was not occuring with is_closed

- After experiments I did find it was best to parallelize two_neighborhood's application on vertices and is_closed's application on vertices. 

- I didn't parallelize the two_neighborhood function itself for similar reasons as is_closed. Instead, I focused on parallelizing the application of the function, which resulted in better overall performance.

- Was forced to use process-based parallelism by joblib for is_strongly_connected (not an issue though), just used chunking + process-based parallelism.

- Passes all tests and has significant speedup

```python
def is_reachable(G, s, t):
    """Subset version of two_neighborhood"""
    def two_neighborhood_subset(G, chunk):
        reList = set()
        for v in chunk:
            reList.update({x for x in G if x == v or x in G[v] or any(is_path(I, [v, z, x]) for z in G)})
        return reList
    
    """Identical to networkx helper implementation"""
    def is_closed(G, nodes):
        return all(v in G[u] for u in set(G) - nodes for v in nodes)

    """helper to check closure conditions for chunk (iterable) of neighborhoods"""
    def check_closure_subset(chunk):
            return all(not (is_closed(G, S) and s in S and t not in S) for S in chunk)

    I = _convert(G)
    num_chunks = max(len(G) // cpu_count(), 1)

    #send chunk of vertices to each process (calculating neighborhoods)
    node_chunks = list(chunks(G.nodes, num_chunks))  
    neighborhoods = Parallel(n_jobs=-1)(
        delayed(two_neighborhood_subset)(G, chunk) for chunk in node_chunks
    )

    #send chunk of neighborhoods to each process (checking closure conditions)
    neighborhood_chunks = list(chunks(neighborhoods, num_chunks))  
    results = Parallel(n_jobs=-1, backend="loky")(
        delayed(check_closure_subset)(chunk) for chunk in neighborhood_chunks
    )
    return all(results)

def tournament_is_strongly_connected(G):
    """Subset version of is_reachable"""
    def is_reachable_subset(G, chunk):
        re = set()
        for v in chunk:
            re.update(is_reachable(G, u, v) for u in G)
        return all(re)
    num_chunks = max(len(G) // cpu_count(), 1)
    node_chunks = list(chunks(G.nodes, num_chunks))  
    results = Parallel(n_jobs=-1)(delayed(is_reachable_subset)(G, chunk) for chunk in node_chunks)
    return all(results)
```

**Next Steps**

- Fix betweenness_centrality
- Clean up imports
- Find a cleaner way to test + organize
- Transition into nx-guides
- Maybe parallelize another function for fun






## BLOG POST 1: June 20th, 2023

I'm extremely excited to work on my GSoC project with the amazing NetworkX team.

I am working under the mentorship of Dan, Ross, and Mridul.

I will be making a NetworkX backend with parallelized algorithms, creating a contributor guide, and adding to nx-guides. This blog will be updated with my periodic progress throughout the summer.I've made some progress defining the scope of my project and creating the backend.

I looked through functions of interest and those marked with a TODO to parallelize, and chose a few functions/files: isolates.py, beteeenness centrality, tournament.py, and closeness vitality.

In addition to that, I'm going to make a traversal Jupyter notebook and a contributor guide throughout the summer.

I've set up my environment and read through the NetworkX documentation on experimental backend plugins (took a look at GraphBLAS's backend to NetworkX as well). Note the use of a backend plugin instead of direct implementations in NetworkX: this is because of the need for core NetworkX to have minimal dependencies and ease of version control.

In terms of other progress, in the past week I've worked on parallelizing isolate.py.

The best candidate to parallelize is the number_of_isolates function, which was also marked with a TODO.

I tried a number of ways to parallelize the function, and compared the runtime to native NetworkX. Some notes:

1. I first used joblib.Parallel...after testing with graphs (random edge probability ~0.5) with 100-100000 vertices I found there was no significant speedup (the parallelized implementation was actually 2-3x slower). This was likely due to process-based parallelism being used in lieu of thread-based parallelism and leading to unnecessary overhead.

2. With joblib.Parallel I then tried using the threading backend instead of loky (also tried multiprocessing) and saw some speed up, but still longer processing times than the normal implementation.

3. Interestingly, I found the fastest runtimes when I used the iterator from nx.isolates() and a lambda function to count the number of isolates like below. Note n_jobs=-1 makes use of all available processors.

```python
num_isolates = Parallel(n_jobs=-1, backend="threading")(
    delayed(lambda x: 1)(v) for v in nx.isolates(G)
)
return sum(num_isolates)
```

This was faster than calling is_isolate multiple times, checking the graph myself for the right edges, and all other methods of processing I used in the delayed() call

4. I tried some implementations with concurrent futures, but still got the fastest runtimes with joblib.Parallel

5. I read about how "chunking" could be useful: specifically, dividing the vertices into chunks (one chunk per available processor) and dispatching each chunk of vertices to each processing unit. I tried something like the below with joblib.Parallel with threading, concurrent.futures, and multiprocessing. I found the fastest to be my implementations with just joblib.Parallel and found there really was a negligible speedup over no chunking.

```python
def number_of_isolates(G):
    num_processors = Parallel(n_jobs=-1, backend="threading").n_jobs
    isolates = list(nx.isolates(G))
    chunk_size = len(isolates) // num_processors

    def process_chunk(chunk):
        return sum(1 for v in chunk if nx.is_isolate(v, G))

    results = Parallel(n_jobs=num_processors, backend="threading")(
        delayed(process_chunk)(isolates[i : i + chunk_size])
        for i in range(0, len(isolates), chunk_size)
    )

    return sum(results)
```

6. As such, I stuck with the implementation in 1.

I plan to use a similar approach for betweenness centrality, but I want to make sure I can test isolates first.
