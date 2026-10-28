# AstroProjection Concepts Wiki

How the numbers and pictures in AstroProjection are produced, with the formulas and a worked example for each. It is written for a curious amateur astronomer who is comfortable with trigonometry and vectors. Each page ends with an **In the code** note saying where the formula lives.

| Page | What it answers |
|---|---|
| [Time](Time.md) | Why one instant has several clocks: UTC, UT1, TT, Julian date, sidereal time |
| [Coordinate Systems](Coordinate-Systems.md) | RA/Dec, Alt/Az and ecliptic coordinates, and how to convert between them |
| [The Ecliptic Line](The-Ecliptic-Line.md) | How the yellow ecliptic line is calculated and drawn |
| [Solar System Positions](Solar-System-Positions.md) | Where the Sun, Moon and planets are; phases and magnitudes |
| [Rise, Transit and Set](Rise-Transit-Set.md) | When an object rises, crosses the meridian and sets, and how the info card finds the next ones |
| [Projections](Projections.md) | How directions become screen points: the flat/stereographic sky family, and why planets are drawn orthographically |
| [Saturn's Rings](Saturns-Rings.md) | Ring tilt `B` for an observer on Earth, its position angle, per-pixel ring radius, occlusion and shadow |
| [Jupiter's Great Red Spot](Jupiters-Great-Red-Spot.md) | Where the spot is on the planet and when it crosses the middle |
| [Plate Solving](Plate-Solving.md) | How a photo is placed on the sky: centroids, quad hashing, tiers, verification |

## Conventions used throughout

- Angles are in degrees unless a formula says radians. `atan2(y, x)` is the two-argument arctangent.
- Positions on the sky are unit vectors in the **ICRS** frame (essentially J2000 equatorial): `x` towards RA 0h, Dec 0, `z` towards the north celestial pole. `(RA a, Dec d)` becomes `(cos d cos a, cos d sin a, sin d)`.
- Every worked example uses real numbers and can be checked with a calculator or a few lines of Python.
- The app favours *visual* accuracy (a fraction of a degree to arc-seconds) over observatory-grade precision. Each page says what is simplified.
