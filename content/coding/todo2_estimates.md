---
title: "Adding Estimates to DevTodo2: A Journey in Go and Hierarchical Tasks"
date: 2026-07-15T00:00:00+00:00
draft: false
categories:
  - Coding
  - Go
  - Open Source
tags:
  - cli
  - task-management
  - hierarchical-data
  - golang
description: Adding hierarchical time estimates and automatic parent summaries to a CLI task manager using Go.
series: "Developer Tools"
slug: adding-estimates-to-devtodo2-go-hierarchical-tasks
---

# Adding Estimates to DevTodo2: A Journey in Go and Hierarchical Tasks

## Intro

Some time ago we went on adapting `devtodo2` as an internal tool to allow task estimates and parent summaries. The core appeal of `devtodo2` is its simplicity: it's a command-line utility where task lists are stored as a simple `.todo2` file right in your project folder. This makes it incredibly handy to have a project's todo list and completion history live alongside the code, effectively mitigating the "forgetting what's been done" (or the Zeigarnik effect)—where unfinished tasks clutter the mind, but completed ones vanish from memory.

## The Adventure in Go

We forked the original repository and began the process of extending it. The tool was written in Go, a language we had to learn on the fly for this adventure. For those unfamiliar, Go's approach to concurrency and its strict typing made it a refreshing choice for a CLI tool, though the initial learning curve was part of the challenge.

Adding estimates in a new column was the easy part. We implemented a flexible time-parsing system that allows users to specify durations in a human-readable format:
- `h` for hours
- `d` for days
- `w` for weeks
- `M` for months
- `y` for years

Here is how we handled the parsing in Go, using a simple loop to separate the numeric value from the unit:

```go
func ParseEstimate(estimate string) (time.Duration, error) {
    // ... (trimming and validation)
    for i, char := range estimate {
        if !unicode.IsDigit(char) && char != '.' && char != '-' && char != '+' {
            numStr = estimate[:i]
            unitStr = estimate[i:]
            break
        }
    }

    num, err := strconv.ParseFloat(numStr, 64)
    // ...
    switch strings.ToLower(unitStr) {
    case "h", "hr", "hrs", "hour", "hours":
        duration = time.Duration(num * float64(time.Hour))
    case "d", "day", "days":
        duration = time.Duration(num * float64(24*time.Hour))
    case "w", "wk", "wks", "week", "weeks":
        duration = time.Duration(num * float64(7*24*time.Hour))
    // ...
    }
    return duration, nil
}
```

Implementing `ParseEstimate` and `FormatEstimate` allowed us to treat these as `time.Duration` objects, making the internal math straightforward.

## The "Stretch": Hierarchical Summation

The real challenge began when we wanted parent tasks to automatically summarize the estimates of their descendants. This is where the project shifted from a simple addition to a structural "stretch."

The goal was to make the tool ignore "unspecified estimates" while accurately summing up the total weight of a task tree from the leaf nodes up to the root. This required a recursive traversal of the task hierarchy.

The implementation of `SumDescendants` demonstrates the simplicity of the recursive approach once the `TaskNode` interface is properly defined:

```go
func (t *taskNodeImpl) SumDescendants() time.Duration {
    var sum = time.Duration(0.0)
    for i := 0; i < t.Len(); i++ {
        sum += t.At(i).Estimate()
    }
    return sum
}
```

One of the biggest hurdles we encountered was data persistence. We initially struggled with how to store these summed totals. The solution, discovered through some trial and error (and a few "dirty quick fixes"), was to move the summation logic to the loading phase. By recomputing the descendant estimates whenever the task list is loaded into memory, we ensure a single source of truth—the leaf nodes—while still providing instant summaries for the parents.

In `main.go`, this is as simple as updating the root estimate immediately after loading the list:

```go
func main() {
    // ... load task list
    tasks, err := loadTaskList()
    // ...
    tasks.SetEstimate(tasks.SumDescendants())
    processAction(tasks)
}
```

However, for individual actions like adding, editing, or reparenting tasks, we need to propagate these changes up the tree. We implemented `updateParentsEstimates`, which climbs the hierarchy from the modified task up to the root, recalculating the sums along the way:

```go
func updateParentsEstimates(tasks TaskList, index string, pop bool) {
    // ... validation
    for {
        path = path[:len(path)-boolToInt(pop)]
        pop = true
        if len(path) == 0 || len(index) == 0 {
            break
        }
        var index = strings.Join(path, ".")
        var parent = resolveTaskReference(tasks, index)
        parent.SetEstimate(parent.SumDescendants())
    }
}
```

This ensures that if you change a task's estimate at depth 4, its parent, grandparent, and the project root are all updated in one operation.

## The Edge Cases: Reparenting and Roots

As with any tree-based structure, reparenting tasks proved to be the ultimate test. Moving a task from one branch to another isn't just a pointer change; it triggers a cascade of recalculations. If you move a task with a 10-hour estimate, every parent in the old branch must decrease and every parent in the new branch must increase.

At a high level, reparenting is a simple "delete and append" operation, but the underlying `Delete` method must carefully handle Go slices to remove the node from its current parent:

```go
func ReparentTask(node TaskNode, below TaskNode) {
    node.Delete()
    below.Append(node)
}

func (t *taskNodeImpl) Delete() {
    parent := t.Parent().(*taskNodeImpl)
    if parent == nil {
        panic("can not delete root node")
    }
    for i := 0; i < parent.Len(); i++ {
        if parent.At(i).Equal(t) {
            // Remove the task from the parent's slice
            parent.tasks = append(parent.tasks[:i], parent.tasks[i+1:]...)
            t.parent = nil
            return
        }
    }
    panic("couldn't find t in parent in order to delete")
}
```

We spent significant time "sanitizing" the reparenting logic to handle edge cases, such as moving a task back to the top level (the root). We also had to squash several bugs where grafting a task directly to the root or initializing a list in an empty directory would cause the indexer to complain about "invalid index 0."

Finally, we polished the UX. We realized that `1m` was ambiguous—did it mean one minute or one month? We refined the DSL so that `1m` refers to a minute, while `1M` refers to a month, ensuring the tool remained intuitive for long-term project planning.

## Conclusion

What started as a simple request for "estimates" turned into a deep dive into tree traversal, Go's type system, and the nuances of CLI UX. The result is a tool that not only tracks what needs to be done but provides a clear, aggregated view of the project's remaining effort, all while keeping the history right where it belongs: in the project folder.
