# XNU builds

The `.github/workflows/xnu.yml` workflow builds the SWEET-config XNU kernel with the repository's ordered `patches/*.patch` and staging scripts, using [blacktop/darwin-xnu-build](https://github.com/blacktop/darwin-xnu-build). Viva uses this shared configuration plus MediaTek-specific patches; `sweet` and `viva` refer to different devices.

The workflow produces an `xnu-kernel` artifact. Select the correct repository, branch and run explicitly when using `gh run download`; the upstream ios-sweet repository does not contain this Viva branch. See [VIVA.md](../VIVA.md) for history and [NOTES.md](../NOTES.md) for current build and device status. A committed patch is not evidence of a successful CI build or boot.
