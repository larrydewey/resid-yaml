# resid-yaml

[YAML 1.2](https://yaml.org/spec/1.2.2/) as a format for
[resid-serial](../resid-serial), the serialization framework.

| File | What it is |
|---|---|
| `src/yaml_text.resid` | scalar rendering: what a string is written as, and what has to be quoted |
| `src/yaml_parse.resid` | the grammar, parsed into a tree: block and flow collections, every scalar style, anchors and aliases, tags, document streams and directives |
| `src/yaml_resolve.resid` | the core schema: what a plain scalar means, and what each tag says |
| `src/yaml.resid` | the `Encoder` and the `Decoder`, over the resolved tree |

## What a YAML value is

A **document** is one value of any kind: a mapping, a sequence, or a bare
scalar. A **stream** is one document or several separated by `---`, each
closed by `...` or by the next `---`; `%YAML` and `%TAG` directives stand
before the document they apply to. A **node** may carry an **anchor** and a
**tag** in front of it, in either order, and an **alias** anywhere a node
may start names the node its anchor holds, including inside a flow
collection.

Two consequences shape the code:

- Decoding resolves the whole document first and then navigates the tree, so
  a field may be asked for by name in any order and an anchor written later
  than the alias that uses it still reads. The resolver is where the core
  schema lives: a plain scalar is an integer, a float, a bool, a null or a
  string by what it looks like, and a tag says what it is whatever it looks
  like.
- Encoding writes block style, since that is what a person writes: a mapping
  is `key: value` per line, a sequence is `- ` per entry, and a value that
  is a collection of its own opens a block one step deeper. A mapping that
  starts on the line of a `- ` is written compactly there (`- name: a`,
  then `  n: 1` beside it), which is what the form means.

The document is written as a **rope** — a list of the pieces it was written
in, joined once at the end — rather than one string copied whole for every
value, and the parser gathers the lines of a scalar, key or flow collection
into a `StrBuf` rather than a fresh string per character.

## The data model

| Resid | YAML |
|---|---|
| a record, a map | a mapping, so a map's keys have to be strings |
| a sequence | a sequence, block or flow |
| `Str`, `Int`, `Float`, `Bool` | the values themselves |
| `Option(T)` | `Some(v)` is `v`; `None` is `null` |
| `Unit` | reads back from `null`, and is written as one, so `Some(unit)` would read back as `None`: refused in the writing direction |
| bytes, `Dec(N)` | no spelling; refused in both directions |
| `Int(128)`, `UInt(128)` | an integer while it fits in the 64 bits the tree holds; YAML's integers are unbounded, so one that does not fit is an error rather than a number that wrapped |
| a variant | a mapping with a `type` key: `{type: Circle, value: 3}`, and a unit variant is `{type: Dot}` |
| an untagged variant | the payload alone, tried in order, so `Num(5)` is `5` and `Word(w)` is `w` |
| a timestamp | a string, since the data model has no date: the text is kept as written and reads back the same |

Unlike a format whose document must be a mapping, a document here may be a
bare scalar: `to_yaml(5)` is `---\n5\n` and `from_yaml` reads it back. A
`!!binary` tag is refused rather than mapped to bytes, and a `!!seq` or
`!!map` tag on a scalar is an error rather than a guess.

## Entry points

```resid
import "src/yaml.resid";

Result(Str, SerialError) text = to_yaml(point) else { return 1; };
Result(Point, SerialError) back = from_yaml(text) else { return 1; };
```

A document can also be read by hand: `yaml_value(text)` resolves one
document into a `YValue`, `yaml_stream(text)` resolves every document of a
stream, `yaml_docs(text)` gives the parsed documents, `yaml_lines(text)`
splits the lines, `yaml_decoder(root)` gives a `Decoder` over a resolved
tree, and `yv_kind`, `yv_is_map`, `yv_str`, `yv_int` and `yv_pairs` read
out of it.

## What it refuses

Checked by `tests/errors.resid`, every case an error value with a kind and a
path rather than an abort: a mapping key with nothing in it, a tab used as
indentation, a quoted scalar, flow collection or anchor that is never
closed, a key set twice, a tag that is not one, a `%TAG` handle that was
never declared, an integer outside the 64 bits, a missing required field, a
field of the wrong type, a variant with no `type` key or an unknown one, and
reading past the end of the document.

## Known limits

- A mapping key is a scalar. A collection as a key is refused rather than
  written as its own text, so a complex key is not read.
- An alias is resolved by re-reading the line the anchor was written on, so
  an alias before its anchor reads (YAML forbids that order) and an anchor
  that would follow itself is an error rather than a recursive value.
- A multi-line quoted scalar is read one line at a time: a line break inside
  single quotes folds to a space, as a plain scalar's does.

## Tests

```
tests/run.sh            # build resid-derive, then every program
tests/run.sh --update   # rewrite the golden .out files
```

| Program | Checks | What it covers |
|---|---|---|
| `tests/vectors.resid` | 116 | the documents the specification writes, parsed and read back, and the text the encoder writes for them |
| `tests/roundtrip.resid` | 36 | values through the whole pipeline, with hand-written instances, so the test shows what one looks like, including a fixed-capacity string streamed a code point at a time |
| `tests/errors.resid` | 56 | what the format reports when the document is wrong |
| `tests/value.resid` | 49 | the format-neutral `Value` tree, which is where `read_peek` and `read_skip` are exercised |
| `tests/derive.resid` | 19 | instances generated by `resid-derive`, to show they need nothing from the format beyond the protocol |

## Requirements

A Resid compiler with the changes made alongside resid-serial (spec v3.8).
The framework lives in the sibling `../resid-serial` checkout.