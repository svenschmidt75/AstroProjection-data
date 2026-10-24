# Coordinate Systems

A position on the sky can be written several ways. The app stores everything as a unit vector in ICRS and converts when it draws or labels.

## Equatorial: RA and Dec (ICRS / J2000)

Built on the Earth's equator projected onto the sky. **Declination** (Dec, δ) is the angle north (+) or south (−) of the celestial equator, −90° to +90°. **Right ascension** (RA, α) is the angle east along the equator from the March equinox, written 0–360° or 0h–24h (1h = 15°).

As a unit vector: `x = cos δ cos α`, `y = cos δ sin α`, `z = sin δ`. Back: `α = atan2(y, x)` (mod 360°), `δ = asin z`.

The app's catalogs and image solutions are in **ICRS**, which for our purposes equals mean J2000 equatorial coordinates (they differ by about 20 milli-arc-seconds). Stars therefore never move in the catalog; the sky *turns* underneath them.

## Horizontal: Alt and Az

Built on *your* horizon. **Altitude** is the angle above the horizon (0° horizon, 90° zenith). **Azimuth** is measured from north through east (N 0°, E 90°, S 180°, W 270°).

### Equatorial to horizontal, with formulas

Given the local sidereal time `LST` (see [Time](Time.md)) and latitude `φ`, form the **hour angle** `H = LST − α` (mod 360°): how far west of the meridian the object is. Then:

```
sin(alt) = sin δ sin φ + cos δ cos φ cos H
tan(az)  = -sin H cos δ  /  (sin δ cos φ - cos δ sin φ cos H)      (use atan2, az from north through east)
```

### The way the app does it: vectors

The app avoids angles. For the observer it builds three unit vectors, expressed in ICRS: **zenith** `Z`, **north** `N` and **east** `E`. In the frame of date (equator and equinox of the moment):

```
Z = (cos φ cos LST,  cos φ sin LST,  sin φ)
N = (-sin φ cos LST, -sin φ sin LST, cos φ)
E = N × Z
```

These are rotated from the frame of date back to ICRS with the transpose of the IAU 2006/2000B precession-nutation matrix `B(T) = N(T) P(T)`. Then, for a star's ICRS vector `p`:

```
alt = asin(p · Z)
az  = atan2(p · E, p · N)    (mod 360°)
```

Going the other way, `p = cos(alt) sin(az) E + cos(alt) cos(az) N + sin(alt) Z`. This route handles precession (about 50″ per year) and nutation (up to about 17″) automatically.

### Worked example

Vega (α = 18h 36m 56.3s = 279.235°, δ = +38° 47′ 01″ = +38.784°) seen from Chicago (φ = 41.8781°, λ = −87.6298°) at 2026-10-23 03:00 UTC.

- From [Time](Time.md): `LST = 348.920°`.
- `H = 348.920 − 279.235 = 69.686°`.
- `sin(alt) = sin 38.784° sin 41.878° + cos 38.784° cos 41.878° cos 69.686° = 0.6267×0.6678 + 0.7792×0.7443×0.3475 = 0.4185 + 0.2015 = 0.6200`, so **alt = 38.29°**.
- `az = atan2(−sin H cos δ, sin δ cos φ − cos δ sin φ cos H) = atan2(−0.7307, 0.4664 − 0.1808) = atan2(−0.7307, 0.2856)`, so **az = 291.35°** (a little north of west).

(This hand calculation skips precession and nutation, which would move the result by about 0.3°. The app includes them.)

## Ecliptic coordinates

Built on the plane of the Earth's orbit. **Ecliptic longitude** λ runs east along the ecliptic from the March equinox, **ecliptic latitude** β is the angle north or south of it. The plane is tilted from the equator by the **obliquity** ε.

The app uses the mean obliquity at J2000, **ε₀ = 84381.406″ = 23.4392794°** (IAU 2006). The obliquity changes by about 47″ a century; the app's own `mean obliquity` function `ε(T) = 84381.406 − 46.836769 T − 0.0001831 T² + 0.0020034 T³ − …` is used for nutation, while ecliptic positions use the fixed J2000 value because every other coordinate is J2000 too.

### Ecliptic to equatorial

A rotation about the x axis (the shared equinox direction) by ε:

```
x_eq =  x_ec
y_eq =  y_ec cos ε - z_ec sin ε
z_eq =  y_ec sin ε + z_ec cos ε
```

with `(x_ec, y_ec, z_ec) = (cos β cos λ, cos β sin λ, sin β)`. The inverse flips the sign of ε:

```
y_ec =  y_eq cos ε + z_eq sin ε
z_ec = -y_eq sin ε + z_eq cos ε       λ = atan2(y_ec, x_ec),  β = asin z_ec
```

![The sphere with the equator, the ecliptic and Vega in both systems](img/ecliptic-to-equatorial.svg)

### Worked example

Vega again: `(x, y, z) = (cos δ cos α, cos δ sin α, sin δ) = (0.1250, −0.7692, 0.6267)`.

- `y_ec = −0.7692 × 0.9175 + 0.6267 × 0.3978 = −0.7057 + 0.2493 = −0.4564`
- `z_ec = +0.7692 × 0.3978 + 0.6267 × 0.9175 = 0.3060 + 0.5750 = 0.8810`
- `λ = atan2(−0.4564, 0.1250) = 285.32°`, `β = asin 0.8810 = 61.73°`.

Vega sits about 62° north of the ecliptic, which is why the planets never come near it.

## Galactic coordinates

The app does not use galactic coordinates; the Milky Way is not drawn from them. If they are added later, they are one more fixed rotation from ICRS (the north galactic pole is at α = 192.859°, δ = +27.128°).

## Converting between systems at a glance

```
ecliptic  <-- R_x(ε) -->  equatorial (ICRS)  <-- B(T)ᵀ, then LST, φ -->  horizontal
```

**In the code:** `math/frames.rs` (`TopocentricBasisIcrs::compute`, `icrs_to_altaz`, `altaz_to_icrs`, `icrs_to_jnow_matrix`, `iau2006_mean_obliquity`), `math/spherical.rs` (`radec_to_unit_f64`, `unit_to_radec_f64`), `ephemeris/vsop87.rs` (`ecliptic_j2000_to_icrs`).
