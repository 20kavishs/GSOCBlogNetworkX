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

I'm extremely excited to work on my GSoC project with the amazing NetworkX team.

I am working under the mentorship of Dan, Ross, and Mridul.

I will be making a NetworkX backend with parallelized algorithms, creating a contributor guide, and adding to nx-guides. This blog will be updated with my periodic progress throughout the summer.

## BLOG POST 1: June 20th, 2023

I've made some progress defining the scope of my project and creating the backend.

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
