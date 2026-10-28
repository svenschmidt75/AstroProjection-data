# Rise, Transit and Set

Every info card (Sun, Moon, planets, stars, deep-sky objects, constellations and asterisms) shows the **next** rise, the **next** transit and the **next** set after the time the sky is showing, in the observer's local time. Next to them are the rise and set azimuths and the altitude at transit. The times follow the app's clock: when you scrub the twilight bar or run time faster, the card moves on to the next events as soon as one has passed.

## What the three events mean

- **Rise** and **set**: the moment the object's top edge touches the horizon you would see on a flat, sea-level horizon. Two effects lift the object, so it is "up" while its centre is still slightly below the geometric horizon:
  - **Refraction.** The air bends light near the horizon by about **34′**.
  - **Size.** For a disk, the top edge (upper limb) shows first. Its angular radius `s` is about 16′ for the Sun and 15′ to 17′ for the Moon. It is negligible for planets and zero for stars.
- **Transit** (upper culmination): the moment the object crosses your meridian, the north–south line through the zenith. Its hour angle is then 0 and it is at its highest. The other crossing, below the pole, is the *lower* culmination and is not shown.

So the object is up when its centre's true altitude `h` is above

```
h₀ = −34′ − s
```

| Object | `s` | `h₀` |
|---|---|---|
| Star, deep-sky object, constellation centre | 0 | −0.567° |
| Planet | under 0.01° | about −0.57° |
| Sun | about 16′ | about −0.83° |
| Moon | about 15.5′ (topocentric) | about −0.83° |

The Moon needs care. Seen from the Earth's centre, rise and set happen at `h₀ ≈ +0.125°`, because the Moon's parallax (about 57′) lowers it as seen from the surface. The app already computes the Moon's direction *from the observer's position* (see [Solar System Positions](Solar-System-Positions.md)), so parallax is included in `h`, and the same rule `−34′ − s` works for the Moon too.

## The classical formula (fixed objects)

For a star at declination `δ`, seen from latitude `φ`, the hour angle `H₀` at which it reaches `h₀` follows from the altitude formula in [Coordinate Systems](Coordinate-Systems.md):

```
cos H₀ = (sin h₀ − sin φ sin δ) / (cos φ cos δ)
```

- If `cos H₀ < −1`, the star never goes below `h₀`: it is **always above the horizon** (circumpolar).
- If `cos H₀ > 1`, it never reaches `h₀`: it **never rises**.
- Otherwise it rises at hour angle `−H₀` (that is, `360° − H₀`) and sets at `+H₀`, and transits at `H = 0`.

The hour angle grows by 360.98565° per solar day (15.04107° per hour; see [Time](Time.md)). So if the hour angle now is `H`, an event at hour angle `H_e` comes `((H_e − H) mod 360°) / 15.04107°` hours from now.

The azimuth at rise follows from the same triangle, measured from north through east:

```
cos A_rise = (sin δ − sin φ sin h₀) / (cos φ cos h₀),      A_set = 360° − A_rise
```

The altitude at transit is `90° − φ + δ` for an object that culminates south of the zenith. If `δ > φ` it culminates north of the zenith, and the altitude is `90° + φ − δ`.

## How the app does it: a numerical search

The formula above assumes the object stands still. The Sun moves about 1° a day against the stars, and the Moon about 13°. That shifts the Moon's rise by roughly 50 minutes a day. Instead of a different formula for each kind of object, the app uses one numerical method for all of them:

1. **A position function.** For any instant `t` it gets the object's ICRS unit vector `p(t)` and angular radius `s(t)`. A fixed object has a constant `p` (a constellation or asterism uses its centre). For the Sun, Moon and planets, the ephemeris is evaluated at `t` with the observer's position at `t`.
2. **Alt/Az at that instant.** Build the zenith, north and east vectors `Z, N, E` for `t` (see [Coordinate Systems](Coordinate-Systems.md)). Then `h = asin(p · Z)` and `A = atan2(p · E, p · N)`. Precession and nutation are included through those vectors.
3. **Two test functions.**
   - Horizon: `f(t) = h(t) − h₀(t)`. A rise is `f` going from negative to non-negative, a set the reverse.
   - Meridian: `g(t) = p(t) · E(t)`, the component of the direction towards the east. An object east of the meridian has `g > 0`. Crossing the meridian from east to west, the upper transit, makes `g` go from positive to zero or negative. Crossing below the pole goes the other way, from west to east, so the lower culmination is ignored without extra tests.
4. **Coarse scan.** Starting at the current sky time `t₀`, sample every **10 minutes** for **48 hours** (289 samples) and look for the first sign change of each kind.
5. **Bisection.** Each bracket is halved 14 times. That narrows 600 s to under 0.04 s, far below the one-minute display.
6. **Results.** The rise and set report the azimuth at the refined instant, and the transit reports the true altitude, without refraction. Only the first event of each kind after `t₀` is kept.
7. **Status.** If neither a rise nor a set is found in the 48 hours, the card says **Always above the horizon** (if `f(t₀) ≥ 0`) or **Never rises** (otherwise). The transit is still shown if there is one.

A full search for the Moon takes about 0.3 ms in a release build, because only the one body is evaluated at each sample.

### Keeping the card current

The card asks for the times once a second. The engine keeps the last answer and reuses it while all of these are true:

- the sky time has not gone backwards,
- it has not reached the earliest of the reported events,
- less than 6 hours have passed since the answer was computed,
- the object, the observer's location and the ephemeris are unchanged.

So the search runs again only when an event has passed, when time jumps, or when something else changed.

### Showing the times

Times are shown in the **observer's** time zone: the IANA zone found from the location (see [Time](Time.md)), with daylight saving. This is not the device's own zone. Each time carries a day word relative to the sky time's local date: `today`, `tomorrow`, or the weekday (for example `Sat`). The header names the zone abbreviation, for example `CDT`. Azimuths are rounded to the nearest degree and given a 16-point compass name: `32° NNE`.

## Worked example

Vega (α = 279.235°, δ = +38.784°) from Chicago (φ = 41.8781°, λ = −87.6298°) at **2026-10-23 03:00 UTC**. That is 22:00 CDT on 22 October, the same instant as the examples in [Time](Time.md) and [Coordinate Systems](Coordinate-Systems.md). There, Vega's hour angle was `H = 69.686°` and its altitude 38.29°: it is past the meridian and sinking in the north-west.

1. Horizon for a star: `h₀ = −34′ = −0.5667°`, `sin h₀ = −0.00989`.
2. `sin φ sin δ = 0.6675 × 0.6264 = 0.4181`, `cos φ cos δ = 0.7446 × 0.7795 = 0.5804`.
3. `cos H₀ = (−0.00989 − 0.4181) / 0.5804 = −0.7375`, so **H₀ = 137.517°**. Vega rises and sets at Chicago.
4. Next **set** (H = 137.517°): `(137.517 − 69.686) / 15.04107 = 4.510 h` → 07:30.6 UTC.
5. Next **rise** (H = 360 − 137.517 = 222.483°): `(222.483 − 69.686) / 15.04107 = 10.159 h` → 13:09.5 UTC.
6. Next **transit** (H = 360°): `(360 − 69.686) / 15.04107 = 19.301 h` → 22:18.1 UTC.
7. Rise azimuth: `cos A = (0.6264 − 0.6675 × (−0.00989)) / (0.7446 × 0.99995) = 0.8502`, so **A = 31.8°** (NNE). The set is at 360° − 31.8° = **328.2°** (NNW).
8. Transit altitude: `90 − 41.878 + 38.784 = 86.91°`, almost overhead.

What the app shows for the same instant (CDT is UTC − 5 h):

| | Hand calculation | App (UTC) | On the card |
|---|---|---|---|
| Set | 07:30.6 UTC, 328.2° | 07:31, 328.3° | `02:31 tomorrow`, `328° NNW` |
| Rise | 13:09.5 UTC, 31.8° | 13:10, 31.7° | `08:10 tomorrow`, `32° NNE` |
| Transit | 22:18.1 UTC, 86.91° | 22:18, 86.93° | `17:18 tomorrow`, `87° alt` |

The two agree to under a minute. The small difference is precession: the app uses Vega's position of date, which has moved about 0.3° since J2000. All three events fall on 23 October in Chicago, which is "tomorrow" for a sky showing the evening of the 22nd.

The same instant for the Sun and the Moon (the Moon is a waxing gibbous, three days before full). For these, the times come from the search, because the formula above does not handle moving bodies:

| | Rise | Transit | Set |
|---|---|---|---|
| Sun | `07:12 tomorrow`, `105° ESE` | `12:34 tomorrow`, `37° alt` | `17:56 tomorrow`, `255° WSW` |
| Moon | `16:31 tomorrow`, `86° E` | `22:08 today`, `46° alt` | `04:18 tomorrow`, `270° W` |

The Sun's times match published sunrise and sunset tables for Chicago on that date to the minute.

## What is simplified

- Refraction is the standard 34′ for every place and weather. Real refraction near the horizon changes with temperature and pressure by a few arc-minutes, which can move a rise by a minute or two.
- The horizon is flat and at sea level. Mountains, buildings and the dip of the horizon from a height are not modelled.
- Two events closer together than one 10-minute step can be missed: an object that just grazes the horizon at high latitudes may rise and set between two samples.
- At very high latitudes the Sun can stay up, or down, longer than the 48-hour window. The card then says "Always above the horizon" or "Never rises" for the next two days, which is correct for those two days but not forever.
- For a constellation or asterism, the times are for the centre of the figure. Parts of a large figure rise noticeably earlier or later.

**In the code:** `observer/rise_set.rs` (`next_rise_transit_set`, `format_event_time`, `format_azimuth`), `engine_sky.rs` (`AstroEngine::rise_transit_set`, with the cache), `ephemeris/mod.rs` (`EphemerisEngine::compute_body_dir_and_radius`), `observer/location.rs` (`ObserverLocation::position_icrs_km`). The shells call `rise_transit_set` in `astro-ffi` once a second while a card is open.
