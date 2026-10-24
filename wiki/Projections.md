# Projections

A projection maps directions in space to points on a flat screen. The app uses three, for three different jobs: a **family of sky projections** for the stars and lines, **orthographic projection** for the planet disks and Saturn's rings, and an affine/tangent mapping for photos (see [Plate Solving](Plate-Solving.md)).

## 1. The sky: flat and stereographic

Let `theta` be the angle between a direction and the centre of the view. A projection says how far from the screen centre, `r`, that direction is drawn:

```
r(theta) = (1 + m) · tan( theta / (1 + m) )
```

| m | Name | r(theta) | Keeps | Costs |
|---|---|---|---|---|
| 0 | Flat (rectilinear, gnomonic) | `tan theta` | straight lines stay straight (great circles are lines) | shapes stretch by `1/cos²(theta)` towards the edges |
| 1 | Stereographic | `2 tan(theta/2)` | shapes stay true (it is *conformal*: circles stay circles) | great circles bend |

### Why two, and why a blend

Wide views are the problem. With the flat projection, a figure 45° off the centre is drawn 2× too large along the radius and 4× at 60°, so panning a wide view shears the constellations and stars turn into ovals at the edges. The stereographic projection avoids that, as in Stellarium and SkySafari, and it can show 120° across. But a narrow view (a photo, a nebula) should stay flat: straight lines and the exact pixel-to-sky mapping of an image matter more than distortion that is still tiny at 10°. So `m` follows the field of view: `m = 0` while the long side of the view is up to 30°, `m = 1` from 60°, and a smooth step in between.

### Worked example

A star 50° off the centre of a view whose vertical half-field is 20° (`r_edge` is `r(20°)`):

```
flat (m = 0):          r = tan 50°     = 1.192;  r_edge = tan 20°     = 0.364;  r / r_edge = 3.27
stereographic (m = 1): r = 2 tan 25°   = 0.933;  r_edge = 2 tan 10°   = 0.353;  r / r_edge = 2.64
```

In the stereographic case the star lands at 2.6 half-heights from the centre instead of 3.3: the wide edge of the view is compressed, and the shape of what is there is not stretched.

The camera does this arithmetic in `f64` and hands the shader the small camera-relative numbers, so stars do not jitter even at 0.01° of field.

**In the code:** `camera/projection.rs` (`Projection`, `radius`), the shared shader helper `renderer/shaders/projection.wgsl` (`project_delta`), and `CameraState::camera_relative_clip_f32` (its CPU twin, tested against the GPU).

## 2. Planets: orthographic projection

Planets and the Sun are drawn as *billboards*: a screen-space square a few dozen to a few hundred pixels wide, whose pixels are shaded as if they were looking at a sphere. That sphere is drawn with an **orthographic** projection: all view rays are parallel, and a point of the sphere `(X, Y, Z)` (units of the planet's radius, `Z` towards the viewer) lands at screen offset `(x, y) = (X, Y)`, simply dropping `Z`.

Going the other way, which is what the shader does for each pixel:

```
r² = x² + y²
inside the disk (r ≤ 1):   z = sqrt(1 - r²),   surface normal n = (x, y, z)
```

That one square root is the whole "3D globe": `n` feeds the lighting (`n · L` for the phase and the terminator), the limb darkening (`z` is the cosine of the angle from the disk centre) and, for Saturn, the ring test (see [Saturn's Rings](Saturns-Rings.md)).

### Why orthographic and not perspective

- **The planet is tiny and far.** Jupiter is about 45 arc-seconds across, Saturn's globe about 18. The globe's own depth (about 120 000 km for Saturn) is under 0.01% of its distance (about 1.4 billion km), so the perspective correction to the picture is about one part in 10 000: a few hundredths of a pixel on a 400-pixel disk. Parallel rays are exact for all practical purposes.
- **The size is chosen on the screen, not from the sky.** A real planet would be a handful of pixels at a normal field of view. The app draws it a readable size (see `solar_body_disk_radius_px`), and a billboard of fixed pixel size stays a circle at every position on the screen. Putting the planet into the sky projection of section 1 instead stretched it into an oval at the corners of a wide view (the original defect that milestone 17 fixed).
- **It makes the ring geometry simple.** In an orthographic view of a plane, a circle of radius `R` becomes an ellipse with semi-axes `R` and `R |sin B|`, where `B` is the angle between the line of sight and the plane. With perspective it would be a more complex curve.
- **Cheap:** one square root per pixel.

### Worked example

A pixel at screen offset `(x, y) = (0.6, 0.3)` planet radii: `r² = 0.45`, `z = sqrt(0.55) = 0.742`, so the normal is `(0.6, 0.3, 0.742)`. With the Sun straight behind the viewer (`L = (0, 0, 1)`), `n · L = 0.742`, so the pixel is lit at 74% of full. A pixel on the limb (`r = 1`) has `z = 0`, `n · L = 0`: dark, which is why the limb of a half-lit planet shows the terminator.

**In the code:** `renderer/shaders/solar_system.wgsl` (`z_globe`, `normal`), `renderer/geometry.rs` (`build_body_instances`, the local frame), `solar_body_disk_radius_px`.

## 3. Where each one is used

| What | Projection | Why |
|---|---|---|
| Stars, grid, ecliptic, constellation lines, labels | Sky family (`m` from 0 to 1) | wide and narrow views both look right |
| Photos placed on the sky | Tangent plane (TAN, with SIP) per image; cut into tiles | matches the WCS the image comes with |
| Sun, Moon, planets, Saturn's rings, Jupiter's spot | Orthographic on a screen-space billboard | exact for distant bodies, round at every position |
