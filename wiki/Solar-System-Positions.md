# Solar System Positions

Where the Sun, Moon and planets are, how far away, how bright, and how much of each is lit.

## What the app computes

For each body it needs a **geocentric (or topocentric) vector in ICRS**, in km. Everything else comes from that vector and the Sun's:

```
r_topo  = r_body(geocentric) - r_observer(from Earth's centre)      observer parallax; matters for the Moon
dist    = |r_topo|              direction = r_topo / dist
angular radius = asin(R_body / dist)
```

Positions are **geometric**: light-time and annual aberration (up to 20″) are not applied, which is invisible at screen scale. The time argument is JD(TT) (see [Time](Time.md)).

## Three tiers of accuracy

The app asks a *provider* for each body and uses the best one available.

1. **NASA/JPL DE440s** (optional pack, about 32 MB, valid 1849-2150). The planetary ephemeris is read with the `anise` SPICE library: a body's position relative to Earth is `translate(body, Earth, epoch)` in the ICRF frame. DE440s has no planet centres for Mars to Neptune, only the *system barycentre*, which differs from the planet by well under 0.1″ for these bodies. Accurate to well under an arc-second.
2. **Built-in Moon series** (always available): a truncated lunar theory with the largest periodic terms (below). Good to roughly 0.1-0.2°.
3. **Built-in planet elements** (always available): Keplerian orbital elements with linear rates (the JPL "approximate positions" set, valid roughly 1800-2050 and still sensible far outside), good to a few arc-minutes for the visible planets. The code lives in `vsop87.rs`, but this tier is the elements method, not the full VSOP87 series.

### The built-in planets

For each planet at `T` centuries from J2000, the elements are `X = X₀ + Ẋ T` for semi-major axis `a`, eccentricity `e`, inclination `I`, mean longitude `L`, longitude of perihelion `ϖ` and longitude of the ascending node `Ω`. Then:

```
M = L - ϖ                      mean anomaly
solve  E - e sin E = M          Kepler's equation, by Newton iteration (E ← E - f/f')
x' = a (cos E - e)              position in the orbit plane
y' = a sqrt(1 - e²) sin E
ω  = ϖ - Ω                     argument of perihelion
```

Rotate by ω in the plane, by `I` about the x axis and by `Ω` about z to get heliocentric ecliptic J2000 coordinates (AU). The Earth is done the same way (Earth-Moon barycentre elements). The geocentric vector is `r_planet - r_earth`, then rotated into ICRS by the obliquity ([The Ecliptic Line](The-Ecliptic-Line.md)). The Sun is simply `-r_earth`.

### The built-in Moon

The mean longitude `L'` and the Delaunay arguments are polynomials in `T`:

```
L' = 218.3164477 + 481267.88123421 T        mean longitude
D  = 297.8501921 + 445267.1114034  T        mean elongation from the Sun
M  = 357.5291092 +  35999.0502909  T        Sun's mean anomaly
M' = 134.9633964 + 477198.8675055  T        Moon's mean anomaly
F  =  93.2720950 + 483202.0175233  T        argument of latitude
```

Longitude, latitude and distance then come from sums of sines and cosines of combinations of these, for example:

```
λ  = L' + 6.288774 sin M' + 1.274027 sin(2D - M') + 0.658314 sin 2D + 0.213618 sin 2M' - 0.185116 sin M + ...   (14 terms)
β  = 5.128154 sin F + 0.280602 sin(M' + F) + ...                                                               (8 terms)
r  = 385000.56 - 20905.355 cos M' - 3699.111 cos(2D - M') - 2955.968 cos 2D + ...                              (8 terms)
```

Converted from ecliptic to ICRS and corrected for the observer's position on Earth (**diurnal parallax**, up to 1°), this gives the Moon to better than a fraction of a degree.

## Phase

The **phase angle** `i` is the angle at the body between the directions to the Sun and to the observer:

```
cos i = (r_body→Sun / |r_body→Sun|) · (r_body→observer / |r_body→observer|)
illuminated fraction k = (1 + cos i) / 2
```

`i = 0°` is full (Sun behind us), `i = 180°` is new. For the Moon this is the same as `k = (1 - cos ψ)/2` with ψ the Moon-Sun elongation measured from Earth, approximately.

**Waxing or waning.** The Moon waxes while it is east of the Sun, i.e. its ecliptic longitude is ahead of the Sun's. In vectors, with `s` the direction to the Sun, `m` to the Moon and `P` the ecliptic pole `(0, −sin ε₀, cos ε₀)`:

```
waxing  <=>  (s × m) · P ≥ 0
```

**Phase name** from `k`: below 0.03 New Moon; to 0.47 Crescent; 0.47-0.53 Quarter (First if waxing, Last if waning); to 0.97 Gibbous; above 0.97 Full.

**Bright limb.** The lit edge points towards the Sun. On the sky, in the frame where `east = ẑ × m̂` and `north = m̂ × east`, with `Δ = ŝ − m̂`:

```
χ = atan2(Δ · east, Δ · north)        position angle of the bright limb, from north through east
```

The shader uses this, plus the phase angle, to draw the terminator.

Mercury and Venus use exactly the same `i` and `k`: they show crescents when between us and the Sun (large `i`) and are nearly full when beyond it (small `i`), but then lie so close to the Sun that they are hard to see.

![The Moon phase from its elongation, eight positions](img/moon-phase.svg)

## Brightness (visual magnitude)

Planets: `V = V₀ + 5 log₁₀(r Δ) + c·i` with `r` the Sun distance and `Δ` the Earth distance in AU, `i` in degrees:

| Planet | V₀ | c (per degree) |
|---|---|---|
| Mercury | −0.42 | 0.038 |
| Venus | −4.40 | 0.009 |
| Mars | −1.52 | 0.016 |
| Jupiter | −9.40 | 0.005 |
| Saturn | −8.88 | 0.044 |
| Uranus | −7.19 | 0 |
| Neptune | −6.87 | 0 |

Moon: `V = −12.74 + 0.026 i + 4×10⁻⁹ i⁴`. Sun: −26.74.

## Worked example: Mars-like phase

Suppose, from the Sun, a planet is at distance `r = 1.5 AU` and from Earth at `Δ = 2.0 AU`, with Earth at 1 AU from the Sun. The angle at the planet follows from the triangle (law of cosines): `cos i = (r² + Δ² − R²) / (2 r Δ) = (2.25 + 4.0 − 1.0) / 6.0 = 0.875`, so `i = 28.96°` and `k = (1 + 0.875)/2 = 94 %`, a gibbous disc. Magnitude `V = −1.52 + 5 log₁₀(1.5 × 2.0) + 0.016 × 28.96 = −1.52 + 2.386 + 0.463 = 1.33`.

## Worked example: the Moon

If the Moon is 120° of ecliptic longitude east of the Sun, the phase angle at the Moon is about `i = 180° − 120° = 60°` (the Moon is so near that the Earth-Sun-Moon triangle is almost degenerate), so `k = (1 + cos 60°)/2 = 75 %`: east of the Sun means waxing, `k` between 0.53 and 0.97 means **Waxing Gibbous**. Its magnitude is `−12.74 + 0.026 × 60 + 4×10⁻⁹ × 60⁴ = −12.74 + 1.56 + 0.05 = −11.13`.

**In the code:** `ephemeris/mod.rs` (`assemble_body`: phase, waxing, bright limb, magnitudes), `ephemeris/provider.rs` (tiers), `ephemeris/spice.rs` (DE440s), `ephemeris/vsop87.rs` (planet elements, Sun, magnitudes), `ephemeris/elp2000.rs` (Moon series, `lunar_phase_name`).
