# Licensing, Long-Term Risk, and Client Apps

Part of the [AppFlowy AGPL build design](./appflowy-agpl-build-design.md).

## Can AppFlowy Change the Licence?

Yes. AppFlowy requires all contributors to sign a CLA, granting them the right to relicense contributions, which is standard open-core practice. However, all source and releases prior to any licence change remain permanently available under the original AGPL. Any version forked today stays AGPL forever. A relicensing would only affect new upstream commits.

## Current Trajectory

A GitHub issue raised in February 2026 noted that released binaries cannot be reproduced from the public AGPL source. AppFlowy confirmed they are operating a closed commercial fork alongside the AGPL repo, without disputing the compliance concern. The Flutter client repo exhibits the same pattern: source currently at `v0.11.4` while release tags run to `v0.12.5`. Development is consolidating into private repos with periodic public merges.

This is the standard path toward either a full relicensing or an increasingly hollow public repo. If AppFlowy relicenses, the last clean AGPL commit becomes the pinned version. Falling off main after 12 to 18 months would be noticeable given the development pace. **Evaluate Outline plus Plane as a fallback hedge.**

## Client Apps

The Flutter client (`AppFlowy-IO/AppFlowy`, AGPL) covers iOS, Android, desktop, and web from a single codebase, with CI workflows for both mobile platforms.

### Contingency: Publishing Our Own Build

If official apps become incompatible with a pinned server version:

- **Android:** Build APK/AAB from source, sign with GPO's key, distribute via Play Store or sideload.
- **iOS:** Requires Apple Developer account ($99 USD/year). Distribute via TestFlight (90-day expiry, 10k user cap, adequate for volunteers) or App Store under GPO's bundle ID.

**The hedge:** Fork the Flutter client at the same commit as the server pin. AGPL requires publishing modifications only if distributing publicly; distributing to GPO volunteers does not trigger this.
