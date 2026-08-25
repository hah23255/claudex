---
name: go-cli-prompts
description: Interactive input for Go CLI tools - when a prompt is the right channel at all, the utils prompt helpers, and the ErrNoTerminal contract that keeps every command scriptable. Use when a command needs to ask the user something, when building or changing utils/input.go, or when adding a password, free-text, or selection prompt. Triggers on PromptInput, PromptPassword, PromptTextArea, PromptSelect, PromptMultiSelect, ErrNoTerminal, StdinIsTerminal, bubbletea textinput, and textarea.
user-invocable: false
---

# Go CLI Prompts

**A prompt is a convenience for someone at a keyboard, never the only way to supply a value.**

Interactive input is the exception rather than the default channel. Reach for it only when the value is unreasonable to type on a command line, such as a choice among options the user has not seen yet, or a secret that would otherwise sit in shell history. Everything else is a flag.

## The Rule

Every prompt has a flag, or a positional argument, that supplies the same value and skips it. Without one the command is unusable from a script, a cron job, or an agent, and no amount of terminal polish makes up for that.

The prompt then fires only when the value is still missing:

```go
target := flags.account
if target == "" {
    idx, err := u.PromptSelect("Account", labels)
    // ...
}
```

## No Terminal

Every helper checks stdin before opening a bubbletea program and returns `ErrNoTerminal` when there is not one, so a piped or backgrounded invocation fails immediately instead of hanging on a read nobody will satisfy.

```go
var ErrNoTerminal = errors.New("no interactive terminal")

func PromptSelect(label string, options []string) (int, error) {
    if !StdinIsTerminal {
        return -1, ErrNoTerminal
    }
    // ...
}
```

The command decides what that means. A prompt with a documented default takes it; a prompt without one fails naming the flag that would have supplied the answer, which is the whole message the caller needs:

```go
picked, err := u.PromptMultiSelect("Presets", labels)
if errors.Is(err, u.ErrNoTerminal) {
    u.PrintFatal("apply-preset needs a preset name when there is no interactive terminal", nil)
}
if err != nil {
    u.PrintFatal("TUI error", err)
}
```

A command that cannot work without a terminal at all, because it hands the session to another program, says so once at the top of `Run` rather than at each prompt.

## The Helpers

| Helper | Behavior | Returns |
|---|---|---|
| `PromptInput(prompt, placeholder)` | single-line textinput | `(string, error)` |
| `PromptPassword(prompt)` | masked textinput | `(string, error)` |
| `PromptTextArea(prompt, placeholder)` | multi-line textarea, Ctrl+D submits | `(string, error)` |
| `PromptSelect(label, options)` | single-choice list | `(int, error)`, `-1` on cancel |
| `PromptMultiSelect(label, options)` | multi-choice list, space toggles | `(map[int]bool, error)`, `nil` on cancel |

Every cancel path is a clean no-op abort: `idx < 0` from `PromptSelect` and a `nil` map from `PromptMultiSelect` mean the user pressed Escape, and treating that as an empty selection would run the operation they just declined.

A password returned from `PromptPassword` never reaches a `Print` function, because the debug tier writes it into a log that outlives the session.

## Single-Line Input

One bubbletea model serves both `PromptInput` and `PromptPassword`; the password variant only sets `EchoMode`.

```go
type inputModel struct {
    textInput textinput.Model
    done      bool
    value     string
    initCmd   tea.Cmd
}

func (m inputModel) Init() tea.Cmd { return m.initCmd }

func (m inputModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "enter":
            m.value = m.textInput.Value()
            m.done = true
            return m, tea.Quit
        case "ctrl+c", "esc":
            m.done = true
            return m, tea.Quit
        }
    }
    m.textInput, cmd = m.textInput.Update(msg)
    return m, cmd
}

func (m inputModel) View() tea.View {
    if m.done {
        return tea.NewView("")
    }
    return tea.NewView(m.textInput.View())
}

func PromptInput(prompt, placeholder string) (string, error) {
    if !StdinIsTerminal {
        return "", ErrNoTerminal
    }
    ti := textinput.New()
    ti.Placeholder = placeholder
    ti.Prompt = prompt + " "
    m := inputModel{textInput: ti, initCmd: ti.Focus()}

    final, err := tea.NewProgram(m).Run()
    if err != nil {
        return "", err
    }
    return strings.TrimSpace(final.(inputModel).value), nil
}

func PromptPassword(prompt string) (string, error) {
    if !StdinIsTerminal {
        return "", ErrNoTerminal
    }
    ti := textinput.New()
    ti.Placeholder = "••••••••"
    ti.Prompt = prompt + " "
    ti.EchoMode = textinput.EchoPassword
    m := inputModel{textInput: ti, initCmd: ti.Focus()}

    final, err := tea.NewProgram(m).Run()
    if err != nil {
        return "", err
    }
    return final.(inputModel).value, nil
}
```

`PromptPassword` skips the `TrimSpace` that `PromptInput` applies, since a trailing space can be part of a password and silently removing it produces an authentication failure nobody can explain.

A secret also arrives from an environment variable or the config directory, and a prompt is the fallback for a first run rather than the only path.

## Multi-Line Input

`PromptTextArea` submits on Ctrl+D rather than Enter, because Enter has to stay available for the newlines that make the field multi-line.

```go
func (m textAreaModel) View() tea.View {
    if m.done {
        return tea.NewView("")
    }
    return tea.NewView(m.textarea.View() + "\n(Ctrl+D to submit, Esc to cancel)")
}

func PromptTextArea(prompt, placeholder string) (string, error) {
    if !StdinIsTerminal {
        return "", ErrNoTerminal
    }
    PrintInfo(prompt)

    ta := textarea.New()
    ta.Placeholder = placeholder
    m := textAreaModel{textarea: ta, initCmd: ta.Focus()}

    final, err := tea.NewProgram(m).Run()
    if err != nil {
        return "", err
    }
    return strings.TrimSpace(final.(textAreaModel).value), nil
}
```

The `Update` method mirrors `inputModel`, matching `"ctrl+d"` for submit instead of `"enter"`.

A body of text that a script would supply belongs in a `--file` flag rather than in this prompt, since a here-doc piped at a TUI is not input the program can read.

## Selection

Both selectors share one model that tracks a cursor and, for the multi variant, a set of toggled indices. Reusing the shared helpers rather than hand-rolling a bubbletea model per command is what keeps the key bindings identical everywhere: arrows or `j`/`k` to move, Enter to confirm, Escape or Ctrl+C to cancel, and space to toggle in the multi variant.

```go
type selectModel struct {
    label    string
    options  []string
    cursor   int
    chosen   map[int]bool // nil for single-choice
    multi    bool
    done     bool
    canceled bool
}

func (m selectModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    key, ok := msg.(tea.KeyPressMsg)
    if !ok {
        return m, nil
    }
    switch key.String() {
    case "up", "k":
        m.cursor = max(m.cursor-1, 0)
    case "down", "j":
        m.cursor = min(m.cursor+1, len(m.options)-1)
    case " ":
        if m.multi {
            m.chosen[m.cursor] = !m.chosen[m.cursor]
        }
    case "enter":
        m.done = true
        return m, tea.Quit
    case "esc", "ctrl+c":
        m.canceled = true
        return m, tea.Quit
    }
    return m, nil
}

// View renders the label, then one line per option: a "> " marker on the cursor
// row, and for the multi variant a "[x]"/"[ ]" box reflecting m.chosen.
```

```go
func PromptSelect(label string, options []string) (int, error) {
    if !StdinIsTerminal {
        return -1, ErrNoTerminal
    }
    final, err := tea.NewProgram(selectModel{label: label, options: options}).Run()
    if err != nil {
        return -1, err
    }
    m := final.(selectModel)
    if m.canceled {
        return -1, nil
    }
    return m.cursor, nil
}

func PromptMultiSelect(label string, options []string) (map[int]bool, error) {
    if !StdinIsTerminal {
        return nil, ErrNoTerminal
    }
    final, err := tea.NewProgram(selectModel{
        label: label, options: options, chosen: map[int]bool{}, multi: true,
    }).Run()
    if err != nil {
        return nil, err
    }
    m := final.(selectModel)
    if m.canceled {
        return nil, nil
    }
    return m.chosen, nil
}
```

The flag behind a selector names the option rather than its position, because a positional index shifts the moment the option list grows and a script written against it then picks the wrong one.
