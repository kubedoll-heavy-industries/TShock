# KubeDoll TShock fork — changes vs upstream

Upstream: [Pryaxis/TShock](https://github.com/Pryaxis/TShock).
Fork base: tag `v5.2.4` (commit `952a6685`).
Branch: `kubedoll/5.2.4`.

This fork exists to ship a TShock 5.2.4 build whose deps.json is free of
known-vulnerable NuGet packages. All changes are *minimal* and aim to be
upstreamable as a single PR. If/when upstream merges them, this fork goes away.

## CVE fixes

### CVE-2024-0057 — NuGet.Packaging X509 cert validation bypass (CRITICAL)

- **Advisory:** [GHSA-68w7-72jg-6qpp](https://github.com/advisories/GHSA-68w7-72jg-6qpp)
- **Patched versions:** `5.11.6, 6.0.6, 6.3.4, 6.4.3, 6.6.2, 6.7.1, 6.8.1`
- **Upstream state:** `TShockPluginManager.csproj` pinned `NuGet.Packaging` at `6.3.4`
  (renovate bot, commit `af969898`), but `NuGet.Resolver` stayed at `6.3.1`, which
  drags `NuGet.Packaging 6.3.1` back in via the same-major-minor constraint NuGet
  enforces across the `NuGet.*` family. Net effect: published `TShockAPI.deps.json`
  ships `NuGet.Packaging 6.3.1` and Trivy flags the CRITICAL.
- **Fix:** bump `NuGet.Resolver` `6.3.1 → 6.3.4`, add an explicit `NuGet.Common 6.3.4`
  pin to keep the transitive graph uniform.

### CVE-2023-29337 — NuGet.Common / NuGet.Protocol race condition (HIGH)

- **Advisory:** [GHSA-cpx2-23pf-6w8m](https://github.com/advisories/GHSA-cpx2-23pf-6w8m)
- **Patched versions:** `5.11.5, 6.0.5, 6.2.4, 6.3.3, 6.4.2, 6.5.1, 6.6.1`
- **Upstream state:** `NuGet.Protocol` pinned at `6.3.3` (patched) but
  `NuGet.Common` resolved transitively at `6.3.1` (unpatched).
- **Fix:** bump `NuGet.Protocol 6.3.3 → 6.3.4`, add explicit `NuGet.Common 6.3.4`.

### CVE-2024-30105 — System.Text.Json DoS (HIGH)

- **Advisory:** [GHSA-hh2w-p6rv-4g7w](https://github.com/advisories/GHSA-hh2w-p6rv-4g7w)
- **Patched versions:** `6.0.10, 8.0.4`
- **Upstream state:** `System.Text.Json` not referenced directly anywhere in TShock's
  csprojs. Resolved transitively at `7.0.1` (unpatched) via `NuGet.Packaging`'s
  dependency chain.
- **Fix:** add `<PackageReference Include="System.Text.Json" Version="8.0.6" />`
  to `TShockAPI.csproj` to displace the transitive 7.0.1 resolution. **8.0.6 is
  the highest STJ version that still ships `lib/net6.0/`** — 9.x and 10.x require
  net8.0+, which TShock can't move to until OTAPI/Open-Terraria-API ships a
  net8+ build of `TerrariaServer.dll`.

## Why not a TFM bump?

TShock 5.2.4 and the bundled OTAPI binaries (`TerrariaServer.dll`, `OTAPI.dll`,
`OTAPI.Runtime.dll`, `ModFramework.dll`) all target `.NETCoreApp v6.0`. .NET 6
is EOL (Nov 2024), but the right answer at the **container** layer is to ship
the .NET 10 runtime in the image and use `DOTNET_ROLL_FORWARD=LatestMajor` —
that's done in the downstream
[kubedoll-heavy-industries/terraria-tshock](https://github.com/kubedoll-heavy-industries/terraria-tshock)
image, not in this source repo.

Bumping the source TFM to net8.0/net10.0 would require a parallel fork of
[NyxStudios/TerrariaAPI-Server](https://github.com/NyxStudios/TerrariaAPI-Server)
and [SignatureBeef/Open-Terraria-API](https://github.com/SignatureBeef/Open-Terraria-API)
(no upstream net8+ release since 2022-11-17). That's out of scope here.

## Diff summary

```
TShockPluginManager/TShockPluginManager.csproj  # NuGet.Protocol 6.3.3→6.3.4, Resolver 6.3.1→6.3.4, +Common 6.3.4
TShockAPI/TShockAPI.csproj                       # +System.Text.Json 8.0.6 (override transitive)
CHANGES.md                                       # this file
```

Two `csproj` edits, ~6 added lines plus comments. Zero source-code changes; no
call-site fix-ups needed (these are same-major or backwards-compatible bumps).
