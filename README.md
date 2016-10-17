# pseudoterm [![Build Status](http://img.shields.io/travis/henvic/pseudoterm/master.svg?style=flat)](https://travis-ci.org/henvic/pseudoterm) [![Coverage Status](https://coveralls.io/repos/henvic/pseudoterm/badge.svg)](https://coveralls.io/r/henvic/pseudoterm) [![codebeat badge](https://codebeat.co/badges/9f970e36-d039-434a-b0db-3c9471fab567)](https://codebeat.co/projects/github-com-henvic-pseudoterm) [![Go Report Card](https://goreportcard.com/badge/github.com/henvic/pseudoterm)](https://goreportcard.com/report/github.com/henvic/pseudoterm) [![GoDoc](https://godoc.org/github.com/henvic/pseudoterm?status.svg)](https://godoc.org/github.com/henvic/pseudoterm)

This framework allows you to create programs using Go to run an interactive program programmatically in an easy way.

This can be useful both for automation and running integration / functional tests that for CLI tools with prompts intended for human beings.

[![asciicast](https://asciinema.org/a/02xhla8u9nvifz6x6q728ymv5.png)](https://asciinema.org/a/02xhla8u9nvifz6x6q728ymv5)

See [example/main.go](https://github.com/henvic/pseudoterm/blob/master/example/main.go) to see the code for this example.

## Story interface

You can either use the accompanying QueueStory or implement the Story interface on a new type.

```go
// Story is interface you can implement to handle commands
type Story interface {
	Setup() (ctx context.Context, err error)
	Teardown()
	TickHandler() (err error)
	HandleLine(s string) (in string, err error)
}
```

1. `Setup()` is called before the story starts running on the terminal.
2. `Teardown()` is called to setup any required clean up, after the story ends.
3. `TickHandler()` is called every time pseudoterm checks if there is new input on the pseudo [tty](https://en.wikipedia.org/wiki/TTY). This happens between LineReaderInterval, and you probably don't want or need to change its default value.
4. `HandleLine()` is called just after TickHandler() when there is a new line to be processed available. If there is not, it is not called.

## Terminal

```go
// Terminal is a pseudo terminal you can use to run commands
// on a pseudo tty programmatically
type Terminal struct {
	Command         *exec.Cmd
	EchoStream      io.Writer
	CopyStreamError error
}
```

If you want to print to standard output, set EchoStream to `os.Stdout`. You most likely can leave CopyStreamError alone (it is more useful for debugging, and it is quite problematic to deal with it and different platforms).

If you need to copy the output both to stdout and somewhere else, you might want to use `io.TeeReader`.

Terminal methods you need to know about:

* `t.Run(story Story) (err error)`
* `t.Wait() (ps *os.ProcessState)`
* `t.WriteLine(s string) (n int, err error)`

There are others. Read the code and tests, if you need more power. You can also execute a program without implementing a story, though generally you don't want to do that. See examples on the test files for that.

## QueueStory
QueueStory is a built-in sequential story type you can use directly for most applications of pseudoterm.

```go
// QueueStory is a command execution story with sequential steps 
// that must be fulfilled before the next is executed
type QueueStory struct {
	Sequence      []Step
	Timeout   time.Duration
}
```

QueueStory's Timeout is a global timeout for the story.
QueueStory methods you need to know about:

* `q.Add(args ...Step)` adds multiple steps at once
* `q.Success() bool` returns if the story was run successfully or not
* `q.Cancel()` is used to cancel a story


### QueueStory Step{}

```go
// Step is like a route rule to handle lines
type Step struct {
	Read       string
	Write      string
	SkipWrite  bool
	Timeout    time.Duration
}
```

Each step has a string it waits to read, a string it writes when the read operation happens (or skip triggered by a SkipWrite: true value), and a timeout.

When not defined *(same as setting to `time.Duration(0)`)*, the lack of timeout means the story or the step never times out.

It is highly recommended for all stories to time out after some time (ignore this if your programs should never exit, though this library is not tested for this use case).

## Special error values for line handling
terminal.HandleLine can return two special error values:
1. `SkipWrite` is used as a return value from Story HandleLine to indicate that a line should not be written when reading a line on a given step. Useful when you want to verify if a line was printed on the terminal, but you don't need to write a line in response.
2. `SkipZeroMatches` is used as a return value from Story HandleLine to indicate that there are no more steps left to be dealt with.

## Dependencies
This framework relies on [kr/pty](https://github.com/kr/pty) and should work on any operating system where it works (Windows is not on the list). Most of the hard work is done there. This provides a high-level API.

## Contributing
In lieu of a formal style guide, take care to maintain the existing coding style. Add unit tests for any new or changed functionality. Integration tests should be written as well.

The master branch of this repository on GitHub is protected:
* force-push is disabled
* tests MUST pass on Travis before merging changes to master
* branches MUST be up to date with master before merging

Keep your commits neat. Try to always rebase your changes before publishing them.

[goreportcard](https://goreportcard.com/report/github.com/wedeploy/cli) can be used online or locally to detect defects and static analysis results from tools such as go vet, go lint, gocyclo, and more. Run [errcheck](https://github.com/kisielk/errcheck) to fix ignored error returns.

Please ignore the warnings about SkipWrite and SkipZeroMatches on golint. This is akin of `filepath.SkipDir` to make it easier for people to understand.

Using go test and go cover are essential to make sure your code is covered with unit tests.