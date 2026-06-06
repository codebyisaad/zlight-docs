---
layout: default
---

# ZLight CSV

[![Gem Version](https://badge.fury.io/rb/zlight_csv.svg)](https://rubygems.org/gems/zlight_csv)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A fast CSV parser for Ruby, powered by Rust.

## Why ZLight?

Ruby's built-in CSV library is slow. ZLight parses CSV files **up to 30x faster** by using Rust under the hood.

### Benchmark Results

Parsing with headers and numeric conversion (Apple M1):

| Dataset | Ruby CSV | ZLight | Speedup |
|---------|----------|--------|---------|
| 1K rows | 12.6ms | 0.3ms | **42x faster** |
| 10K rows | 133ms | 4.4ms | **30x faster** |
| 100K rows | 1,458ms | 78ms | **19x faster** |

File reading comparison (100K rows):

| Method | Ruby CSV | ZLight | Speedup |
|--------|----------|--------|---------|
| Read all | 1,596ms | 58ms | **27x faster** |
| Streaming | 1,104ms | 74ms | **15x faster** |

## Installation

```ruby
gem 'zlight_csv'
```

No Rust toolchain required — prebuilt binaries are available for Linux, macOS, and Windows.

## Usage

```ruby
require 'zlight_csv'

# Parse a CSV string
data = ZLight.parse("name,age\nAlice,30\nBob,25")
# => [{:name=>"Alice", :age=>"30"}, {:name=>"Bob", :age=>"25"}]

# With automatic numeric conversion
data = ZLight.parse(csv_string, converters: :numeric)
# => [{:name=>"Alice", :age=>30}, {:name=>"Bob", :age=>25}]

# Read from a file
data = ZLight.read("users.csv")

# Iterate over rows
ZLight.foreach(csv_string) do |row|
  puts row[:name]
end
```

## Streaming Large Files

For large files, use streaming to process rows one at a time without loading everything into memory:

```ruby
# Stream from a file (auto-closes when block exits)
ZLight.open("large_file.csv") do |reader|
  reader.each do |row|
    process(row)
  end
end

# Lazy enumeration — stop early without loading remaining rows
ZLight.open("huge_file.csv", converters: :numeric) do |reader|
  high_scores = reader.lazy
                      .select { |row| row[:score] > 90 }
                      .first(100)
end

# Manual control
reader = ZLight.stream_file("data.csv")
while row = reader.next_row
  break if row[:id] > 1000
  process(row)
end
reader.close
```

**Streaming is especially efficient for partial reads:**

```
Finding first 100 rows from 100K dataset:

Ruby CSV (full parse):  1,696ms
ZLight.parse (full):       73ms
ZLight.stream (lazy):     0.3ms  ← 5,600x faster!
```

## Writing CSV

```ruby
# Generate CSV string from array of hashes
csv_string = ZLight.generate([
  { name: "Alice", age: 30 },
  { name: "Bob", age: 25 }
])
# => "name,age\nAlice,30\nBob,25\n"

# Generate from array of arrays (no headers)
csv_string = ZLight.generate([
  ["Alice", 30],
  ["Bob", 25]
], headers: false)

# Write directly to a file
ZLight.write("output.csv", data)

# Force-quote all fields
ZLight.write("output.csv", data, force_quotes: true)
```

---

## API Reference

### Module Methods (Reading)

#### `ZLight.parse(csv_string, **options)` → Array

Parse a CSV string and return an Array of Hashes (with headers) or Arrays (without headers).

```ruby
ZLight.parse("name,age\nAlice,30")
# => [{:name=>"Alice", :age=>"30"}]

ZLight.parse("Alice,30\nBob,25", headers: false)
# => [["Alice", "30"], ["Bob", "25"]]
```

#### `ZLight.read(path, **options)` → Array

Read and parse a CSV file. Accepts the same options as `parse`.

```ruby
ZLight.read("users.csv", converters: :numeric)
```

#### `ZLight.foreach(csv_string, **options, &block)` → Enumerator or nil

Iterate over rows. Returns an Enumerator if no block is given.

```ruby
ZLight.foreach(csv_string) { |row| puts row[:name] }

# Without block, returns Enumerator
ZLight.foreach(csv_string).map { |row| row[:name].upcase }
```

#### `ZLight.stream(csv_string, **options)` → StreamReader

Create a streaming reader from a string.

```ruby
reader = ZLight.stream(csv_string)
row = reader.next_row
reader.close
```

#### `ZLight.stream_file(path, **options)` → StreamReader

Create a streaming reader from a file.

```ruby
reader = ZLight.stream_file("large.csv")
reader.each { |row| process(row) }
reader.close
```

#### `ZLight.open(path, **options, &block)` → Object

Stream a file with auto-close. Works like `File.open`.

```ruby
ZLight.open("data.csv") do |reader|
  reader.each { |row| process(row) }
end
```

---

### Module Methods (Writing)

#### `ZLight.generate(rows, **options)` → String

Generate a CSV string from an Array of Hashes or Arrays.

```ruby
ZLight.generate([{ name: "Alice", age: 30 }])
# => "name,age\nAlice,30\n"

ZLight.generate([["Alice", 30]], headers: false)
# => "Alice,30\n"
```

#### `ZLight.write(path, rows, **options)` → Integer

Write CSV data to a file. Returns the number of bytes written.

```ruby
ZLight.write("output.csv", data)
ZLight.write("output.csv", data, force_quotes: true, col_sep: ";")
```

---

### StreamReader Instance Methods

StreamReader includes `Enumerable`, providing access to `map`, `select`, `find`, `lazy`, and other enumerable methods.

| Method | Description |
|--------|-------------|
| `#next_row` | Read and return the next row (Hash or Array), or `nil` at EOF |
| `#each(&block)` | Iterate over all rows; returns Enumerator if no block given |
| `#headers` | Return headers as Array of Symbols, or `nil` if headers disabled |
| `#close` | Close the reader and release resources |
| `#closed?` | Returns `true` if the reader is closed |
| `#eof?` | Returns `true` if the reader has reached end of file |

```ruby
reader = ZLight.stream_file("data.csv", converters: :numeric)

# Manual iteration
while row = reader.next_row
  break if row[:id] > 100
end

# Lazy enumeration for partial reads
first_ten = reader.lazy.first(10)

reader.close
```

---

### Options

#### Reading Options

| Option | Default | Description |
|--------|---------|-------------|
| `headers` | `true` | Use first row as headers (returns Hashes). Set `false` for Arrays. |
| `converters` | `nil` | Set to `:numeric` to auto-convert numeric strings to Integer/Float |
| `col_sep` | `","` | Column separator (`"\t"` for TSV, `";"` for European CSV) |
| `quote_char` | `"` | Quote character for fields containing separators or newlines |
| `flexible` | `true` | Allow rows with varying column counts |

#### Writing Options

| Option | Default | Description |
|--------|---------|-------------|
| `headers` | `true` | Write header row when input is Array of Hashes |
| `col_sep` | `","` | Column separator |
| `quote_char` | `"` | Quote character |
| `force_quotes` | `false` | Quote all fields, even if not required |

---

### Error Classes

All errors inherit from `ZLight::Error`.

| Class | Description |
|-------|-------------|
| `ZLight::Error` | Base error class for all ZLight errors |
| `ZLight::ParseError` | Raised when CSV is malformed |
| `ZLight::EncodingError` | Raised when headers contain invalid UTF-8 |
| `ZLight::StreamClosedError` | Raised when operating on a closed StreamReader |

```ruby
begin
  ZLight.parse(malformed_csv)
rescue ZLight::ParseError => e
  puts "Failed to parse: #{e.message}"
end
```

---

### Constants

| Constant | Description |
|----------|-------------|
| `ZLight::VERSION` | Gem version string (e.g., `"1.0.0"`) |

```ruby
puts ZLight::VERSION
# => "1.0.0"
```

---

## Author

**Saad Chaudhary** (aka Zaidan Chaudhary)

## Requirements

- Ruby 3.0+
- Linux (x86_64, aarch64), macOS (Intel, Apple Silicon), or Windows (x64)

## License

MIT — [RubyGems](https://rubygems.org/gems/zlight_csv)
