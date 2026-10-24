# Plate Solving

**Plate solving** finds where on the sky a photograph points: its centre, scale, rotation and whether it is mirrored. AstroProjection does it on the device, offline, with no position hint needed (a *blind* solve). The result is a **WCS** (world coordinate system): a map from image pixels to RA and Dec.

The pipeline: find stars in the image → form 4-star shapes → look the shapes up in a catalog of pre-computed shapes → fit a WCS to each candidate → *verify* it against the real catalog before believing it.

## 1. Star centroids

Find each star's centre to a fraction of a pixel in the image's luminance:

1. **Background.** The median `μ` of the pixel values and the robust spread `σ` from the median absolute deviation (MAD), `σ ≈ 1.4826 · MAD`. Both ignore the stars themselves.
2. **Peaks.** A pixel counts if it is the strict maximum of its 3×3 neighbourhood and exceeds `μ + k σ` (about `k = 5`; the stars are sampled from at most 20 000 pixels, so this is fast).
3. **Centre of mass** over the 5×5 window around the peak, weighting by brightness above the background:

```
x_c = x + Σ (Δx · w) / Σ w        y_c = y + Σ (Δy · w) / Σ w
```

4. **Keep the brightest.** Sort by total flux, keep the top stars (only the best 24 form quads in a blind solve, 28 when a position hint exists).

## 2. Quad hashing

A *quad* is four stars. Its **hash** is a shape code that does not change with image position, rotation or scale, so a photo and a catalog can be compared without knowing either.

1. Find the two stars `A`, `B` that are farthest apart; `L = |AB|`.
2. Build a coordinate frame with `A` at the origin and `B` at `(1, 0)`: `u = (B − A) / L`, `v = (−u_y, u_x)`.
3. Express the other two stars `C`, `D` in this frame: `(x_C, y_C)`, `(x_D, y_D)`.
4. Make the code unique. If `x_C + x_D > 1`, swap `A` and `B` and redo it (so the pair is always nearer `A`); then order so that `x_C ≤ x_D`.

The hash is the four numbers `(x_C, y_C, x_D, y_D)`.

**Worked example.** Stars at `A = (0, 0)`, `B = (10, 0)`, `C = (3, 2)`, `D = (6, −1)` pixels. The widest pair is `AB` with `L = 10`. In its frame `C = (0.3, 0.2)`, `D = (0.6, −0.1)`. `x_C + x_D = 0.9 ≤ 1` and `x_C < x_D`, so the hash is **(0.3, 0.2, 0.6, −0.1)**. Scale the picture by 2, rotate it by 30° and shift it, and the hash is the same, which is why a quad seen in a photo can be found in a catalog.

## 3. The quad catalogs and their tiers

Pre-computed hashes live in *packs* of reference quads, each built for a range of field sizes. Searching all of them in one go would drown the right answer in false matches, so each pack is a **tier** with its own star list and time share. A frame is tried on tiers whose field matches its plausible scale, in order of cost and likelihood:

| Tier | Frames (long side) | Reference stars | Time weight |
|---|---|---|---|
| `wide` | 6° to 60° | Tycho-2, V ≤ 6 | 1 |
| `mid` | 1.5° to 9° | Tycho-2, V ≤ 9 | 2 |
| `narrow` | 0.5° to 2.2° (along the width) | Gaia | 3 |
| `gaia_mid` | 1.5° to 6° | Gaia, several passes to allow for lens distortion | 5 |

A missing pack just means a shorter list. The overall limit for one placement is 20 s.

**Scale priors.** To avoid trying every tier, the app first narrows down the **pixel scale** (arcseconds per pixel) as a *range*, never a single value, because reducers, crops and binning change it:

```
scale = 206.265 × pixel_pitch[µm] / focal_length[mm]    arcsec per pixel
```

Sources, best first: focal length and pixel pitch in the file header (±15 %), pixel pitch alone with plausible focal lengths 300-8000 mm, the catalog size of an object named in the file, and otherwise a geometric sweep in bands no wider than 1.5×. A tier is tried if its scale range overlaps the prior.

## 4. Matching and fitting

For each image quad, look up catalog quads whose four hash numbers all agree within **0.02** (buckets of width 0.04 are probed together with their neighbours). For each match the four star pairs give a candidate **TAN (gnomonic) projection**. The catalog's `(RA, Dec)` is first projected onto the tangent plane at a centre `(α₀, δ₀)`:

```
cos c = sin δ₀ sin δ + cos δ₀ cos δ cos(α − α₀)
ξ  = cos δ sin(α − α₀) / cos c
η  = (cos δ₀ sin δ − sin δ₀ cos δ cos(α − α₀)) / cos c
```

and a 2×2 linear map plus offset links pixel offsets to `(ξ, η)`:

```
[ξ, η]ᵀ = CD · [x − x₀, y − y₀]ᵀ
```

`CD` holds scale, rotation and mirroring. It is found by **linear least squares** over the matched star pairs, the centre `(α₀, δ₀)` is where the image centre maps, and the scale is `sqrt(|det CD|)`. A positive determinant means the image is mirrored.

## 5. Verification

A hash match alone proves nothing: four stars can look alike by chance, and a blind solve is never accepted on a match alone. The candidate WCS must predict the *rest* of the stars:

1. **Cheap pre-check** against the brightest reference stars (V ≲ 8 for `wide`, ≲ 10 for `mid`), so false candidates are dropped quickly.
2. **Full check.** Project every catalog star of the right area into the image. Each that lands within a **pixel tolerance** of an unused detection is an **inlier**.
3. **Accept** only if all hold:
   - the scale is inside the allowed range;
   - there are at least `min_inliers` inliers;
   - `inliers / min(expected, detected)` is at least `min_inlier_ratio`, *or* there are at least 12 inliers and at least 8× as many as chance would give. Chance matches are `N_expected × N_detected × π tol² / (W × H)`; the second rule keeps correct solutions against a deep catalog that lists far more stars than the photo shows;
   - the **trimmed RMS** residual (the best 80 % of matches, so saturated or blended stars cannot veto a solution that dozens of others confirm) is below the tolerance;
   - if a position prior exists (for example a name in the file), the solved centre is within `max(2 × field, 1°)` of it.
4. **Refine.** The tangent point is moved to the mean of the inliers and the linear map is re-fitted on all of them, which sharpens the first four-star estimate. (Lens distortion over a frame of 1.5° and more is handled by running the `gaia_mid` tier in several passes, not by a distortion polynomial in the fit.)

If a blind and a hinted solve disagree by more than 0.25° the result is flagged.

## Worked example of the arithmetic

A frame from a sensor with 2.4 µm pixels behind a 250 mm lens: `scale = 206.265 × 2.4 / 250 = 1.98 arcsec/px`. A 4000 × 3000 pixel image then spans `4000 × 1.98 / 3600 = 2.2°` on its long side. The app uses a range of about `[1.7, 2.3]` arcsec/px (±15 %), which overlaps `mid` (1.5°-9°) and `gaia_mid` (1.5°-6°) but not `wide` (6°-60°), so those two are tried first.

**In the code:** `wcs/centroids.rs` (extraction), `wcs/quad_solver.rs` (hash, TAN projection, least-squares fit), `wcs/blind.rs` (blind index, passes), `wcs/tiers.rs` (the tiers and `Solver`), `wcs/scale_prior.rs`, `wcs/placement.rs` (`verify_solution`, `refine_wcs`), `wcs/tuning.rs` (every constant above, with the reasons).
