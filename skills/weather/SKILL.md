---
name: weather
description: >
  OpenCode skill: get current weather + hourly/daily forecast for any location
  using Open-Meteo (no API key, JSON). Works without browser redirects.
  Use for: weather, temperature, rain chance, wind, forecast (today/tomorrow/7-day).
  Not for: severe alerts, METAR/aviation, long historical climate analysis.
homepage: https://open-meteo.com/
metadata:
  openclaw:
    emoji: "🌦️"
    requires:
      bins: ["curl", "jq"]
---

# WEATHER (OpenCode Skill)

This skill answers weather questions by:
1) resolving a location name -> lat/lon via Open-Meteo Geocoding
2) fetching forecast/current from Open-Meteo Forecast API
3) printing a clean human summary + optional compact JSON

## Input Rules
- If user provides a city/region/country: geocode it.
- If user provides lat,lon: skip geocoding.
- Default timezone: `auto`.
- If location is ambiguous: pick top result but always print “Resolved to: …”.

## API Endpoints
- Geocoding:
  https://geocoding-api.open-meteo.com/v1/search?name=...&count=...&language=en&format=json
- Forecast:
  https://api.open-meteo.com/v1/forecast?latitude=...&longitude=...&timezone=auto&...

## Commands

### 1) Resolve location -> lat/lon (JSON)
```bash
# Usage: weather_geocode "Hanoi"
PLACE="${1:-Hanoi}"

curl -sG "https://geocoding-api.open-meteo.com/v1/search" \
  --data-urlencode "name=${PLACE}" \
  --data-urlencode "count=5" \
  --data-urlencode "language=en" \
  --data-urlencode "format=json" \
| jq -c '
  if (.results|length)==0 then
    {ok:false, error:"Could not resolve location"}
  else
    {
      ok:true,
      chosen:{
        name:(.results[0].name),
        admin1:(.results[0].admin1 // ""),
        country:(.results[0].country // ""),
        latitude:(.results[0].latitude),
        longitude:(.results[0].longitude),
        timezone:(.results[0].timezone // "auto")
      },
      candidates:(.results | map({
        name, admin1:(.admin1 // ""), country:(.country // ""),
        latitude, longitude, timezone:(.timezone // "auto")
      }))
    }
  end
'
````

### 2) Current + 48h hourly + 7-day daily (JSON)

```bash
# Usage: weather_forecast <lat> <lon> [timezone]
LAT="$1"; LON="$2"; TZ="${3:-auto}"

curl -sG "https://api.open-meteo.com/v1/forecast" \
  --data-urlencode "latitude=${LAT}" \
  --data-urlencode "longitude=${LON}" \
  --data-urlencode "timezone=${TZ}" \
  --data-urlencode "current=temperature_2m,apparent_temperature,is_day,precipitation,rain,showers,snowfall,weather_code,wind_speed_10m,wind_direction_10m" \
  --data-urlencode "hourly=temperature_2m,precipitation_probability,precipitation,rain,weather_code,wind_speed_10m" \
  --data-urlencode "forecast_hours=48" \
  --data-urlencode "daily=weather_code,temperature_2m_max,temperature_2m_min,precipitation_probability_max,precipitation_sum,wind_speed_10m_max,uv_index_max" \
  --data-urlencode "forecast_days=7"
```

### 3) One-shot summary from place name (recommended)

```bash
# Usage: weather "Hoan Kiem, Hanoi"
PLACE="${1:-Hanoi}"

GEO="$(curl -sG "https://geocoding-api.open-meteo.com/v1/search" \
  --data-urlencode "name=${PLACE}" \
  --data-urlencode "count=1" \
  --data-urlencode "language=en" \
  --data-urlencode "format=json")"

LAT="$(echo "$GEO" | jq -r '.results[0].latitude // empty')"
LON="$(echo "$GEO" | jq -r '.results[0].longitude // empty')"
TZ="$(echo "$GEO" | jq -r '.results[0].timezone // "auto"')"
NAME="$(echo "$GEO" | jq -r '.results[0] | "\(.name), \(.admin1 // ""), \(.country // "")" | gsub(", ,"; ", ") | gsub(", $"; "")')"

if [ -z "$LAT" ] || [ -z "$LON" ]; then
  echo "Could not resolve location: ${PLACE}" >&2
  echo "$GEO" | jq -r '.' >&2
  exit 1
fi

WX="$(curl -sG "https://api.open-meteo.com/v1/forecast" \
  --data-urlencode "latitude=${LAT}" \
  --data-urlencode "longitude=${LON}" \
  --data-urlencode "timezone=${TZ}" \
  --data-urlencode "current=temperature_2m,apparent_temperature,precipitation,rain,showers,snowfall,weather_code,wind_speed_10m,wind_direction_10m" \
  --data-urlencode "hourly=precipitation_probability,precipitation,weather_code" \
  --data-urlencode "forecast_hours=24")"

echo "$WX" | jq -r --arg NAME "$NAME" '
  def wmo(code):
    (code|tostring) as $c |
    if   $c=="0" then "Clear"
    elif $c=="1" or $c=="2" then "Partly cloudy"
    elif $c=="3" then "Overcast"
    elif $c=="45" or $c=="48" then "Fog"
    elif $c=="51" or $c=="53" or $c=="55" then "Drizzle"
    elif $c=="61" or $c=="63" or $c=="65" then "Rain"
    elif $c=="66" or $c=="67" then "Freezing rain"
    elif $c=="71" or $c=="73" or $c=="75" then "Snow"
    elif $c=="80" or $c=="81" or $c=="82" then "Rain showers"
    elif $c=="95" then "Thunderstorm"
    elif $c=="96" or $c=="99" then "Thunderstorm w/ hail"
    else "Unknown"
    end;

  .current as $c |
  ($NAME + "\n")
  + ("Now: \($c.temperature_2m)°C (feels \($c.apparent_temperature)°C) · \(wmo($c.weather_code)) · Wind \($c.wind_speed_10m) km/h\n")
  + ("Precip now: \($c.precipitation) mm (rain \($c.rain) mm, showers \($c.showers) mm)\n")
  + ("Next 24h rain chance (max): \((.hourly.precipitation_probability | max) // "n/a")% · precip sum: \((.hourly.precipitation | add) // "n/a") mm")
'
```

## Routing Heuristics (Agent Behavior)

* If user asks “now / hiện tại”: run `weather "<place>"`.
* If user asks “mấy tiếng tới / next hours”: run `weather "<place>"` (24h) OR `weather_forecast` and summarize first 12–24 hours.
* If user asks “tuần này / 7 ngày”: run `weather_forecast` then summarize `.daily`.
* If user provides coordinates “lat,lon”: call `weather_forecast <lat> <lon> auto` directly.
* Always show:

  * resolved location name
  * temperature + feels_like
  * condition (from weather_code)
  * wind
  * rain chance / precipitation summary (when relevant)

## Notes

* No API key required
* Don’t spam: max 2 network calls per user question (geocode + forecast)
* Open-Meteo weather codes are WMO style and mapped in jq `wmo()`
