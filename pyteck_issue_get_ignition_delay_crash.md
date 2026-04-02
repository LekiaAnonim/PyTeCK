## Bug: `get_ignition_delay` crashes on empty peak array

### Description

`get_ignition_delay()` in `pyteck/simulation.py` raises `ValueError: attempt to get argmax of an empty sequence` when `detect_peaks` returns no peaks. This happens when the input signal is flat or nearly flat (e.g., from a non-igniting reactor simulation).

All four ignition types are affected: `max`, `d/dt max`, `1/2 max`, and `d/dt max extrapolated`.

The function already has a safety check at the bottom:

```python
if ign_delays.size == 0:
    # ... warning + return np.array([0.0])
```

But execution never reaches it because `peak_inds[np.argmax(target[peak_inds])]` crashes first when `peak_inds` is empty.

### Minimal Reproducer

```python
import numpy as np
from pyteck.simulation import get_ignition_delay

# Flat signal — no ignition occurred
time = np.linspace(0, 0.05, 100)
temperature = np.full_like(time, 500.0)

# Crashes with: ValueError: attempt to get argmax of an empty sequence
ign = get_ignition_delay(time, temperature, 'temperature', 'd/dt max')
```

### Expected Behavior

The function should return `np.array([0.0])` (the existing "no ignition detected" sentinel), not crash.

### Root Cause

In each branch (`max`, `d/dt max`, etc.), `detect_peaks` may return an empty array when the signal has no meaningful peaks. The line:

```python
max_ind = peak_inds[np.argmax(target[peak_inds])]
```

then calls `np.argmax` on an empty slice, which raises `ValueError`.

### Suggested Fix

Add an empty-array guard after each `detect_peaks` call so that execution falls through to the existing safety check:

```python
peak_inds = detect_peaks(target, edge=None, mph=1.e-9*np.max(target))

if peak_inds.size == 0:
    ign_delays = np.array([])
else:
    max_ind = peak_inds[np.argmax(target[peak_inds])]
    ign_delays = time[peak_inds[peak_inds <= max_ind]]
```

This needs to be applied in all four branches (`max`, `d/dt max`, `1/2 max`, `d/dt max extrapolated`).

### Context

I'm using `get_ignition_delay` as a standalone function to post-process Cantera 0-D reactor output across a large parameter grid (temperature × pressure × equivalence ratio × multiple kinetic models). The grid intentionally includes conditions where ignition does not occur, which triggers this crash.

### Environment

- PyTeCK development branch (commit with `get_ignition_delay` as a standalone function)
- NumPy 1.x
- Python 3.9
