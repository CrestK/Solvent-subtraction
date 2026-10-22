# Solvent-subtraction

`index.html` is a single-page web app that subtracts solvent Raman bands from a spectrum, written for ultrafast (FSRS)
Raman data. Load a sample wave and a solvent (reference) wave saved from Igor Pro, pick a method from the menu, check
the fit, and download the result as an Igor wave.

**Inputs:** two waves: the sample and the solvent. They can be Igor binary waves (.ibw, versions 1, 2, 3, 5, Mac or
Windows byte order), Igor text (.itx), a packed experiment (.pxp/.pxt, every numeric wave listed by data folder) or
delimited text. For 2D waves you pick a column. The x axis is the wave scaling, the point number, or any monotonic wave of
the same length: from the same file (an x column) or from an optional x wave file (a wavenumber axis saved as its own
wave). The solvent is interpolated onto the sample's x values, so the two may have different points and ranges, and the
axis may run either way. Fit regions are typed (`770-835, 1005-1050`) or added by zooming into a band on the plot.
Options: a baseline polynomial (none to cubic) fitted with the solvent, fitting on the first or second difference
instead of the spectrum, and Savitzky–Golay smoothing of the solvent. Everything runs in the browser; nothing is uploaded.

**Methods** (the menu):

- Scale factor only: least-squares scale; peak area ratio (local straight baselines, per-region ratios listed); minimum
  roughness (smoothest result, first or second difference); minimum total variation (L1, solved exactly as a weighted
  median); manual (scale, shift, Gaussian and Lorentzian broadening and offset by hand, starting from the last fit).
- Taylor expansion (linear): S ≈ a₀R + a₁R′ + a₂R″ + a₃R‴ + B. First order (scale + shift), second order (+ broadening
  or narrowing), second order even (broadening, no shift), third order (+ asymmetry κ₃), or any chosen terms (for example
  R′ and R″ alone, to remove derivative-shaped solvent residues from a transient spectrum). The table converts the
  coefficients to δ = −a₁/a₀, σ² = 2a₂/a₀ − δ², an equivalent Gaussian FWHM and κ₃. "Iterate" shifts and broadens the
  reference by the fitted δ and σ² and refits until they converge (Gauss–Newton), which removes the small-shift and
  small-width limit of the expansion.
- Shift and broadening (nonlinear, exact): a·[R ⊗ V](x − δ) + B with no broadening, Gaussian, Lorentzian or Voigt
  broadening, or with a linear axis stretch (shift δ + ε(x − x_c), for a calibration error across the window). Shifts are
  exact (spline + cubic interpolation on a half-spacing grid), broadening is done by FFT; the nonlinear parameters come from
  a grid search and Nelder–Mead, the scale and baseline from least squares at every step.

**Output:**

- Two plots sharing one x axis: the sample with the fitted solvent (+ baseline) and the fit regions, and the result
  S − M. Drag to zoom in x, double-click to zoom out, hover to read values.
- A table of the fitted parameters with standard errors from the fit covariance, the number of points, the RMS of the
  residual in the fit regions and the roughness of the result.
- "Compare all methods": every method on the same regions and options, side by side; click a row to switch to it.
- Downloads: Igor binary (.ibw v5, double precision, the sample's x scaling, method and parameters in the wave note),
  Igor text (.itx: result, solvent fit, baseline and the x wave if one was used) or tab-delimited text. Points outside the
  solvent's x range are NaN.
- The methods and equations at the bottom of the page.

A synthetic example loads at start: cyclohexane-like solvent bands that are shifted by 0.6 cm⁻¹, broadened by a 3.0 cm⁻¹
Gaussian and scaled by 0.8 in the sample, with a made-up solute, a sloping baseline and noise.

## Use

Open `index.html` in a browser. It needs no server, no build step and no network. To publish it, serve the repository
root from any static host, for example GitHub Pages with "Deploy from a branch" set to the repository root.

## Files

| path | what |
|---|---|
| `index.html` | the app: markup, the core (`<script id="core">`: Igor readers and writers, numerics, all methods), the UI and the equations |
| `tests/validate.mjs` | checks the core inside `index.html` |

## Tests

```
node tests/validate.mjs     # about 3 s
```

- File formats: .ibw write/read round trip (NaN, ±INF, extended units, note, checksum 0); hand-built big-endian
  version 2 and 2D version 5 files; .itx with several waves, liberal names, SetScale /P, /I and plain, units, notes, a 2D
  wave and a skipped text wave, and the .itx writer; .pxp in both byte orders with data folders and superseded records;
  delimited text; content sniffing. The readers were also checked value by value against the igor2 Python package on the
  WaveMetrics TN003 sample files, and igor2 reads the .ibw files the page writes.
- Numerics: the spline against its error bound, FFT against a direct DFT, FFT broadening against the closed forms
  (Gaussian ⊗ Gaussian, Lorentzian ⊗ Lorentzian, 2e-4 and 4e-4 of the band height), interpolated shifts, least-squares
  standard errors against the textbook formula, Savitzky–Golay on a quadratic.
- Methods on synthetic spectra with known values: scale-only methods recover 0.7 (to 1e-9 for least squares); first-order
  Taylor finds a 0.3 shift to 0.005; iterated second-order Taylor and the nonlinear fit agree to 1e-4 and find δ = 0.3,
  FWHM = 2 (and δ = 3, FWHM = 2); narrowing gives σ² < 0; nonlinear shift-only (2.5), Gaussian (3.0), Lorentzian (2.0),
  Voigt and axis-stretch (ε = 0.002) fits recover their values; manual mode with the fitted values reproduces the fitted
  model exactly; a reversed x axis gives the same fit; a solvent on another grid and range works, with NaN outside it; the
  example's true scale, shift and width are recovered within the noise, and every method runs on it.
