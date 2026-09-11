# Metric support

Release 0.1.0 was validated against the supplied synthetic VibFrame metric catalogs.

## Support summary

| Item | 0.1.0 status |
|---|---:|
| Unique metric names in supplied catalogs | 104 |
| Reproducible metric names | 76 |
| Explicitly unsupported metric names | 28 |
| Supported catalog descriptors in full local regression | 430 / 430 |
| Reference comparisons in full local regression | 3,440 |
| Regression tolerance | `atol=1e-3`, `rtol=1e-3` |

The package deliberately rejects definitions it cannot reproduce from the persisted raw signals.

## Supported calculation families

The reproducible catalog definitions include:

- acceleration, velocity and displacement overall RMS;
- spectrum maxima;
- 1X/2X/3X/4X/5X peak amplitudes where defined;
- RMS frequency/order bands;
- harmonic-family RMS;
- bearing-frequency families (`BPFI`, `BPFO`, `BSF`, `FTF`) where catalog definitions provide the required parameters;
- sideband RMS;
- waveform positive/negative peaks and peak-to-peak;
- waveform crest factor, kurtosis and skewness;
- envelope/demodulation metrics including `gE_rms`, `env_overall_rms` and related bearing families.

## Unsupported phase metrics

The following families require complex spectrum / phase information that is not present in the magnitude-only persisted spectra used by the current VibFrame contract:

- `peak_phase_1X*`
- `peak_phase_2X*`
- `peak_phase_3X*`
- `peak_phase_4X*`
- `peak_phase_5X*`
- `peak_phase_2xLine`
- `peak_phase_BPFI`
- `peak_phase_BPFO`
- `peak_phase_BSF`
- `peak_phase_FTF`
- `cross_phase_HA_1X`
- `cross_phase_HA_2X`
- `cross_phase_HV_1X`
- `cross_phase_HV_2X`
- `cross_phase_VA_1X`
- `cross_phase_VA_2X`

These definitions raise a controlled feature error instead of returning an approximation.

## Unsupported cross-point ratios

Two catalog names use the statistic `cross_point_ratio`, but the supplied reference metric implementation did not contain an authoritative calculation:

- `axial_radial_ratio_1X`
- `vertical_horizontal_ratio_1X`

They remain unsupported in 0.1.0.

## Runtime audit

Catalog support is ultimately descriptor-specific. Inspect an imported dataset directly:

```python
from vibframe_anndata import audit_metric_catalog

statuses = audit_metric_catalog(adata)

supported = [s for s in statuses if s.supported]
unsupported = [s for s in statuses if not s.supported]

print(f"supported: {len(supported)}")
for status in unsupported:
    print(status)
```

This is preferable to assuming that two similarly named metrics from different catalogs have identical contracts.

## Units

Catalog unit codes are preserved as supplied. The package does not silently reinterpret or convert catalog unit codes during feature calculation.
