# Tree-Line Horizons

**Status: planned, not built yet.** This page describes the design of the tree-line module (plan item TL-1). The method was tried on recordings from a phone in November 2026, and the numbers below come from that trial. The page will be updated as the module is built.

Most planning tools give an object's altitude above a flat horizon. At a clearing ringed by trees, what matters is the altitude above the *trees*. A target at 30° is still hidden if a pine at its azimuth reaches 35°. The tree-line module answers two questions:

1. **Where are the trees?** You scan the tree line once per observing site with the phone's camera. The app turns the scan into a **horizon profile**: the tree line's altitude for every quarter degree of azimuth (1440 values).
2. **When is an object clear?** For an object, a date and a site, the app lists the **windows**: the times when the object stays at least a margin (5° by default) above the tree line, in darkness, for example "Above tree line 20:42–21:17", in the site's local time. The times can be typed into a telescope's plan, such as the Seestar S50's.

## The scan

You stand at the telescope's spot and turn the phone through two full circles, about 8–10° per second. The camera points about 30° up on the first turn and about 65° up on the second. The phone records:

- **Frames** from the main camera, five per second, at full resolution (1080 × 1920).
- **Gyroscope** samples (rotation rate) and **gravity** samples (which way is up), each 100 times a second.

Frames and samples carry timestamps on one clock, so every frame can be paired with the phone's orientation at that moment. The frames are processed as they arrive and then thrown away; only the result is kept.

The best results come from turning the phone about itself, not about your body. A phone held 40 cm in front of you moves on a 40 cm circle, and a tree 15 m away then shifts by up to 1.5°. A phone clamp on the telescope's tripod avoids this.

## Which way is the phone pointing?

### Tilt and relative heading: the gyroscope

The gyroscope measures the rotation rate `ω` (radians per second, about the phone's own axes). The orientation is a rotation matrix `R` that takes a direction in phone coordinates to a direction in the world (east, north, up). Each sample turns it a little:

```
θ = ω · Δt,   R ← R · Rot(θ/|θ|, |θ|)
```

`Rot(axis, angle)` is the right-handed rotation about an axis (Rodrigues' formula). Gyroscopes have a small bias, so the tilt slowly drifts. The gravity sensor knows where "up" is, so each sample nudges `R` back towards it by 2% of the remaining tilt error. It does this only while the phone turns slower than 1 rad/s, because the gravity sensor lags fast swings.

That fixes the tilt, but not the **heading**. The heading starts at an arbitrary value (the app simply calls the first view "north") and drifts slowly, by about 0.1–0.9° over 30 s in the trial.

### Why not the compass?

The magnetometer was tried and failed. In the trial it was off by 5–67° from the true heading, and by different amounts on two turns of the same scan. Metal, cars and the phone's own electronics disturb it. A better magnetic model would fix the declination (a few degrees) but not these local errors. The compass is never used for the heading.

### Absolute heading: the Sun

The Sun's direction at any instant and place is known precisely. So the scan must catch the Sun in view at least once. For each frame:

1. Compute the Sun's true altitude and azimuth for that instant (see [Solar System Positions](Solar-System-Positions.md)), with refraction for the site's air pressure.
2. Look for saturated pixels (all three channels above 248) whose altitude, from the phone's tilt, is within 4° of the Sun's. No azimuth window is used: the heading may be off by any amount.
3. Take the centroid of that blob. The bare solar disc is only half a degree across; the detection relies on the camera's glare around it, at least 15 pixels on a copy of the frame scaled to a focal length of 430 px.
4. Turn the centroid into an altitude and azimuth (next section) and record the **heading error** `e = measured azimuth − true azimuth`.

Sightings less than 3 s apart form a **pass** (one crossing of the Sun through the view). A pass counts if it lasts at least a second and its altitude agrees with the Sun's within 1.5°. All passes of one scan must agree on `e` within 2°; otherwise something else (glare on a wall, a bright cloud) was taken for the Sun, and no heading is used.

Between passes, `e(t)` is interpolated linearly in time, and held constant before the first and after the last. The corrected orientation turns the world about the vertical:

```
R'(t) = Rot(ẑ, +e(t)) · R(t)
```

Azimuth runs clockwise seen from above (north through east), while `Rot(ẑ, +e)` turns anticlockwise, so this *lowers* every azimuth by `e`. That is what removes the error.

A scan that never saw the Sun has no absolute heading. It can still be kept: in the editor you rotate it against a landmark whose azimuth you know.

## From pixel to direction

The camera is modelled as a pinhole with focal length `f` in pixels. For pixel `(u, v)` in a frame `W` wide and `H` high, with `v` counting downwards, the ray in phone coordinates is

```
d = normalise(u + 0.5 − W/2,  −(v + 0.5 − H/2),  −f)
```

(the back camera looks along the phone's −z axis). The world direction is `w = R' · d`, and

```
altitude = asin(w_z),   azimuth = atan2(w_x, w_y)
```

with `x` east, `y` north and `z` up. The focal length comes from the phone's lens data where it reports it. The trial's main lens worked out at 1290 px at 1080 × 1920, close to the nominal value for its 23 mm-equivalent lens (about 1276 px). As a check, the app also fits `f` from the Sun sightings. A wrong value (1140 px) put the Sun 2.4° low whenever it sat off-centre.

Lens distortion bends rays towards the frame edges. The app asks the phone to correct it where the phone can; otherwise it is not modelled. The outer 5% of the width and 4% of the height are dropped.

## Sky or tree?

Each pixel is first classified by colour:

```
sky = (B > R + 12  and  B > G − 5  and  B > 60)  or  (min(R,G,B) > 175  and  max − min < 45)
```

The first part is blue sky, the second white cloud or glare. Raising the `B > 60` limit to 100 made the dark blue near the zenith count as "tree".

Colour alone is not enough. Sky pixels have a blueness `B − R` of about +80, tree pixels about −30, so single pixels separate well. But the top of a crown is mostly holes: in the 4° below the tree line about 70% of the pixels are sky seen through the branches. Video also stores colour at half resolution, so a twig one or two pixels wide takes the sky's colour. So a second rule marks a pixel as an obstruction when it is clearly **less blue than the sky next to it**:

```
obstruction = not sky   or   (B − R) < max(B − R within about 1.8°) − 30
```

That window is 41 × 41 pixels at `f` = 1290.

## Votes and the tree line

Every classified pixel is projected to the sky and **votes** into a grid of quarter-degree cells (480 altitude rows from −30° to 90°, 1440 azimuth columns). Each cell counts how often it was seen (`n`) and how often as an obstruction (`obs`). A cell counts as seen once `n ≥ 3`.

The heading is final only after the scan, but its correction turns the sky about the vertical, which only shifts azimuth. So the votes go into one grid per 15 seconds of scanning, in the provisional azimuth. At the end each grid is shifted by its own correction and the grids are summed. The heading drifts by about 0.1° within 15 s, so nothing has to be stored per frame.

Per azimuth column, with `frac = obs / n`:

1. Start at the lowest seen cell. It must be an obstruction (`frac ≥ 3%`), or the column is undefined.
2. Walk up while cells are obstructions. Gaps (sky or unseen) up to 4° are allowed inside a crown; a longer gap ends the walk.
3. The column is valid only if sky follows above the run. Trees that reach beyond the upper turn leave the column undefined, and the app asks for a higher turn.
4. The tree line is the top edge of the last obstruction cell. A running maximum over ±0.5° smooths it.

The 3% threshold looks low, but it is the point: over a degree or two, a crown that blocks even a little of the view should count. Because frames disagree slightly (focal length, tilt), the vote blurs the edges by about a degree. That puts the line on the safe side over thin twigs.

**How good is it?** At four spots not used to choose the values, the owner traced the tree line by hand on a sharp panorama. The detected line was a median −0.5° to +0.2° from the trace, with at most 5% of azimuths more than 5° too low. The owner had traced with a margin of 0.5–2.5° above the crowns, so the detector sits about that much above the real crown tops. Two scans of the same spot agreed to a median of 0.75–1.0°.

## The sharp panorama and the editor

For checking and correcting the line, the app builds a panorama at 0.1° per pixel (3600 pixels around). Each panorama pixel takes its colour from the **one** frame that saw that direction nearest its centre, and prefers frames taken while the phone turned slowly:

```
score = −((u − W/2)/(W/2))² − ((v − H/2)/(H/2))² − 0.5 · min(|ω|, 1 rad/s)
```

Averaging all frames instead blurs every edge by a degree or two. The best-frame composite shows single branches.

The detected line is drawn on it as a red dashed line through red dots where its slope changes. The dots come from simplifying the 1440 values with the Ramer–Douglas–Peucker algorithm at a tolerance of 0.25°. You can zoom in, drag a dot, or press on the line between dots to insert one. Draw the line **on** the crown tops, not above them: the app adds the margin itself. The accepted profile records which parts you changed.

## Windows

For one night (local noon to local noon in the site's time zone) the app steps through time in steps of about a minute, small enough that the object moves at most one quarter-degree azimuth column. At each step it compares the object's apparent altitude `h` with the tree line at the object's azimuth `A`:

```
clear(t) = dark(t)  and  h(t) > profile(A(t)) + margin
```

"Dark" means the Sun is below −18° (or −12°, as a setting). Each change of `clear` is narrowed down to the second by bisection, as for [rise and set](Rise-Transit-Set.md). A window must hold for its whole length: an object that dips behind one tall tree for a minute gives two windows. Windows shorter than 30 minutes are greyed out. When a night has none, the app offers the next date that has one.

**How many minutes is a degree?** For a star, the altitude changes at

```
dh/dt = 15.041°/h · cos φ · sin A
```

where `φ` is the latitude and `A` the azimuth (north through east). So an object is slowest near the meridian and fastest due east or west. A degree of error in the profile moves a window's start or end by `1° / (dh/dt)`.

## Worked example

**A Sun sighting.** The camera points level, at a provisional azimuth of 230.0°, with the phone upright. The glare centroid is at pixel `(700, 600)` of a 1080 × 1920 frame, with `f = 1290`:

```
d ∝ (700.5 − 540,  −(600.5 − 960),  −1290) = (160.5,  359.5,  −1290)
altitude = atan2(359.5, √(160.5² + 1290²)) = 15.46°
azimuth offset = atan2(160.5, 1290) = +7.09°   →   measured azimuth 237.09°
```

The ephemeris puts the Sun at altitude 15.4°, azimuth 200.0° at that instant. The altitudes agree (0.06°), so this is the Sun, and the heading error is `e = 237.09° − 200.00° = 37.09°`. Every azimuth of that scan is lowered by about 37°.

**Between two passes.** Passes at 20 s (`e` = 37.1°) and 70 s (`e` = 37.8°) give, for a frame at 45 s,

```
e(45 s) = 37.1° + (37.8° − 37.1°) · (45 − 20)/(70 − 20) = 37.45°
```

**Minutes per degree.** At latitude 50°, an object rising in the east-southeast (`A` = 120°) climbs at `15.041 · cos 50° · sin 120° = 8.37°` per hour, so a 1° error in the tree line shifts the window by about 7 minutes. The trial's repeat scans differed by up to 3° at the 90th percentile, about 21 minutes there. The 5° margin covers that, and it applies to the object's centre, not the whole field of view.

## What is simplified

- The tree line is measured from where the phone was. Turning about your body moves the phone by arm's length, which can shift near trees by 1–2° between scans.
- Lens distortion is not modelled unless the phone corrects it.
- White clouds touching the treetops count as trees (safe: windows come out shorter). Narrow sky gaps between crowns can read too low, and bare deciduous crowns are found through their middle; the editor is there for those.
- The detection values (3% threshold, 4° gaps, the colour limits) were chosen on two recordings and confirmed at four other spots, all in daylight with the Sun low. Other skies, such as the deeper blue at high altitude, may need checking.
- The windows use the object's centre and a fixed margin; the telescope's field of view is not modelled.

**In the code:** planned as the `treeline` module of `astro-core` (scan session, attitude, Sun anchoring, classifier, vote grid, profile, panorama, editor, window search), reached by every app through `astro-ffi`. The Sun position uses `ephemeris/vsop87.rs` (`compute_sun_ephemeris`) and `math/frames.rs` (`TopocentricBasisIcrs`, `atmospheric_refraction_deg`). The trial's Python scripts are the reference implementation.
