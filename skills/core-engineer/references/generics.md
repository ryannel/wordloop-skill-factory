# Generics

## Table of Contents
- [Type Constraints](#type-constraints)
- [Approximate Constraints](#approximate-constraints)
- [Generic Data Structures](#generic-data-structures)
- [Generic Collection Utilities](#generic-collection-utilities)
- [Generic Channel Operations](#generic-channel-operations)
- [Recursive Generic Constraints](#recursive-generic-constraints)

---

Use generics to eliminate boilerplate in collection operations and data structures. Prefer concrete types for domain entities — generics excel for utility code, not business logic.

## Type Constraints

```go
// Custom constraint using union types
type Number interface {
    constraints.Integer | constraints.Float
}

func Sum[T Number](numbers []T) T {
    var total T
    for _, n := range numbers {
        total += n
    }
    return total
}
```

## Approximate Constraints

Use `~` to accept named types derived from base types:

```go
type Integer interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

type MeetingCount int

func Double[T Integer](n T) T {
    return n * 2
}

// Works with both int and MeetingCount
count := Double(MeetingCount(5))
```

## Generic Data Structures

```go
// Type-safe stack
type Stack[T any] struct {
    items []T
}

func NewStack[T any]() *Stack[T] {
    return &Stack[T]{items: make([]T, 0)}
}

func (s *Stack[T]) Push(item T)       { s.items = append(s.items, item) }
func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}
```

## Generic Collection Utilities

```go
// Filter elements matching a predicate
func Filter[T any](slice []T, predicate func(T) bool) []T {
    result := make([]T, 0, len(slice))
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

// Map transforms a slice from one type to another
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Keys extracts keys from a map
func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

// Contains checks if a slice contains a specific value
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}
```

## Generic Channel Operations

```go
// Merge multiple channels
func Merge[T any](channels ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan T) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}

// Stage creates a typed pipeline step
func Stage[T, U any](in <-chan T, fn func(T) U) <-chan U {
    out := make(chan U)
    go func() {
        defer close(out)
        for v := range in {
            out <- fn(v)
        }
    }()
    return out
}
```

## Recursive Generic Constraints

F-bounded polymorphism allows a generic type to reference itself in its own constraint. This enables method signatures where a type must return or accept its own concrete type:

```go
// Constraint: T must implement Mapper that returns T itself
type Mapper[T any] interface {
    Map(func(T) T) T
}

// Builder pattern with method chaining and type safety
type Builder[T any] interface {
    WithField(name string, value any) T
    Build() (T, error)
}
```

This pattern ensures compile-time safety for builder patterns, mathematical types, and recursive data structures — no reflection or type assertions needed.
