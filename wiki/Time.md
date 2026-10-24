# Time

The sky depends on time in two ways: where the planets are (a smooth, uniform clock, TT) and how the Earth has turned (an irregular clock tied to the Earth's rotation, UT1). The app needs both.

## The clocks

| Name | What it is | Used for |
|---|---|---|
| **Local time** | UTC plus the time zone's offset (including daylight saving) | What you read on screen |
| **UTC** | Atomic time with leap seconds, kept within 0.9 s of UT1 | What the device clock gives |
| **UT1** | Time defined by the Earth's rotation angle | Sidereal time, hence Alt/Az |
| **TT** | Terrestrial Time, a uniform scale: `TT = TAI + 32.184 s` | Planet and Moon positions, precession, nutation |
| **TDB** | Barycentric time used by JPL ephemerides; differs from TT by under 2 ms | Treated as equal to TT |

The gap between TT and UT1 is **ΔT = TT − UT1**. It is about 69.2 s in 2026 and grows slowly and unpredictably as the Earth's spin varies.

## Julian date

The Julian date (JD) counts days since noon UT on 1 January 4713 BC, so subtracting two of them gives a time span directly. From Unix seconds `s` (seconds since 1970-01-01 00:00 UTC, which is JD 2440587.5):

```
JD = 2440587.5 + s / 86400
```

Two derived quantities are used everywhere:

```
d = JD - 2451545.0            days since J2000.0 (2000-01-01 12:00 TT)
T = d / 36525                 Julian centuries since J2000.0
```

## From the device clock to JD(UT1) and JD(TT)

The app takes the device's UTC as UT1. The two differ by less than 0.9 s, which turns the sky by under 14 arc-seconds, far below what is visible. Then:

```
JD(TT) = JD(UT1) + 69.2 s / 86400
```

ΔT is a fixed 69.2 s, not a table, which is accurate to well under a second for dates near 2026 and is irrelevant at screen resolution.

## Sidereal time

Sidereal time is the right ascension currently crossing your meridian. **Greenwich mean sidereal time** (IAU 1982 expression, from JD(UT1)):

```
GMST = 280.46061837 + 360.98564736629 d + 0.000387933 T² - T³/38710000     (degrees, mod 360)
```

The coefficient 360.98564736629 °/day is one full turn plus 0.9856° (the Earth's yearly orbit), so a sidereal day is 23 h 56 m 4.09 s.

**Apparent** sidereal time adds the *equation of the equinoxes*, `Δψ cos ε`, where Δψ is the nutation in longitude (a wobble of up to about 17″) and ε the obliquity. **Local** sidereal time adds the observer's east longitude:

```
GAST = GMST + Δψ cos ε
LST  = GAST + longitude         (east positive)
```

## Time zones and daylight saving

The zone is looked up from the observer's latitude and longitude (an embedded time-zone map giving an IANA name such as `America/Chicago`) and the offset for that instant, including daylight saving, is applied only for display. All astronomy uses UTC-derived JD.

## Worked example

**2026-10-23 03:00:00 UTC**, observer in Chicago (longitude −87.6298°).

1. Unix seconds: `1 792 724 400`.
2. `JD(UT1) = 2440587.5 + 1792724400/86400 = 2461336.625`.
3. `JD(TT) = 2461336.625 + 69.2/86400 = 2461336.625801`.
4. `d = 2461336.625 − 2451545.0 = 9791.625`, `T = 0.268081`.
5. `GMST = 280.46061837 + 360.98564736629 × 9791.625 + … = 76.550°` after reducing mod 360, which is 5 h 06.2 m.
6. `LST = 76.550 − 87.630 = −11.080° = 348.920°` (23 h 15.7 m), ignoring the equation of the equinoxes (a few seconds).

Local time in Chicago is 22:00 CDT on 22 October.

**In the code:** `observer/time.rs` (`ObserverTime`, `jd_ut1`, `jd_tt`), `math/frames.rs` (`gmst_rad`, `gast_rad`, `last_rad`).
