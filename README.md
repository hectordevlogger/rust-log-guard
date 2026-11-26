# rust-log-guard

A fast, zero-bloat **Rust** CLI utility for streaming log files, aggregating log severity levels, matching custom terms, and exporting structured JSON reports.

## Features

- Streams large files line-by-line via `BufReader` without loading the full file into memory.
- Classifies log severity levels (`CRITICAL`, `ERROR`, `WARN`, `INFO`, `OTHER`).
- Supports custom search patterns (`--pattern`) and machine-readable JSON output (`--json`).
- Includes unit tests via `cargo test`.

## Usage

```bash
cargo run -- /var/log/syslog --pattern "connection refused" --pattern "502"
cargo run -- /var/log/syslog --json
cargo test

* `rust-log-guard/src/main.rs`
```rust
use serde::Serialize;
use std::collections::BTreeMap;
use std::env;
use std::fs::File;
use std::io::{self, BufRead, BufReader};
use std::process;

#[derive(Debug, Default, Serialize)]
struct LogReport {
    file: String,
    total_lines: usize,
    level_counts: BTreeMap<String, usize>,
    matched_patterns: BTreeMap<String, usize>,
}

fn analyze_reader<R: BufRead>(reader: R, file_label: &str, patterns: &[String]) -> io::Result<LogReport> {
    let mut report = LogReport {
        file: file_label.to_string(),
        total_lines: 0,
        level_counts: BTreeMap::new(),
        matched_patterns: BTreeMap::new(),
    };

    for line_result in reader.lines() {
        let line = line_result?;
        report.total_lines += 1;

        let upper = line.to_uppercase();
        let level = if upper.contains("CRITICAL") || upper.contains("FATAL") {
            "CRITICAL"
        } else if upper.contains("ERROR") || upper.contains("ERR") {
            "ERROR"
        } else if upper.contains("WARN") {
            "WARN"
        } else if upper.contains("INFO") {
            "INFO"
        } else {
            "OTHER"
        };

        *report.level_counts.entry(level.to_string()).or_insert(0) += 1;

        for pattern in patterns {
            if line.contains(pattern) {
                *report.matched_patterns.entry(pattern.clone()).or_insert(0) += 1;
            }
        }
    }

    Ok(report)
}

fn print_usage() {
    eprintln!("Usage: rust-log-guard <log_file> [--json] [--pattern <term> ...]");
}

fn main() {
    let args: Vec<String> = env::args().skip(1).collect();
    if args.is_empty() {
        print_usage();
        process::exit(1);
    }

    let mut file_path: Option<String> = None;
    let mut json_output = false;
    let mut patterns: Vec<String> = vec!["timeout".to_string(), "unauthorized".to_string()];
    let mut custom_patterns = false;

    let mut i = 0;
    while i < args.len() {
        match args[i].as_str() {
            "--json" => json_output = true,
            "--pattern" => {
                if !custom_patterns {
                    patterns.clear();
                    custom_patterns = true;
                }
                i += 1;
                if i >= args.len() {
                    eprintln!("Error: --pattern requires a value.");
                    process::exit(1);
                }
                patterns.push(args[i].clone());
            }
            arg if arg.starts_with("--") => {
                eprintln!("Unknown flag: {}", arg);
                print_usage();
                process::exit(1);
            }
            arg if file_path.is_none() => {
                file_path = Some(arg.to_string());
            }
            other => {
                eprintln!("Unexpected argument: {}", other);
                print_usage();
                process::exit(1);
            }
        }
        i += 1;
    }

    let path = match file_path {
        Some(p) => p,
        None => {
            print_usage();
            process::exit(1);
        }
    };

    let file = match File::open(&path) {
        Ok(f) => f,
        Err(err) => {
            eprintln!("Failed to open '{}': {}", path, err);
            process::exit(1);
        }
    };

    let reader = BufReader::new(file);
    let report = match analyze_reader(reader, &path, &patterns) {
        Ok(r) => r,
        Err(err) => {
            eprintln!("Failed to read '{}': {}", path, err);
            process::exit(1);
        }
    };

    if json_output {
        match serde_json::to_string_pretty(&report) {
            Ok(json) => println!("{}", json),
            Err(err) => {
                eprintln!("Serialization error: {}", err);
                process::exit(1);
            }
        }
    } else {
        println!("File: {}", report.file);
        println!("Total lines: {}", report.total_lines);
        println!("Levels:");
        for (level, count) in &report.level_counts {
            println!("  {}: {}", level, count);
        }
        println!("Pattern matches:");
        for (pat, count) in &report.matched_patterns {
            println!("  {}: {}", pat, count);
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::io::Cursor;

    #[test]
    fn counts_levels_and_patterns() {
        let sample = "\
2026-11-27 INFO Service started
2026-11-27 WARN Upstream timeout on node-2
2026-11-27 ERROR unauthorized access attempt
";
        let cursor = Cursor::new(sample);
        let patterns = vec!["timeout".to_string(), "unauthorized".to_string()];
        let report = analyze_reader(cursor, "app.log", &patterns).unwrap();

        assert_eq!(report.total_lines, 3);
        assert_eq!(report.level_counts.get("INFO"), Some(&1));
        assert_eq!(report.level_counts.get("WARN"), Some(&1));
        assert_eq!(report.level_counts.get("ERROR"), Some(&1));
        assert_eq!(report.matched_patterns.get("timeout"), Some(&1));
        assert_eq!(report.matched_patterns.get("unauthorized"), Some(&1));
    }
}