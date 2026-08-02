# 10 · Project — Weather CLI

The capstone for Level 2: a command-line tool that looks up a city, fetches
its current conditions and an hourly forecast from a real public API, and
prints a summary. It deliberately pulls together everything from this
level: a **trait** (`WeatherSource`) so the fetch logic is swappable, a
**generic function** built against that trait, **closures and iterator
adapters** to crunch the forecast data, a **custom error type** that unifies
network and "city not found" failures, and **unit tests** for the pure
logic that don't require a network call to run.

## The API — no key required

This project uses [Open-Meteo](https://open-meteo.com), a free weather API
that needs no API key or sign-up. It's actually two endpoints:

1. **Geocoding** — turn a city name into coordinates:
   `https://geocoding-api.open-meteo.com/v1/search?name=Tokyo&count=1&language=en&format=json`
2. **Forecast** — fetch current + hourly weather for those coordinates:
   `https://api.open-meteo.com/v1/forecast?latitude=..&longitude=..&current_weather=true&hourly=temperature_2m`

If you swap in a different weather API that *does* require a key, the only
place it belongs is inside `OpenMeteoClient` (or a renamed equivalent) —
that's exactly the point of hiding the fetch behind the `WeatherSource`
trait.

## Project layout

```text
weather_cli/
    Cargo.toml
    src/
        main.rs      -- CLI entry point, argument handling, report rendering
        client.rs     -- WeatherSource trait + the OpenMeteoClient implementation
        error.rs      -- WeatherError, the custom error type
        models.rs      -- Deserialize structs for the API responses + WeatherReport
        stats.rs        -- Pure functions over forecast data (unit tested)
```

```toml
# Cargo.toml
[package]
name = "weather_cli"
version = "0.1.0"
edition = "2021"

[dependencies]
reqwest = { version = "0.12", features = ["blocking", "json"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

`reqwest`'s `blocking` feature avoids needing an async runtime for a small
CLI — fine here since the program does one thing and exits; a server would
want the async client instead. `serde`'s `derive` feature is what generates
`Deserialize` implementations from `#[derive(Deserialize)]`.

## src/error.rs — one error type for everything that can go wrong

```rust
// src/error.rs
use std::fmt;

#[derive(Debug)]
pub enum WeatherError {
    Http(reqwest::Error),
    CityNotFound(String),
}

impl fmt::Display for WeatherError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            WeatherError::Http(e) => write!(f, "network error: {e}"),
            WeatherError::CityNotFound(city) => write!(f, "could not find a city named '{city}'"),
        }
    }
}

impl std::error::Error for WeatherError {}

impl From<reqwest::Error> for WeatherError {
    fn from(e: reqwest::Error) -> Self {
        WeatherError::Http(e)
    }
}
```

The `From<reqwest::Error>` impl is what lets `?` inside `client.rs` convert
a `reqwest::Error` into a `WeatherError` automatically — the same pattern
from [Module 05](05-error-handling-advanced.md).

## src/models.rs — API response shapes and the report we build from them

```rust
// src/models.rs
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct GeoResult {
    pub name: String,
    pub latitude: f64,
    pub longitude: f64,
    pub country: String,
}

#[derive(Debug, Deserialize)]
pub struct GeoResponse {
    pub results: Option<Vec<GeoResult>>,
}

#[derive(Debug, Deserialize)]
pub struct CurrentWeather {
    pub temperature: f64,
    pub windspeed: f64,
    pub weathercode: u32,
}

#[derive(Debug, Deserialize)]
pub struct HourlyData {
    pub temperature_2m: Vec<f64>,
}

#[derive(Debug, Deserialize)]
pub struct ForecastResponse {
    pub current_weather: CurrentWeather,
    pub hourly: HourlyData,
}

/// The shape our own code works with, independent of the API's JSON layout --
/// if the API changed its field names tomorrow, only `client.rs` would need
/// to change, not every place that consumes a report.
#[derive(Debug)]
pub struct WeatherReport {
    pub city: String,
    pub country: String,
    pub current_temp_c: f64,
    pub windspeed_kmh: f64,
    pub weathercode: u32,
    pub hourly_temps: Vec<f64>,
}
```

`results: Option<Vec<GeoResult>>` matters: when a city genuinely isn't
found, Open-Meteo returns JSON with no `"results"` key at all rather than an
empty array — `Option` models "the field might not be there," exactly like
[Level 1's error handling](../level-1/07-error-handling-basics.md) intro
covered for `Option<T>` in general.

## src/stats.rs — pure logic, kept separate so it's testable

```rust
// src/stats.rs

/// Describes a WMO weather code as a short human-readable string.
/// See https://open-meteo.com/en/docs for the full code table -- this
/// covers the common buckets.
pub fn describe_weathercode(code: u32) -> &'static str {
    match code {
        0 => "clear sky",
        1 | 2 | 3 => "partly cloudy",
        45 | 48 => "fog",
        51..=57 => "drizzle",
        61..=67 => "rain",
        71..=77 => "snow",
        80..=82 => "rain showers",
        95..=99 => "thunderstorm",
        _ => "unknown conditions",
    }
}

/// Returns (min, max, average) of a slice of hourly temperatures.
/// Returns None if the slice is empty -- an average of zero items is undefined.
pub fn temperature_stats(temps: &[f64]) -> Option<(f64, f64, f64)> {
    if temps.is_empty() {
        return None;
    }

    let min = temps.iter().cloned().fold(f64::INFINITY, f64::min);
    let max = temps.iter().cloned().fold(f64::NEG_INFINITY, f64::max);
    let sum: f64 = temps.iter().sum();
    let avg = sum / temps.len() as f64;

    Some((min, max, avg))
}

/// Counts how many of the next `hours` hourly readings are above `threshold`.
pub fn hours_above(temps: &[f64], hours: usize, threshold: f64) -> usize {
    temps.iter().take(hours).filter(|&&t| t > threshold).count()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn describes_known_codes() {
        assert_eq!(describe_weathercode(0), "clear sky");
        assert_eq!(describe_weathercode(63), "rain");
        assert_eq!(describe_weathercode(999), "unknown conditions");
    }

    #[test]
    fn computes_stats_correctly() {
        let temps = vec![10.0, 20.0, 15.0];
        let (min, max, avg) = temperature_stats(&temps).unwrap();
        assert_eq!(min, 10.0);
        assert_eq!(max, 20.0);
        assert!((avg - 15.0).abs() < f64::EPSILON);
    }

    #[test]
    fn empty_slice_has_no_stats() {
        assert_eq!(temperature_stats(&[]), None);
    }

    #[test]
    fn counts_hours_above_threshold() {
        let temps = vec![10.0, 25.0, 30.0, 5.0, 40.0];
        assert_eq!(hours_above(&temps, 3, 20.0), 2);
        assert_eq!(hours_above(&temps, 5, 20.0), 3);
    }
}
```

This is a deliberate design choice worth calling out: none of these
functions touch the network, so `cargo test` runs them instantly and
deterministically, with no risk of flaking due to a slow or unreachable API.
Keeping "pure computation" separate from "I/O" like this is one of the most
useful habits for making Rust code testable.

## src/client.rs — the trait, and the one implementation that talks HTTP

```rust
// src/client.rs
use crate::error::WeatherError;
use crate::models::{ForecastResponse, GeoResponse, WeatherReport};

/// Anything that can look up a weather report for a city name.
/// Defining this as a trait (rather than calling OpenMeteoClient directly
/// everywhere) means the code that consumes a report can work with any
/// implementation -- a mock for tests, or a different weather API --
/// without changing a single line of the code that consumes it.
pub trait WeatherSource {
    fn fetch(&self, city: &str) -> Result<WeatherReport, WeatherError>;
}

pub struct OpenMeteoClient {
    http: reqwest::blocking::Client,
}

impl OpenMeteoClient {
    pub fn new() -> Self {
        OpenMeteoClient {
            http: reqwest::blocking::Client::new(),
        }
    }
}

impl WeatherSource for OpenMeteoClient {
    fn fetch(&self, city: &str) -> Result<WeatherReport, WeatherError> {
        // Step 1: turn the city name into coordinates via the (free, no-key)
        // Open-Meteo geocoding API.
        let geo_url = format!(
            "https://geocoding-api.open-meteo.com/v1/search?name={}&count=1&language=en&format=json",
            urlencode(city)
        );
        let geo: GeoResponse = self.http.get(&geo_url).send()?.json()?;

        let place = geo
            .results
            .and_then(|mut results| if results.is_empty() { None } else { Some(results.remove(0)) })
            .ok_or_else(|| WeatherError::CityNotFound(city.to_string()))?;

        // Step 2: fetch current weather + an hourly forecast for those coordinates.
        let forecast_url = format!(
            "https://api.open-meteo.com/v1/forecast?latitude={}&longitude={}&current_weather=true&hourly=temperature_2m",
            place.latitude, place.longitude
        );
        let forecast: ForecastResponse = self.http.get(&forecast_url).send()?.json()?;

        Ok(WeatherReport {
            city: place.name,
            country: place.country,
            current_temp_c: forecast.current_weather.temperature,
            windspeed_kmh: forecast.current_weather.windspeed,
            weathercode: forecast.current_weather.weathercode,
            hourly_temps: forecast.hourly.temperature_2m,
        })
    }
}

/// Minimal query-string escaping -- good enough for city names, which are
/// mostly letters and spaces. A real project would pull in the `url` crate.
fn urlencode(s: &str) -> String {
    s.chars()
        .map(|c| if c == ' ' { "%20".to_string() } else { c.to_string() })
        .collect()
}
```

The two `?` calls after `self.http.get(&url).send()` and `.json()` both
return `reqwest::Error` on failure, and both convert into `WeatherError`
automatically via the `From` impl in `error.rs`. `.ok_or_else()` turns the
"no results" case into the other `WeatherError` variant — this single
function ends up demonstrating both directions of Module 05's error story:
automatic conversion via `?`, and manual conversion via `.ok_or_else()`/`.map_err()`.

## src/main.rs — tying it together with a generic function

```rust
// src/main.rs
mod client;
mod error;
mod models;
mod stats;

use client::{OpenMeteoClient, WeatherSource};
use error::WeatherError;
use models::WeatherReport;

/// Generic over any WeatherSource -- this function doesn't care whether the
/// report came from OpenMeteoClient or a test double, only that it
/// implements the trait. This is the traits + generics half of the project.
fn print_report<S: WeatherSource>(source: &S, city: &str) -> Result<(), WeatherError> {
    let report: WeatherReport = source.fetch(city)?;
    render(&report);
    Ok(())
}

/// Uses closures and iterator adapters (filter, count, fold via the stats
/// module) over the hourly forecast to build the printed summary.
fn render(report: &WeatherReport) {
    println!("Weather for {}, {}", report.city, report.country);
    println!(
        "  Right now: {:.1}C, wind {:.1} km/h, {}",
        report.current_temp_c,
        report.windspeed_kmh,
        stats::describe_weathercode(report.weathercode)
    );

    if let Some((min, max, avg)) = stats::temperature_stats(&report.hourly_temps) {
        println!(
            "  Next {} hours: min {:.1}C, max {:.1}C, avg {:.1}C",
            report.hourly_temps.len(),
            min,
            max,
            avg
        );
    }

    let warm_hours = stats::hours_above(&report.hourly_temps, 24, 25.0);
    println!("  Hours above 25C in the next day: {warm_hours}");
}

fn main() {
    let city = std::env::args().nth(1).unwrap_or_else(|| "London".to_string());

    let client = OpenMeteoClient::new();
    if let Err(e) = print_report(&client, &city) {
        eprintln!("Error: {e}");
        std::process::exit(1);
    }
}
```

`print_report<S: WeatherSource>` never mentions `OpenMeteoClient` by name —
if you later write a `struct FakeSource` for testing that returns a
hand-built `WeatherReport` without touching the network, `print_report`
works with it unchanged, because it only depends on the trait.

## Running it

```bash
cargo run -- Tokyo
```

```text
Weather for Tokyo, Japan
  Right now: 30.8C, wind 5.4 km/h, partly cloudy
  Next 168 hours: min 19.9C, max 32.4C, avg 26.2C
  Hours above 25C in the next day: 13
```

(Real numbers, from live weather at the time this was run — yours will
differ.) An unknown city hits the custom error path instead of crashing:

```bash
cargo run -- Zzznotacityxyz
```

```text
Error: could not find a city named 'Zzznotacityxyz'
```

Run the test suite — this exercises `stats.rs` only, with zero network
calls:

```bash
cargo test
```

```text
running 4 tests
test stats::tests::describes_known_codes ... ok
test stats::tests::computes_stats_correctly ... ok
test stats::tests::empty_slice_has_no_stats ... ok
test stats::tests::counts_hours_above_threshold ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## Stretch goals

- Add a `FakeSource` implementing `WeatherSource` that returns a hand-built
  `WeatherReport` with no network call, and write an integration test in
  `tests/` (see [Module 06](06-testing.md)) that calls `print_report`
  against it.
- Cache the geocoding lookup for a city in a local JSON file (using
  `serde_json` and [Module 08's file I/O](08-files-stdlib.md)) so repeat
  runs for the same city skip the geocoding API call.
- Accept multiple cities on the command line and print a report for each,
  using `std::env::args().skip(1)` and a `for` loop — or, using
  [Module 07's `Rc<RefCell<T>>`](07-smart-pointers.md), collect results into
  a shared summary structure updated from a closure passed to `.for_each()`.
- Add a `--json` flag that, instead of the human-readable report, prints the
  `WeatherReport` serialized with `serde_json::to_string_pretty` (you'll
  need `#[derive(Serialize)]` on `WeatherReport`).
