# Vendored OptiScaler

This directory contains the OptiScaler source used by the P5R frame-generation build. It is
vendored so the standalone repository and its Windows CI can build Luma and OptiScaler together.

- Upstream: https://github.com/optiscaler/OptiScaler
- Upstream commit: `a1e0e19335c1da2366d12ee2fdb9276daec7d080`
- License: GPL-3.0 (see `LICENSE`)

Only the upstream `external/` files required to build and package this P5R integration are
included as regular tracked files, so CI does not need submodule credentials. In particular, the
large FidelityFX SDK sample trees and FSR4 signed runtime DLLs are omitted; the Luma panel does
not expose FSR4 outputs. No upstream `.git` directories are vendored.

The Release post-build step in `OptiScaler/OptiScaler.vcxproj` was adjusted to match this trimmed
dependency set: it creates `OptiScaler\Licenses` explicitly and no longer copies the omitted FSR4
signed DLLs. Upstream's `.gitmodules` is retained for provenance only; CI builds from the tracked
files and does not initialize submodules.
