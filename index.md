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

## API Reference

### Parsing Methods

| Method | Description |
|--------|-------------|
| `ZLight.parse(string, **opts)` | Parse CSV string, returns array of hashes/arrays |
| `ZLight.read(path, **opts)` | Read and parse file |
| `ZLight.foreach(string, **opts) { }` | Iterate over rows |

### Streaming Methods

| Method | Description |
|--------|-------------|
| `ZLight.stream(string, **opts)` | Create stream reader from string |
| `ZLight.stream_file(path, **opts)` | Create stream reader from file |
| `ZLight.open(path, **opts) { }` | Stream with auto-close block |

### StreamReader Methods

| Method | Description |
|--------|-------------|
| `reader.next_row` | Get next row (nil if exhausted) |
| `reader.each { }` | Iterate all rows |
| `reader.headers` | Get header symbols |
| `reader.close` | Close and release resources |
| `reader.closed?` | Check if closed |
| `reader.eof?` | Check if exhausted |

### Options

| Option | Default | Description |
|--------|---------|-------------|
| `headers` | `true` | Use first row as headers (returns hashes). Set `false` for arrays. |
| `converters` | `nil` | Set to `:numeric` to convert numbers automatically |
| `col_sep` | `","` | Column separator (`"\t"` for TSV, `";"` for European CSV) |
| `quote_char` | `"` | Quote character |
| `flexible` | `true` | Allow rows with varying column counts |

## Requirements

- Ruby 3.0+
- Linux (x86_64, aarch64), macOS (Intel, Apple Silicon), or Windows (x64)

## License

MIT — [RubyGems](https://rubygems.org/gems/zlight_csv)
