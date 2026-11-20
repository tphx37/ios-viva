# Viva GPU

The Redmi Note 11 Pro 4G (`viva`) uses a Mali-G57 MC2 GPU. The port includes an XNU-side GPU accelerator and uses the separate [viva-gpu](https://github.com/Nik300/viva-gpu) project for the iOS Metal driver. The driver builds on Mesa/Panfrost and translates AIR shaders to Valhall.

The kernel accelerator manages GPU power, memory, jobs and synchronization; the userspace driver provides the Metal-facing implementation. IOSurface connects rendered surfaces to the display path. For the engineering history, device results and patch references, see [VIVA.md](../VIVA.md). For current project status, see [README.md](../README.md) and [NOTES.md](../NOTES.md).

This replaces the original 2026-11-03 GPU proposal, which described work before implementation and should not be treated as the current design or a compatibility guarantee. Consult the GPU project for its current implementation and limitations.
