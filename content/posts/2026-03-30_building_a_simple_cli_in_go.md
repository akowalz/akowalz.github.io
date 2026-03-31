---
title: Building a Simple CLI in Go
date: 2026-03-30
---

I've been reaching for Go a lot lately when I need a quick command-line tool. The standard library gives you almost everything you need, and the result is a single binary you can drop anywhere. Here's a walkthrough of building a small CLI that reads a JSON file and prints a filtered summary.

## Setting up the project

Nothing fancy — just `go mod init` and a single `main.go`:

```
mkdir jsonfilter && cd jsonfilter
go mod init github.com/akowalz/jsonfilter
```

## The data model

Let's say we're working with a list of events. Each event has a `name`, a `timestamp`, and a `level` that's one of `info`, `warn`, or `error`:

```go
type Event struct {
    Name      string    `json:"name"`
    Timestamp time.Time `json:"timestamp"`
    Level     string    `json:"level"`
}
```

We'll load these from a JSON file, so we need a function that reads the file and unmarshals it into a slice of `Event`. The `os.ReadFile` function makes this pretty clean:

```go
func loadEvents(path string) ([]Event, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading %s: %w", path, err)
    }

    var events []Event
    if err := json.Unmarshal(data, &events); err != nil {
        return nil, fmt.Errorf("parsing JSON: %w", err)
    }

    return events, nil
}
```

Note the `%w` verb in `fmt.Errorf` — this wraps the original error so callers can still use `errors.Is` or `errors.As` to inspect it. Small thing, but it matters when you're debugging.

## Filtering

The filter itself is straightforward. We take a `level` string and return a new slice:

```go
func filterByLevel(events []Event, level string) []Event {
    var result []Event
    for _, e := range events {
        if e.Level == level {
            result = append(result, e)
        }
    }
    return result
}
```

You could get clever with generics here — a generic `Filter[T]` function that takes a predicate — but for a small tool like this, the explicit version is easier to read and easier to change.

## Parsing flags

Go's `flag` package is minimal but sufficient. I always forget that `flag.Parse()` has to be called _after_ all flags are defined, which has bitten me more than once:

```go
func main() {
    path := flag.String("file", "", "path to JSON file")
    level := flag.String("level", "", "filter by level (info, warn, error)")
    flag.Parse()

    if *path == "" {
        fmt.Fprintln(os.Stderr, "missing required flag: -file")
        os.Exit(1)
    }

    events, err := loadEvents(*path)
    if err != nil {
        fmt.Fprintf(os.Stderr, "error: %v\n", err)
        os.Exit(1)
    }

    if *level != "" {
        events = filterByLevel(events, *level)
    }

    for _, e := range events {
        fmt.Printf("[%s] %s — %s\n", e.Level, e.Timestamp.Format(time.RFC3339), e.Name)
    }
}
```

One thing I like about this pattern: the `-level` flag is optional. If you omit it, you get everything. If you include it, you get just the events at that level. No subcommands, no config files, no YAML. Just `go run . -file events.json -level error` and you're done.

## Testing it

I usually throw a quick test file together by hand:

```json
[
  {
    "name": "server started",
    "timestamp": "2026-03-30T08:00:00Z",
    "level": "info"
  },
  {
    "name": "disk usage high",
    "timestamp": "2026-03-30T09:15:00Z",
    "level": "warn"
  },
  {
    "name": "connection refused",
    "timestamp": "2026-03-30T09:30:00Z",
    "level": "error"
  },
  {
    "name": "retry succeeded",
    "timestamp": "2026-03-30T09:31:00Z",
    "level": "info"
  }
]
```

Then run it:

```
$ go run . -file events.json
[info] 2026-03-30T08:00:00Z — server started
[warn] 2026-03-30T09:15:00Z — disk usage high
[error] 2026-03-30T09:30:00Z — connection refused
[info] 2026-03-30T09:31:00Z — retry succeeded

$ go run . -file events.json -level error
[error] 2026-03-30T09:30:00Z — connection refused
```

## Writing an actual test

Go's testing story is one of its best features. No framework needed — just a `_test.go` file and the `testing` package:

```go
func TestFilterByLevel(t *testing.T) {
    events := []Event{
        {Name: "a", Level: "info"},
        {Name: "b", Level: "error"},
        {Name: "c", Level: "info"},
    }

    got := filterByLevel(events, "info")

    if len(got) != 2 {
        t.Errorf("expected 2 events, got %d", len(got))
    }
    for _, e := range got {
        if e.Level != "info" {
            t.Errorf("expected level info, got %s", e.Level)
        }
    }
}
```

Run it with `go test ./...` and you're done. No mocking frameworks, no dependency injection containers, no test configuration files. Just code that calls other code and checks the result.

## Wrapping up

The whole thing is about 80 lines. It reads a file, filters some data, and prints output. There's nothing fancy about it, and that's the point. Not every tool needs to be a production service with graceful shutdown and structured logging. Sometimes `fmt.Printf` is all you need.
