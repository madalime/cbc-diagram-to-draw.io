# cbc-diagram-to-draw.io
Converts CbC diagrams imported from WebCorC into draw.io diagrams

## Converting a diagram

```
python main.py samples/*.json --drawio out              # writes CbC_<name>.drawio
python main.py samples/MaxElement.json --drawio         # into the current directory
```

Each diagram becomes one draw.io file, `CbC_MaxElement.drawio` for a diagram
named `MaxElement`: one 440×40 box per statement holding its
[triple](#the-gcl-style), and one arrow per refinement carrying the step's
title (`S₁: COMPOSITION`) as an edge label to its right.

## Inspecting a diagram

```
python main.py samples/MaxElement.json                 # readable tree
python main.py samples/*.json --style gcl              # guarded commands
python main.py samples/MaxElement.json --style json    # normalized JSON
python main.py samples/MaxElement.json -o out.txt      # write to a file
```

## Rendering styles

`--style` (alias `--format`) picks how the parsed diagrams are printed:

| Style  | Output                                                          |
| ------ | --------------------------------------------------------------- |
| `text` | indented tree of the whole diagram (default)                    |
| `gcl`  | the refinement steps as annotated guarded commands, in HTML     |
| `json` | the models as normalized JSON (`--indent` sets the indentation) |

### The `gcl` style

Every statement becomes one numbered refinement step: a title naming the
placeholder it refines and its type, then its guarded-command form between
pre- and postcondition.

| Type          | Body                                                   |
| ------------- | ------------------------------------------------------ |
| `ROOT`        | `{pre} S_1 {post}`                                     |
| `STATEMENT`   | `{pre} i := 0; {post}` (Java `=` becomes `:=`)         |
| `SKIP`        | `{pre} skip {post}`                                    |
| `COMPOSITION` | `{pre} S_a {intermediate} S_b {post}`                  |
| `SELECTION`   | `{pre} if Guard1 -> S_a elif Guard2 -> S_b fi {post}`  |
| `REPETITION`  | `{pre} do [invariant, variant] Guard -> S_a od {post}` |

`S_a`, `S_b` being the numbers of the nested statements.

Numbers are handed out level by level, left to right and top to bottom, so a
statement is numbered before the statements it refines into and the steps come
out in ascending order. The root refines nothing and stays plain `S`; every
other statement is `S<sub>n</sub>` starting at 1 — including the branches of a
selection, which use their global number rather than a local one.

Conditions are written with the mathematical connectives: `&&` becomes `∧` and
`||` becomes `∨`. All remaining text is HTML-escaped (`&lt;`, `&gt;`), the index
is a `<sub>` tag, and line breaks stay real newlines:

```html
S<sub>3</sub>: REPETITION
    {maxe(A,0,j,i) ∧ j != A.length} do [maxe(A,0,j,i), A.length - j] j != A.length -&gt; S<sub>6</sub> od {maxe(A,0,j,i)}

S<sub>7</sub>: SELECTION
    {maxe(A,0,j,i) ∧ j != A.length} if A[j] &gt; A[i] -&gt; S<sub>9</sub> elif A[j] &lt;= A[i] -&gt; S<sub>10</sub> fi {maxe(A,0,j+1,i)}

S<sub>9</sub>: STATEMENT
    {maxe(A,0,j,i) ∧ j != A.length ∧ A[j] &gt; A[i]} i := j; {maxe(A,0,j+1,i)}
```

Only assignments are rewritten to `:=`; `==`, `<=`, `>=`, `!=` and `!` are left
alone.

## Layout

| Path                        | Purpose                                                      |
| --------------------------- | ------------------------------------------------------------ |
| `main.py`                   | CLI: arguments, file loading, output                         |
| `parser/cbc_json_parser.py` | extracts the models from the WebCorC JSON                    |
| `models/`                   | one model per statement type, plus `Diagram` and `Condition` |
| `rendering/`                | one module per style, registered in `rendering/__init__.py`  |
| `drawio/`                   | builds and writes the `.drawio` files                        |

A new style is a module with a `render(diagrams, options)` function plus an
entry in `rendering.RENDERERS` — the CLI picks up its name automatically.

### The `drawio` module

| Path                 | Purpose                                                    |
| -------------------- | ---------------------------------------------------------- |
| `drawio/document.py` | `to_xml(diagram)` — the mxGraph XML: boxes, labels, edges  |
| `drawio/layout.py`   | `tree_layout(diagram)` — where each box goes               |
| `drawio/writer.py`   | `write(diagram, dir)` / `write_all(...)` — the named files |

The labels are the HTML of the `gcl` style, which is what draw.io wants in a
`html=1` cell: the triple goes into the box, the step title onto the arrow that
refines into it. Boxes are a fixed 440×40 and long triples overflow them.

The boxes get a fresh tree layout — one row per refinement level, leaves packed
left to right, parents centred above their children — because WebCorC's own
coordinates (kept as `Statement.position`) place its nodes about 250px apart.
Rows and columns are 40px apart and the drawing starts 40px into the page.

The page is a landscape A4 (1169×826) at `pageScale="1"`, and nothing is ever
scaled to it: boxes stay 440×40 and a tree wider than one page runs across
several. The layout is a starting point for the import — rearrange it by hand in
draw.io.

```python
import drawio
from parser import parse_file

drawio.write(parse_file("samples/MaxElement.json"), "out")  # -> out/CbC_MaxElement.drawio
```

## Extracted data

`Diagram`: `name`, `java_variables`, `global_conditions`, `root`. Its
`precondition`, `postcondition` and `statement` are shortcuts to the root's.

The contract written directly in the diagram's `content` is a statement of its
own type, `ROOT`, wrapping the statement that has to fulfil it. Every statement
carries `type`, `precondition`, `postcondition`, `name`, `id`, `position`; per
type additionally:

| Type          | Model              | Extra fields                                                    |
| ------------- | ------------------ | --------------------------------------------------------------- |
| `ROOT`        | `Root`             | `statement`                                                     |
| `STATEMENT`   | `ProgramStatement` | `program_statement`                                             |
| `SKIP`        | `Skip`             | –                                                               |
| `COMPOSITION` | `Composition`      | `intermediate_condition`, `first_statement`, `second_statement` |
| `REPETITION`  | `Repetition`       | `variant`, `invariant`, `guard`, `loop_statement`               |
| `SELECTION`   | `Selection`        | `guards`, `commands` (guard *i* belongs to command *i*)         |

## Using it as a library

```python
from parser import parse_file

diagram = parse_file("samples/MaxElement.json")
for statement in diagram.walk():          # the ROOT first, then depth first
    print(statement.type, statement.precondition, statement.postcondition)
```
