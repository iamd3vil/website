+++
title = "Using WaitGroups in Go without Leaky Abstractions"
description = ""
author = "Sarat Chandra"
date = 2023-05-25
slug = "waitgroup_leaky_abstractions"
draft = false

[taxonomies]
tags = ["go", "golang", "style"]
+++

In many Go codebases, I come across patterns like this:

```go
func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go func1(&wg)
    wg.Wait()
}
```

In this pattern, the goal is to let a function call the `Done()` method once it is finished, synchronizing multiple goroutines with a `sync.WaitGroup`. However, there's a disadvantage: now the function needs to take a WaitGroup as one of its arguments. Consequently, we force the caller to pass in a waitgroup, which might not always be necessary. This issue is a leaky abstraction provoking unwanted dependencies.

A better way to achieve the same result without affecting the function signature is:

```go
func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        func1()

        // Any cleanup code can be run here.
    }()
    wg.Wait()
}
```

By using an anonymous function, we encapsulate the WaitGroup handling without affecting function signatures. This way, we don't force the caller to pass in a waitgroup unnecessarily, and func1 doesn't need to be aware of our goroutine orchestration. It achieves the same goal but in a cleaner and more elegant way.

This small post serves as a documentation, so I can easily reference it whenever I come across the earlier mentioned pattern and suggest this improved approach.
