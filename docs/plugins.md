# Skipper plugins

Skipper may be extended with functionality not present in the core. Currently
you can add additional tracers and filters. These additions need to be built
as go plugin.

Note the warning from Go's plugin.go:
```go
    // The plugin support is currently incomplete, only supports Linux,
    // and has known bugs. Please report any issues.
```

## Plugin directories

Plugins are loaded from sub directories of the plugin directories. By default
the plugin directory is set to `./plugins` (i.e. relative to skipper's working
directory). An additional directory may be given with the `-plugindir=/path/to/dir`
option to skipper.

Any file with the suffix `.so` found below the plugin directories (also in sub
directories) is attempted to load without any arguments. When a plugin needs an
argument, this must be explicitly loaded and the arguments passed, e.g. with
`-filter-plugin geoip,db=/path/to/db`.

## Building a plugin

Each plugin should be built with
```bash
go build -buildmode=plugin -o example.so example.go
```
There are some pitfalls:
* packages which are shared between skipper and the plugin **must not** be in
  a `vendor/` directory, otherwise the plugin will fail to load
* plugins must be rebuilt when skipper is rebuilt

## Filter plugins

All plugins must have a function named "InitFilter" with the following signature

```go
func([]string) (filters.Spec, error)
````

The parameters passed are all arguments for the plugin, i.e. everything after the first
word from skipper's `-filter-plugin` parameter. E.g. when the `-filter-plugin` 
parameter is

```
"myfilter,datafile=/path/to/file,foo=bar"
```

the "myfilter" plugin will receive

```go
[]string{"datafile=/path/to/file", "foo=bar"}
```

as arguments.

The filter plugin implementation is responsible to parse the received arguments.

### Example filter plugin

An example "noop" plugin looks like

```go
package main

import (
	"github.com/zalando/skipper/filters"
)

type noopSpec struct{}

func InitFilter(opts []string) (filters.Spec, error) {
	return noopSpec{}, nil
}

func (s noopSpec) Name() string {
	return "noop"
}
func (s noopSpec) CreateFilter(config []interface{}) (filters.Filter, error) {
	return noopFilter{}, nil
}

type noopFilter struct{}

func (f noopFilter) Request(filters.FilterContext)  {}
func (f noopFilter) Response(filters.FilterContext) {}
```

## Predicate plugins

All plugins must have a function named "InitPredicate" with the following signature

```go
func([]string) (routing.PredicateSpec, error)
````

The parameters passed are all arguments for the plugin, i.e. everything after the first
word from skipper's `-predicate-plugin` parameter. E.g. when the `-predicate-plugin` 
parameter is

```
"mypred,datafile=/path/to/file,foo=bar"
```

the "mypred" plugin will receive

```go
[]string{"datafile=/path/to/file", "foo=bar"}
```

as arguments.

The predicate plugin implementation is responsible to parse the received arguments.

### Example predicate plugin

An example "MatchAll" plugin looks like

```go
package main

import (
	"github.com/zalando/skipper/filters"
)

type noopSpec struct{}

func InitPredicate(opts []string) (routing.PredicateSpec, error) {
	return noopSpec{}, nil
}

func (s noopSpec) Name() string {
	return "MatchAll"
}
func (s noopSpec) Create(config []interface{}) (routing.Predicate, error) {
	return noopPredicate{}, nil
}

type noopPredicate struct{}

func (p noopPredicate) Match(*http.Request) bool {
    return true
}
```


## OpenTracing plugins

The tracers, except for "noop", are built as Go Plugins. A tracing plugin can
be loaded with `-opentracing NAME` as parameter to skipper.

Implementations of OpenTracing API can be found in the
https://github.com/skipper-plugins/opentracing repository.

All plugins must have a function named "InitTracer" with the following signature

```go
func([]string) (opentracing.Tracer, error)
```

The parameters passed are all arguments for the plugin, i.e. everything after the first
word from skipper's -opentracing parameter. E.g. when the -opentracing parameter is
"mytracer foo=bar token=xxx somename=bla:3" the "mytracer" plugin will receive

   []string{"foo=bar", "token=xxx", "somename=bla:3"}

as arguments.

The tracer plugin implementation is responsible to parse the received arguments.

An example plugin looks like
```go
package main

import (
     basic "github.com/opentracing/basictracer-go"
     opentracing "github.com/opentracing/opentracing-go"
)

func InitTracer(opts []string) (opentracing.Tracer, error) {
     return basic.NewTracerWithOptions(basic.Options{
         Recorder:       basic.NewInMemoryRecorder(),
         ShouldSample:   func(traceID uint64) bool { return traceID%64 == 0 },
         MaxLogsPerSpan: 25,
     }), nil
}
```