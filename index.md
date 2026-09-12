---
layout: default
---

[![Gem Version](https://badge.fury.io/rb/zlight_csv.svg)](https://rubygems.org/gems/zlight_csv)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Source](https://img.shields.io/badge/source-GitHub-181717.svg)](https://github.com/codebyisaad/zlight)

Ruby's built-in CSV library is written in Ruby. ZLight does the parsing in Rust
instead, and hands you back ordinary Ruby Hashes and Arrays.

```ruby
ZLight.parse("name,age\nAlice,30")
# => [{ name: "Alice", age: "30" }]
```

That is the whole idea. Same shape of result, considerably less waiting.

**Jump to:** [Install](#install) · [Quick start](#quick-start) ·
[Reading](#reading-csv) · [Large files](#large-files) · [Writing](#writing-csv) ·
[Converters](#converters) · [Coming from CSV](#coming-from-rubys-csv) ·
[Errors](#errors) · [Threads](#threads-and-ractors) ·
[Compatibility](#compatibility)

---

## Install

```ruby
gem 'zlight_csv'
```

No Rust toolchain needed. Precompiled binaries ship for Linux, macOS and
Windows, across Ruby 3.1 to 4.0. If you are on a platform without one, the gem
compiles from source instead — that path needs Rust.

---

## Quick start

Five things cover most of what people do with this gem.

```ruby
require 'zlight_csv'

# 1. Parse a string
ZLight.parse("name,age\nAlice,30")
# => [{ name: "Alice", age: "30" }]

# 2. Parse a file
ZLight.read("users.csv")

# 3. Numbers as numbers, not strings
ZLight.read("users.csv", converters: :numeric)
# => [{ name: "Alice", age: 30 }]

# 4. Stream a file too big for memory
ZLight.open("huge.csv") do |reader|
  reader.each { |row| process(row) }
end

# 5. Write one out
ZLight.write("out.csv", [{ name: "Alice", age: 30 }])
```

Rows are **Hashes with Symbol keys** by default. Pass `headers: false` and you
get Arrays instead.

---

## Reading CSV

| Method | Takes | Returns |
|---|---|---|
| `ZLight.parse(string, **opts)` | a CSV string | Array of rows |
| `ZLight.read(path, **opts)` | a file path | Array of rows |
| `ZLight.foreach(string, **opts)` | a CSV string | Enumerator, or yields each row |

```ruby
ZLight.parse("name,age\nAlice,30\nBob,25")
# => [{ name: "Alice", age: "30" }, { name: "Bob", age: "25" }]

ZLight.parse("Alice,30\nBob,25", headers: false)
# => [["Alice", "30"], ["Bob", "25"]]

ZLight.read("data.tsv", col_sep: "\t")

ZLight.foreach(csv_string) { |row| puts row[:name] }
ZLight.foreach(csv_string).map { |row| row[:name] }   # no block → Enumerator
```

All three read the entire input into memory. For anything large, see
[Large files](#large-files).

### Options

| Option | Default | What it does |
|---|---|---|
| `headers` | `true` | First row becomes Symbol keys. `false` gives Arrays. |
| `converters` | `nil` | `:numeric`, a callable, or an Array of them. See [Converters](#converters). |
| `col_sep` | `","` | Column separator. One byte — `"\t"` for TSV, `";"` for European CSV. |
| `quote_char` | `'"'` | Quote character. One byte. |
| `flexible` | `true` | Allow rows whose field count differs from the header. |

`col_sep` and `quote_char` must be exactly one byte. A longer string raises
`ArgumentError` rather than being silently truncated.

---

## Large files

`parse` and `read` build one Ruby object per field up front. For a file that
does not comfortably fit in memory, stream it instead: rows are read one at a
time and the memory used stays flat no matter how big the file is.

```ruby
ZLight.open("huge.csv") do |reader|
  reader.each { |row| process(row) }
end
```

`ZLight.open` closes the reader when the block exits, including if it raises.
That is the form to reach for.

### Stopping early

A stream reads only as far as you ask it to, so stopping early genuinely costs
nothing for the rest of the file:

```ruby
ZLight.open("huge.csv", converters: :numeric) do |reader|
  top = reader.lazy.select { |row| row[:score] > 90 }.first(100)
end
```

On a 100,000-row file, taking the first 100 matches this way finishes in well
under a millisecond, because the other 99,000 rows are never read.

### The reader itself

`ZLight.open` hands you a `ZLight::StreamReader`. You can also get one directly
from `ZLight.stream(string)` or `ZLight.stream_file(path)`, in which case
**closing it is your job**.

| Method | Returns |
|---|---|
| `#each` | yields each row; an Enumerator with no block |
| `#next_row` | the next row, or `nil` once exhausted |
| `#headers` | Array of Symbols, or `nil` when `headers: false` |
| `#close` | closes it; safe to call twice |
| `#closed?` | whether `close` has been called |
| `#eof?` | whether it is exhausted or closed |

It includes `Enumerable`, so `map`, `select`, `find` and `lazy` all work.

> **A reader is single-pass.** Each row is consumed as it is read, so a second
> `each` yields nothing and there is no rewind. Call `ZLight.stream` again to
> read the file over.

```ruby
reader = ZLight.stream_file("data.csv", converters: :numeric)

while (row = reader.next_row)
  break if row[:id] > 1000
  process(row)
end

reader.close
```

`stream_file` opens the file immediately, so a missing or unreadable path
raises there rather than later on the first row.

---

## Writing CSV

| Method | Does | Returns |
|---|---|---|
| `ZLight.generate(rows, **opts)` | builds a CSV string | String |
| `ZLight.write(path, rows, **opts)` | writes a CSV file | bytes written |

```ruby
ZLight.generate([{ name: "Alice", age: 30 }, { name: "Bob", age: 25 }])
# => "name,age\nAlice,30\nBob,25\n"

ZLight.generate([["Alice", 30], ["Bob", 25]])
# => "Alice,30\nBob,25\n"

ZLight.write("out.csv", rows)
ZLight.write("out.csv", rows, col_sep: ";", force_quotes: true)
```

Give it an Array of Hashes **or** an Array of Arrays — not a mix. Arrays have
no header row to write; Hashes get one unless you pass `headers: false`.

For Hashes, **column order comes from the keys of the first row**. A key that
only appears later is not written; a key missing from a later row becomes an
empty field.

### Options

| Option | Default | What it does |
|---|---|---|
| `headers` | `true` | Write a header row. Only applies to Hash input. |
| `col_sep` | `","` | Column separator. One byte. |
| `quote_char` | `'"'` | Quote character. One byte. |
| `force_quotes` | `false` | Quote every field, not just the ones that need it. |

Both build the whole string in memory before writing, so they are not the tool
for output larger than RAM.

---

## Converters

`converters:` decides what each field becomes. It takes a built-in name, any
object answering `call`, or an Array of them applied left to right.

```ruby
# Built-in: recognise integers and floats
ZLight.parse(csv, converters: :numeric)

# Your own
ZLight.parse(csv, converters: ->(field) { field.strip })

# Chained: trim, then recognise numbers
ZLight.parse(csv, converters: [->(f) { f.strip }, :numeric])

# Anything that responds to #call
ZLight.parse(csv, converters: MyConverter.new)
```

Two rules, both matching Ruby's CSV:

**The chain stops** as soon as a converter returns something that is not a
String. So in `[:numeric, other]`, `other` is never called for a field that
`:numeric` already turned into a number.

**Converters never touch the header row.** Headers are always Symbols.

An exception raised inside a converter propagates to you unchanged. A converter
that is neither a known name nor callable raises `ArgumentError` saying so.

> **A converter must not use the reader it is converting for.** Calling
> `next_row`, `headers` or `close` on that reader raises `ZLight::Error`. Any
> *other* reader, and `ZLight.parse` itself, are fine.

---

## Coming from Ruby's CSV?

ZLight covers the common `CSV.parse` patterns:

```ruby
# Before
CSV.parse(data, headers: true, header_converters: :symbol, converters: :numeric)

# After
ZLight.parse(data, converters: :numeric)
```

It is not a complete reimplementation, though. These differences are deliberate
and are the ones most likely to bite during a migration.

**Shape of the API**

- `ZLight.foreach` takes a **CSV string**; `CSV.foreach` takes a **file path**.
  Use `ZLight.open` to iterate a file.
- `ZLight.foreach` parses everything before yielding. For genuinely lazy
  iteration use `ZLight.open` or `ZLight.stream`.
- `ZLight::StreamReader` is single-pass — no second pass, no rewind.

**Parsing**

- Duplicate headers collapse. Rows are Hashes, so `"a,a"` keeps only the last
  `:a` column; `CSV` keeps both.
- Fields past the header count are dropped rather than collected.
- Headers are always Symbols, the equivalent of `header_converters: :symbol`.

**`converters: :numeric` on edge cases**

| Field | ZLight | Ruby CSV |
|---|---|---|
| `""` | `""` | `nil` |
| `"0x10"` | `"0x10"` | `16` |
| `"1_000"` | `"1_000"` | `1000` |
| `"Infinity"` | `Float::INFINITY` | `"Infinity"` |
| `"NaN"` | `Float::NAN` | `"NaN"` |

Integers of any size stay exact, as in `CSV`.

**Writing**

- `generate` takes column order from the first Hash. Keys appearing only in
  later rows are not written.

---

## Errors

Everything the gem raises descends from `ZLight::Error`, so one rescue catches
the lot.

| Class | Raised when |
|---|---|
| `ZLight::Error` | base class; also raised directly for re-entrant reader use |
| `ZLight::ParseError` | the CSV is malformed |
| `ZLight::EncodingError` | a header is not valid UTF-8 and cannot become a Symbol |
| `ZLight::StreamClosedError` | a row is read from a closed reader |

```ruby
begin
  ZLight.parse(data)
rescue ZLight::ParseError => e
  warn "Bad CSV: #{e.message}"
end
```

Two that are **not** `ZLight::Error`, because they come from elsewhere:

- `ArgumentError` — an option is invalid, such as a multi-byte `col_sep` or an
  unusable converter.
- `IOError` — `stream_file` or `open` could not read the file. (`read` and
  `write` go through Ruby's `File`, so those raise `Errno::ENOENT` and friends.)

---

## Threads and Ractors

| | |
|---|---|
| `ZLight.parse`, `.generate`, `.read`, `.write` | safe from any thread |
| A reader shared between threads | safe while no converter is running |
| A reader shared between threads, with a converter | raises `ZLight::Error` |
| Ractors | not supported; Ruby refuses to cross the boundary |

A converter runs Ruby code, which lets another thread take the GVL part-way
through a row. The reader refuses that rather than handing back a corrupted
row. **Give each thread its own reader** and none of this comes up.

---

## Compatibility

- **Ruby 3.1 to 4.0**
- Linux (x86_64, aarch64, musl), macOS (Intel, Apple Silicon), Windows (x64)

Precompiled gems ship for every combination above. Anything else builds from
source and needs a Rust toolchain.

```ruby
ZLight::VERSION   # => "0.6.0"
```

---

## How fast, really

Parsing with headers and numeric conversion, on an Apple M1:

| Rows | Ruby CSV | ZLight | |
|---|---|---|---|
| 1,000 | 12.6 ms | 0.3 ms | **42× faster** |
| 10,000 | 133 ms | 4.4 ms | **30× faster** |
| 100,000 | 1,458 ms | 78 ms | **19× faster** |

Reading a 100,000-row file:

| | Ruby CSV | ZLight | |
|---|---|---|---|
| Read it all | 1,596 ms | 58 ms | **27× faster** |
| Stream it | 1,104 ms | 74 ms | **15× faster** |

The gap is widest on small inputs and narrows as they grow. Your numbers will
differ with hardware and Ruby version — the benchmark lives in the repository
if you want to run it yourself.

---

## Links

- [Source on GitHub](https://github.com/codebyisaad/zlight)
- [Gem on RubyGems](https://rubygems.org/gems/zlight_csv)
- [Changelog](https://github.com/codebyisaad/zlight/blob/main/CHANGELOG.md)
- [Report an issue](https://github.com/codebyisaad/zlight/issues)

By **Saad Chaudhary**, who also publishes as Zaidan Chaudhary. Released under
the MIT licence.
