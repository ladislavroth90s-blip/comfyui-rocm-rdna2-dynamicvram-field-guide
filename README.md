# ComfyUI on Windows AMD ROCm: RDNA2 DynamicVRAM Field Notes

Public, anonymized technical notes from a reproducible troubleshooting session involving **ComfyUI + AMD ROCm + a 12 GB RDNA2 / gfx1031-class GPU + Krea2 mixed-precision models**.

This document intentionally contains **no user name, host name, local directory, motherboard, CPU model, disk layout, network data, monitor topology, device serial, account information, or exact GPU SKU**. Hardware is described only at the architecture / VRAM-class level needed to make the results useful to other AMD users.

> These are field observations, not official ComfyUI or AMD support guidance. The exact root cause of the original slowdown was not conclusively proven.

## The problem

A Krea2 mixed-precision workflow whose diffusion model staged at roughly **12.86 GB** was being run on a device reporting about **12.27 GB VRAM**. DynamicVRAM and async weight offloading were therefore necessary.

Healthy runs were typically around **16-26 s/it**. In bad runtime states, sampling could degrade to approximately:

- ~35 s/it
- ~90-100 s/it
- more than 300 s/it in an extreme case

The slowdown could occur with **0 LoRA patches attached**, so LoRA processing was not a required trigger.

## Strongest telemetry clue

One captured comparison showed a clear residency shift:

| Metric | Healthy | Severely degraded |
| --- | ---: | ---: |
| Dedicated GPU memory | ~10.47 GB | ~8.48 GB |
| Shared GPU memory | ~0.27 GB | ~2.51 GB |
| Process RAM | ~4.36 GB | ~5.22 GB |

The useful signal was not raw WDDM compute percentage. It was that roughly **2 GB left Dedicated GPU Memory and appeared in Shared GPU Memory at the same time sampling speed collapsed**.

A plausible mechanism is that an over-VRAM model normally streams correctly through DynamicVRAM, but a bad runtime state can push too much of the active working set into Shared GPU Memory, producing a slow system-RAM / PCIe path. The exact boundary could be ComfyUI DynamicVRAM, comfy-aimdo, PyTorch/ROCm, WDDM, or an interaction between them.

## Things that were tested and deprioritized

### LoRA was not the root cause

An extreme slow run occurred with `0 patches attached`. LoRAs can still change memory pressure, but they are not required for the failure.

### VRAM hardware failure became unlikely

Both Vulkan-visible AMD adapters passed the standard **memtest_vulkan** run without reported data errors at stock memory settings.

### HAGS was not active

The tested AMD drivers reported Hardware Scheduling as `AlwaysOff` / disabled, so HAGS was not an active variable.

### The secondary GPU did not add VRAM to ComfyUI

The active PyTorch/ROCm environment reported only one compute device. The older secondary adapter therefore did not act as `gpu:1` for ComfyUI and its VRAM was not pooled with the primary GPU.

## Software stack before and after

| Component | Before | After |
| --- | --- | --- |
| ComfyUI | 0.34.0 | 0.42.0 |
| comfy-aimdo | 0.4.15 | 0.5.6 |
| comfy-kitchen | 0.2.31 | 0.2.39 |
| Frontend | 1.51.9 | 1.57.1 |
| PyTorch | 2.13.0+rocm10.1.0a20260822 | unchanged for the first A/B test |
| ROCm/HIP runtime | ROCm `(7,16)` / HIP 7.16.x | unchanged for the first A/B test |

Keeping PyTorch/ROCm unchanged during the first application update was deliberate: it isolated the effect of the ComfyUI / aimdo / kitchen update from a simultaneous runtime change.

## Stable post-update launcher baseline

These were the tested flags, not a claim that they are universally optimal:

```text
--cache-none
--disable-smart-memory
--disable-pinned-memory
--enable-manager
--enable-manager-legacy-ui
--disable-triton-backend
--use-quad-cross-attention
```

DynamicVRAM remained enabled and async weight offloading used two streams.

## Windows restart hygiene

Fast Startup was disabled before the later comparison tests so that a normal Windows Restart created a fresh driver/kernel state.

Check it with:

```powershell
Get-ItemProperty `
'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power' `
-Name HiberbootEnabled |
Select-Object HiberbootEnabled
```

Disable it for testing (Administrator PowerShell):

```powershell
Set-ItemProperty `
'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Power' `
-Name HiberbootEnabled -Value 0
```

This does **not** prove Fast Startup was the bug. It removes one source of persistent driver state and makes comparisons cleaner.

## Driver hygiene

The final validation used current supported 2026 AMD driver branches for both the RDNA2 primary adapter and the older Polaris-class display adapter.

Relevant driver versions in the final test state:

- RDNA2 / gfx1031-class: `32.0.21045.16010`
- Polaris-class secondary adapter: `31.0.21925.1001`

Generic PowerShell inventory:

```powershell
Get-CimInstance Win32_PnPSignedDriver |
Where-Object { $_.DeviceName -match 'Radeon' } |
Select-Object DeviceName,DriverVersion,DriverDate |
Format-Table -AutoSize
```

## High-value diagnostic: call `/free` before rebooting

If the slowdown returns, try clearing model/memory state before restarting anything:

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "http://127.0.0.1:8188/free" `
  -ContentType "application/json" `
  -Body '{"unload_models":true,"free_memory":true}'
```

Interpretation:

| Result | What it suggests |
| --- | --- |
| `/free` immediately restores speed | ComfyUI model-manager / DynamicVRAM residency state |
| `/free` does not help, restarting ComfyUI does | process-local AIMDO / HIP / allocator state |
| restarting ComfyUI does not help, Windows Restart does | driver / WDDM / HIP virtual-memory state |

A separate public ComfyUI issue reports a similar AMD ROCm / DynamicVRAM repeated-run slowdown where `/free` restores the fast state.

## Performance observations

These are field measurements, not standardized benchmarks.

### Before the application update

- no LoRA: ~22.29 s/it, ~361 s total
- LoRA-active: ~20.73 s/it, ~334 s total
- warm fast runs: ~17.34-17.43 s/it, ~275-288 s total
- same session later: ~35.28 s/it, ~550 s total
- extreme bad state with 0 patches: ~331 s/it early in the sampling loop

### After the ComfyUI / aimdo update

Repeated heavy runs stayed stable:

- 2 LoRAs + upscale, 15 steps: ~337 s total
- 3 LoRAs + upscale, 15 steps: ~285 s total
- another 15-step run: 17.34 s/it, ~279 s total
- 4 LoRAs + harder prompt + DPM++ 2M SDE + 17 steps: stable ~17.2 s/it after initialization, ~337 s total
- identical repeat: ~345 s total, only ~2.5% slower
- same heavy workload with Euler ancestral + 17 steps: ~311 s total and stable

The important observation was **not** the absolute benchmark. It was the absence of the old multi-fold degradation pattern across repeated runs.

## Sampler observation

For this specific Krea2 + LoRA workflow, Euler ancestral was both faster and subjectively preferable to DPM++ 2M SDE. This is not a universal image-quality claim.

A useful health test is to lower step count while changing nothing else. In a healthy runtime state, seconds per iteration should stay broadly similar while total sampling time falls with the number of steps.

## VRAM integrity testing

Consumer Radeon cards do not expose an SSD-like “wear percentage” for VRAM. A practical test is data integrity under sustained access.

`memtest_vulkan` can test each Vulkan-visible GPU independently:

```text
memtest_vulkan-v0.5.0.exe
```

Verify the selected GPU in the output and allow the standard 5-minute test to finish. Any reported data error is important.

## Mixed-generation multi-GPU note

A second AMD adapter may still be useful for displays or applications using other backends. But if PyTorch reports one ROCm device, it cannot be used as a second ComfyUI compute GPU and cannot pool VRAM with the primary adapter.

Generic device check:

```powershell
<COMFY_ROOT>\python_env\python.exe -c "import torch; print(torch.__version__); print(torch.version.hip); print(torch.cuda.device_count()); [print(i, torch.cuda.get_device_name(i)) for i in range(torch.cuda.device_count())]"
```

## Recommended troubleshooting order

1. Capture the failing state before changing anything: s/it, Dedicated GPU Memory, Shared GPU Memory, clocks/fans, and patch count.
2. Call `/free` and repeat the exact workflow.
3. If still slow, restart ComfyUI and repeat.
4. If still slow, perform a real Windows Restart.
5. Verify AMD display-driver versions and HAGS state.
6. Run a VRAM integrity test at stock clocks/timings.
7. Update ComfyUI/aimdo using the distribution-specific updater.
8. Keep the ROCm/PyTorch updater separate at first so the changed layer remains identifiable.
9. Then isolate async offload, cache policy, custom nodes, and runtime version one variable at a time.

## Public references

- `patientx-cfz/comfyui-rocm`: https://github.com/patientx-cfz/comfyui-rocm
- ComfyUI startup flags: https://github.com/Comfy-Org/docs/blob/main/development/comfyui-server/startup-flags.mdx
- ComfyUI issue #16502 - AMD ROCm / DynamicVRAM repeated-run slowdown: https://github.com/Comfy-Org/ComfyUI/issues/16502
- ComfyUI issue #16878 - gfx1031 / DynamicVRAM report: https://github.com/Comfy-Org/ComfyUI/issues/16878
- `memtest_vulkan`: https://github.com/GpuZelenograd/memtest_vulkan

## Closing note

The most useful takeaway is methodological: on Windows AMD ROCm, an over-VRAM ComfyUI workload can fail as a **runtime-state / residency problem** rather than a simple “not enough VRAM” error. Large jumps in Shared GPU Memory, a suddenly underutilized/quiet GPU, and multi-fold increases in s/it are more informative than the mere presence of LoRAs or a complex prompt.

If you reproduce this behavior, a high-quality bug report should include:

- exact ComfyUI / comfy-aimdo / PyTorch / ROCm versions
- GPU architecture and VRAM class
- staged model size
- whether the slowdown happens with `0 patches attached`
- Dedicated vs Shared GPU Memory during good and bad runs
- whether `POST /free` restores performance

Those details are much more actionable than a generic “AMD is slow” report.
