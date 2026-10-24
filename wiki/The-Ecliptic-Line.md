# The Ecliptic Line

The yellow line you can switch on is the **ecliptic**: the Sun's apparent path through the stars over a year, and the plane close to which the Moon and planets travel. In the app it is the great circle of ecliptic latitude 0° in the J2000 frame.

## How it is calculated

Every point on the line has ecliptic latitude β = 0 and a longitude λ from 0° to 360°. In ecliptic Cartesian coordinates that is `(cos λ, sin λ, 0)`. Rotating about the x axis by the obliquity ε₀ = 23.4392794° puts it in ICRS (see [Coordinate Systems](Coordinate-Systems.md)):

```
x = cos λ
y = sin λ cos ε₀
z = sin λ sin ε₀
```

Equivalently, the ecliptic is the plane through the origin perpendicular to the **ecliptic pole** `(0, −sin ε₀, cos ε₀)`, which is at RA 18h, Dec +66.56°.

## How it is drawn

The line is 360 segments, one per degree of λ, each a pair of ICRS vertices. They are sent to the same line shader as constellation lines, which projects every vertex through the camera each frame. Because the points are in ICRS they do not need updating when time passes; only the camera moves. One degree per segment is smooth at every zoom the app offers for a circle of this size.

## Facts you can check on screen

| λ | ICRS point (x, y, z) | RA, Dec | What is there |
|---|---|---|---|
| 0° | (1, 0, 0) | 0h, 0° | March equinox (Sun on about 20 March) |
| 90° | (0, 0.9175, 0.3978) | 6h, +23.44° | June solstice (Sun on about 21 June) |
| 180° | (−1, 0, 0) | 12h, 0° | September equinox |
| 270° | (0, −0.9175, −0.3978) | 18h, −23.44° | December solstice |

The line therefore climbs to +23.44° declination in Gemini and dips to −23.44° in Sagittarius, passing through the 13 constellations of the zodiac (Ophiuchus included).

## Why the planets sit near it

The planets' orbits are within a few degrees of the Earth's orbital plane (Mercury 7°, Venus 3.4°, Mars 1.9°, Jupiter 1.3°, Saturn 2.5°), so from Earth they always appear close to this line. The Moon is within 5.1°.

**In the code:** `renderer/pipelines.rs` (`OBLIQUITY_J2000_DEG`, `generate_ecliptic_line`); shown when `show_ecliptic` is set in `renderer/context.rs`.
