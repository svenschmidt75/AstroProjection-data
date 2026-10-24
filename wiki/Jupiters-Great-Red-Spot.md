# Jupiter's Great Red Spot

Two questions: **where on the planet** is the spot, and **which way is that planet turned towards us** right now? The difference of the two longitudes tells where the spot sits on the disc.

## 1. Where the spot is on Jupiter

The spot lives at planetographic latitude **−22°** and a **System II** longitude `L_GRS`. System II is the rotation system of Jupiter's temperate and tropical cloud layer, turning 870.27° a day (a period of 9 h 55 m 40.6 s).

The spot has no fixed longitude: it **drifts**, lately by 16 to 21° a year. The app therefore uses one observed value plus a drift rate:

```
L_GRS(JD) = 91.0° + (1.75 / 30.4375) (JD - 2461192.5)
```

Here 91° is the longitude on 2026-06-01 (from Sky & Telescope's transit calculator), and 1.75° a month is the drift from the same source. The anchor needs refreshing every year or two.

## 2. Which longitude faces us: the central meridian

The **central meridian** (CM) is the System II longitude at the centre of the disc as seen from Earth. The app uses Project Pluto's low-accuracy series (about 1°):

```
jup_mean   = (JD - 2455636.938) × 360 / 4332.89709           Jupiter's mean orbital anomaly
eq_centre  = 5.55 sin(jup_mean)                              equation of centre
angle      = (JD - 2451870.628) × 360 / 398.884 - eq_centre  Jupiter-Earth synodic geometry
correction = 11 sin(angle) + 5 cos(angle) - 1.25 cos(jup_mean) - eq_centre
CM         = (181.62 + 870.1869147 × JD + correction) mod 360
```

The `870.1869147 × JD` term is the rotation; the correction accounts for Jupiter's and Earth's orbital motion (and so for light-time and viewing geometry). It was checked against ten published GRS transit times (2025-11-01 to 04): at every transit the CM was within 1° of one longitude.

## 3. Offset from the centre

```
delta = wrap(CM - L_GRS)     into (-180°, +180°]
```

- `delta < 0`: the spot has yet to reach the central meridian.
- `delta = 0`: it is **transiting**, in the middle of the disc.
- `delta > 0`: it has crossed.

Jupiter turns left to right across the disc, so `delta` grows at about `870.19 − 0.06 = 870.13` degrees a day, i.e. **36.26° an hour**, and the spot crosses the meridian every `360 / 36.26 = 9.93 h` (9 h 56 m).

## 4. Direction on the globe

In the planet's local frame the viewing direction is `+z` and the north pole is `P`:

```
meridian = normalize(z - (z·P) P)       the direction of the central meridian on the sphere
side     = P × meridian
c        = cos φ (cos δ meridian + sin δ side) + sin φ P          φ = -22°, δ = delta
```

`c` is the unit vector from the planet's centre to the spot.

## 5. The oval

The spot is about 16 000 × 12 000 km, which is **0.224 × 0.168** Jupiter radii. It is drawn in the plane tangent to the globe at `c`, so foreshortening towards the limb comes out right. With `along = normalize(P × c)`, `across = c × along`, and for a globe surface normal `n` the tangent offset `q = n − c (n·c)`:

```
e = (q·along / 0.224)² + (q·across / 0.168)²
```

The core of the spot is where `e` is small, a dark rim lies near `e = 0.9` and a pale collar around `e = 1.05`. It fades in over the limb as `n·c` goes from 0 to 0.2, so it appears gradually as it rotates into view.

## Worked example

**2026-10-16 12:00 TT** (JD 2461330.0):

1. `jup_mean = (2461330.0 − 2455636.938) × 360 / 4332.89709 = 473.0° → 113.01°`; `eq_centre = 5.55 sin 113.01° = 5.11°`.
2. `angle = (2461330.0 − 2451870.628) × 360 / 398.884 − 5.11 = 8537.2° → 252.15°` after removing whole turns.
3. `correction = 11 sin 252.15° + 5 cos 252.15° − 1.25 cos 113.01° − 5.11 = −10.46 − 1.54 + 0.49 − 5.11 = −16.62°`.
4. `CM = (181.62 + 870.1869147 × 2461330.0 − 16.62) mod 360 = 203.76°`.
5. `L_GRS = 91.0 + (1.75/30.4375)(2461330.0 − 2461192.5) = 91.0 + 7.91 = 98.91°`.
6. `delta = 203.76 − 98.91 = +104.85°`: the spot has crossed the central meridian and is 105° past it, which is beyond the limb at +90°, so it has just rotated out of sight on the right.

**When is the next transit?** `delta` has to reach 360° (the next 0°): `(360 − 104.85) / 36.26° per hour = 7.04 h`, so at about **19:02 TT** on the same day (about 19:01 UT, since TT is 69 s ahead of UT).

**In the code:** `ephemeris/jupiter.rs` (`central_meridian_system_ii_deg`, `grs_longitude_system_ii_deg`, `grs_offset_from_central_meridian_rad`), `renderer/pipelines.rs` (`grs_direction_local`) and the GRS block of `renderer/shaders/solar_system.wgsl`.
