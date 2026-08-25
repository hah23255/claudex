---
name: go-cli-commands
description: Cobra command wiring for Go CLI tools - root command, simple commands, subcommand packages, the three input channels, and flag conventions. Use when scaffolding main.go or cmd/root.go, adding a command or subcommand, registering flags, deciding whether a value should be a flag or a prompt, or setting up --debug. Triggers on cobra.Command, rootCmd, AddCommand, PersistentFlags, BoolVar, StringVarP, MarkFlagRequired, MarkFlagsMutuallyExclusive, cmd/ files, and AppVersion ldflags injection.
user-invocable: false
---

# Go CLI Commands

**How a Cobra command tree is wired: the root, the commands hanging off it, and the flags that feed them.**

CLI Only projects and Web Only projects use different roots, and a CLI + Web hybrid uses the CLI Only root plus a `serve` command.

| Aspect | CLI Only | Web Only |
|---|---|---|
| Imports | zerolog, utils, subcommand packages | zerolog, cobra, subcommand packages |
| Global flags | `--debug` | `--debug` |
| Logging setup | `setupLogs()` via `cobra.OnInitialize`, terminal check through `utils` | `setupLogs()` via `cobra.OnInitialize`, terminal check inline |
| Output | `utils.Print*` | zerolog only |

Both set `Use`, `Short`, `Version` from the `AppVersion` ldflag, and `CompletionOptions.HiddenDefaultCmd: true`.

## Entry Point

```go
package main

import "github.com/[GITHUB_USER]/REPO_NAME/cmd"

func main() {
    cmd.Execute()
}
```

`main.go` holds nothing else, so no logic ends up in the one function a test cannot call.

## Root Command, CLI Only

```go
package cmd

import (
    "fmt"
    "os"
    "time"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
    "github.com/spf13/cobra"
    "github.com/[GITHUB_USER]/REPO_NAME/utils"

    featureCmd "github.com/[GITHUB_USER]/REPO_NAME/cmd/feature-cmd"
)

var AppVersion = "dev-build" // set at build time via ldflags

var debugFlag bool

var rootCmd = &cobra.Command{
    Use:               "appname",
    Short:             "Brief description of the application",
    Version:           AppVersion,
    CompletionOptions: cobra.CompletionOptions{HiddenDefaultCmd: true},
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func setupLogs() {
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnix
    output := zerolog.ConsoleWriter{Out: os.Stdout, TimeFormat: time.DateTime, NoColor: !utils.StdoutIsTerminal}
    log.Logger = zerolog.New(output).With().Timestamp().Logger()
    zerolog.SetGlobalLevel(zerolog.InfoLevel)
    if debugFlag {
        zerolog.SetGlobalLevel(zerolog.DebugLevel)
        utils.GlobalDebugFlag = true
    }
}

func init() {
    rootCmd.SetHelpCommand(&cobra.Command{Hidden: true})

    rootCmd.PersistentFlags().BoolVar(&debugFlag, "debug", false, "Enable debug logging")

    cobra.OnInitialize(setupLogs)

    rootCmd.AddCommand(serveCmd)
    rootCmd.AddCommand(featureCmd.FeatureCmd)
}
```

`setupLogs` runs through `cobra.OnInitialize` rather than at package init, because the flag values it reads are not parsed until Cobra has matched the command.

The help and completion commands are hidden so `appname --help` lists only the commands the tool actually offers.

## Root Command, Web Only

```go
package cmd

import (
    "fmt"
    "io"
    "os"
    "time"

    "github.com/charmbracelet/x/term"
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
    "github.com/spf13/cobra"
)

var AppVersion = "dev-build"

var debugFlag bool

var rootCmd = &cobra.Command{
    Use:               "appname",
    Short:             "Brief description of the application",
    Version:           AppVersion,
    CompletionOptions: cobra.CompletionOptions{HiddenDefaultCmd: true},
}

func Execute() {
    if err := rootCmd.Execute(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func setupLogs() {
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnix
    var out io.Writer = os.Stdout
    if term.IsTerminal(os.Stdout.Fd()) {
        out = zerolog.ConsoleWriter{Out: os.Stdout, TimeFormat: time.DateTime}
    }
    log.Logger = zerolog.New(out).With().Timestamp().Logger()
    zerolog.SetGlobalLevel(zerolog.InfoLevel)
    if debugFlag {
        zerolog.SetGlobalLevel(zerolog.DebugLevel)
    }
}

func init() {
    rootCmd.SetHelpCommand(&cobra.Command{Hidden: true})
    rootCmd.PersistentFlags().BoolVar(&debugFlag, "debug", false, "Enable debug logging")
    cobra.OnInitialize(setupLogs)
    rootCmd.AddCommand(serveCmd)
}
```

Same logger and same `--debug` as CLI Only, without the `utils` import. What a server has no use for is the printers, since nobody is watching a container's stdout for a styled checkmark.

## Simple Command

A command without subcommands is defined directly in `cmd/`, with its own flag struct and `init()`. Grouping flags in a struct per command keeps two commands from colliding over a variable named `output` in the shared `cmd` package.

```go
// cmd/serve.go
package cmd

import (
    "fmt"

    "github.com/spf13/cobra"
    "github.com/[GITHUB_USER]/REPO_NAME/internal/server"
    u "github.com/[GITHUB_USER]/REPO_NAME/utils"
)

var serveFlags struct {
    port int
    host string
}

var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Start the web server",
    Run: func(cmd *cobra.Command, args []string) {
        srv := server.New(serveFlags.host, serveFlags.port)
        if err := srv.Setup(); err != nil {
            u.PrintFatal("Failed to set up server", err)
        }
        u.PrintInfo(fmt.Sprintf("Starting server on %s:%d", serveFlags.host, serveFlags.port))
        if err := srv.Run(); err != nil {
            u.PrintFatal("Server error", err)
        }
    },
}

func init() {
    serveCmd.Flags().IntVarP(&serveFlags.port, "port", "p", 8080, "Port to listen on")
    serveCmd.Flags().StringVarP(&serveFlags.host, "host", "H", "0.0.0.0", "Host to bind to")
}
```

The same command in a Web Only project swaps `u.PrintFatal(msg, err)` for `log.Fatal().Err(err).Msg(msg)` and `u.PrintInfo` for `log.Info()`, and drops the `utils` import.

## Run Function Shape

A `Run` body validates flags, builds a config struct, calls into `internal/`, and reports the result. Keeping the work in `internal/` leaves the command as the only layer that knows about flags and terminals, which is what lets the logic be tested without one.

```go
Run: func(cmd *cobra.Command, args []string) {
    if flags.required == "" {
        u.PrintFatal("--required flag is required", nil)
    }

    cfg := internal.Config{Field1: flags.field1, Field2: flags.field2}

    result, err := internal.DoThing(cfg)
    if err != nil {
        u.PrintFatal("Failed to do thing", err)
    }

    u.PrintSuccess("Thing completed")
    u.PrintGeneric(result)
}
```

## Subcommand Package

A group of related subcommands gets a package under `cmd/`. The parent command is exported and has no `Run`, so invoking it bare prints help instead of doing something arbitrary; the children are unexported and each carry a `Run`.

```go
// cmd/feature-cmd/feature.go
package featureCmd

import (
    "fmt"

    "github.com/spf13/cobra"
    "github.com/[GITHUB_USER]/REPO_NAME/internal/feature"
    u "github.com/[GITHUB_USER]/REPO_NAME/utils"
)

var createFlags struct {
    name   string
    config string
}

var FeatureCmd = &cobra.Command{
    Use:   "feature",
    Short: "Feature management commands",
}

var createCmd = &cobra.Command{
    Use:   "create",
    Short: "Create a new feature",
    Run: func(cmd *cobra.Command, args []string) {
        cfg := feature.CreateConfig{Name: createFlags.name, Config: createFlags.config}
        if err := feature.Create(cfg); err != nil {
            u.PrintFatal("Failed to create feature", err)
        }
        u.PrintSuccess(fmt.Sprintf("Created feature: %s", createFlags.name))
    },
}

func init() {
    FeatureCmd.AddCommand(createCmd)

    createCmd.Flags().StringVarP(&createFlags.name, "name", "n", "", "Feature name (required)")
    createCmd.MarkFlagRequired("name")
    createCmd.Flags().StringVarP(&createFlags.config, "config", "c", "", "Config file path")
}
```

## Input Channels

Three channels carry a value into a command, and which one a given value uses is a decision rather than a preference.

| Channel | Carries | Where it lives |
|---|---|---|
| Config and environment | credentials, endpoints, anything set once and reused | `~/.config/[APP_NAME]/`, and `[APP_NAME]_*` variables |
| Flags | everything else a single run needs | `init()` on the command that reads them |
| Prompts | a choice among options the user has not seen yet, or a secret that would land in shell history | a `utils` prompt helper |

Flags are the default channel. A prompt is added only when a flag cannot reasonably carry the value, and it always has a flag beside it supplying the same answer, so the command still runs from a script.

Precedence, highest first: an explicit flag, then the environment, then the config file, then the built-in default. The flag wins because it is the most specific thing the caller said in this invocation. A value piped in belongs to the flag tier rather than to a tier of its own, since it arrives as that flag's value.

## Flags

Flags are registered in `init()`, one call per flag, next to the command they belong to.

A boolean flag takes a long name and no shorthand. A single letter standing for a switch abbreviates nothing the reader can recover, and it collides with the next switch somebody adds. A flag that takes a value may have a shorthand, because the value sitting beside it already says what it is.

```go
cmd.Flags().BoolVar(&flags.all, "all", false, "Include all items")
cmd.Flags().BoolVar(&flags.force, "force", false, "Overwrite an existing file")
cmd.Flags().StringVarP(&flags.name, "name", "n", "default", "Description")
cmd.Flags().IntVarP(&flags.count, "count", "c", 10, "Number of items")
cmd.Flags().StringSliceVarP(&flags.tags, "tag", "t", []string{}, "Tags (repeatable)")
cmd.Flags().DurationVarP(&flags.timeout, "timeout", "T", 30*time.Second, "Request timeout")
```

A flag registers at the level that reads it. `PersistentFlags` on the root is for what the whole tree honors, which in practice is `--debug` and nothing else; `PersistentFlags` on a parent command covers that group; everything else is `Flags()` on the command itself. A flag registered a level too high appears in the help of every command that ignores it.

`MarkFlagRequired` states a requirement Cobra enforces before `Run` is reached, which produces a usage message rather than a nil dereference:

```go
cmd.Flags().StringVarP(&flags.input, "input", "i", "", "Input file (required)")
cmd.MarkFlagRequired("input")
```

An environment variable supplies a default rather than being read inside `Run`, which is what puts the flag above it in precedence and makes `--help` show the value the command will actually use. A variable the tool owns is namespaced with the tool's name; one belonging to another tool keeps that tool's name.

```go
defaultToken := os.Getenv("GITHUB_TOKEN")
cmd.Flags().StringVarP(&flags.token, "token", "t", defaultToken, "GitHub token (or GITHUB_TOKEN env)")
```

`MarkFlagsMutuallyExclusive` rejects contradictory combinations at parse time, which is where the user can still fix them:

```go
cmd.MarkFlagsMutuallyExclusive("file", "url")
```

## Values from stdin

A flag given the value `-` reads that value from stdin. The flag keeps the name of the thing it carries, so a command taking two of them stays unambiguous and the invocation says where each value came from.

```
pwmgr add github --password -        # from the pipe
pwmgr add github --password hunter2  # inline, and now in shell history
pwmgr add github                     # prompts, and needs a terminal
```

```bash
echo "hunter2" | pwmgr add github --password -
```

A command reads stdin only when a flag said `-`, and never because stdin happens not to be a terminal. Inferring it means a run with stdin on `/dev/null`, closed, or inherited from a scheduler stores an empty value and reports success, which is the failure nobody notices until they need the value back. The sentinel makes that impossible rather than unlikely.

Two flags cannot both be `-` in one invocation, since there is one stream and no way to say where the first value ends. Reject the combination rather than splitting on a blank line.

An inline secret is a convenience that leaks: it lands in shell history and is visible in `ps` output for the life of the process. `-` is the path a script should take, and the help text for a secret-bearing flag says so.
