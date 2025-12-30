# Comparison: pnpm2nix-honk vs pnpm pnpmfile

This document compares two approaches for modifying and controlling pnpm package behavior: **pnpm2nix-honk** (a Nix-based build system) and **pnpmfile** (pnpm's native hook system).

## Overview

### pnpm2nix-honk
A Nix derivation builder that processes `pnpm-lock.yaml` at Nix evaluation/build time to create reproducible, hermetic builds of pnpm-based projects. It extracts dependency information from the lockfile and rewrites it to use local Nix store paths.

### pnpmfile
A runtime hook system that allows JavaScript code in `.pnpmfile.cjs` to modify package manifests and lockfile resolution during `pnpm install` operations, before dependencies are resolved and downloaded.

## Core Purpose

| Aspect | pnpm2nix-honk | pnpmfile |
|--------|---------------|----------|
| **Primary Goal** | Build pnpm packages in Nix with reproducibility and hermeticity | Patch/modify dependency metadata during installation |
| **Environment** | Nix build system | pnpm package manager |
| **Execution Time** | Nix build time (offline, after lockfile exists) | Runtime during `pnpm install` (before resolution) |
| **Use Case** | Creating immutable, reproducible builds for deployment/distribution | Development-time dependency patching and workarounds |

## Technical Approach

### pnpm2nix-honk

**How it works:**
1. Parses `pnpm-lock.yaml` using Import From Derivation (IFD)
2. Extracts all dependency information (packages, resolutions, integrity hashes)
3. Downloads dependencies from npm registry into Nix store
4. Rewrites lockfile to replace registry URLs with `file://` paths pointing to Nix store
5. Runs `pnpm install` with the patched lockfile
6. Executes build scripts and packages output

**Key mechanisms:**
- `processLockfile` (lockfile.nix:13): Parses YAML lockfile and processes packages
- `findTarball` (lockfile.nix:29): Fetches dependencies via `fetchurl` or `fetchGit`
- `patchedLockfileYaml` (derivation.nix:186): Rewrites resolution URLs to local file paths
- `pnpm store add` (derivation.nix:131): Pre-populates pnpm store with Nix store paths
- Supports multiple dependency types: npm registry, git repos, GitHub tarballs, local tarballs

**Lockfile transformation:**
```nix
# Original lockfile has:
resolution: {tarball: "https://registry.npmjs.org/foo/-/foo-1.0.0.tgz", integrity: "sha512-..."}

# pnpm2nix transforms to:
resolution: {tarball: "file:/nix/store/xxx-foo-1.0.0.tgz"}
```

### pnpmfile

**How it works:**
1. pnpm reads `.pnpmfile.cjs` from the project root (or configured location)
2. During installation, hooks execute at specific phases
3. JavaScript functions modify package manifests in-memory
4. Modified manifests affect dependency resolution
5. Changes are reflected in `pnpm-lock.yaml` (requires lockfile deletion for already-resolved deps)

**Key mechanisms:**
- `readPackage` hook: Mutates `package.json` after parsing, before resolution
- `afterAllResolved` hook: Mutates lockfile object before serialization
- `updateConfig` hook: Modifies pnpm configuration settings
- `preResolution` hook: Runs after reading lockfiles, before resolving
- `importPackage` hook: Controls how packages are written to `node_modules`
- `fetchers` hook: Overrides fetchers for different dependency types
- `finders`: Custom query functions for `pnpm list`/`pnpm why`

**Example transformation:**
```javascript
// .pnpmfile.cjs
function readPackage(pkg) {
  if (pkg.name === 'foo') {
    pkg.dependencies.bar = '^2.0.0'; // Change bar dependency version
  }
  return pkg;
}
```

## Feature Comparison

| Feature | pnpm2nix-honk | pnpmfile |
|---------|---------------|----------|
| **Modify dependency versions** | ❌ No (uses exact lockfile versions) | ✅ Yes (via `readPackage`) |
| **Add/remove dependencies** | ❌ No | ✅ Yes (via `readPackage`) |
| **Modify lockfile structure** | ✅ Yes (rewrites resolution URLs) | ✅ Yes (via `afterAllResolved`) |
| **Change registry URLs** | ✅ Yes (to local file:// paths) | ✅ Yes (via resolution rewriting) |
| **Override configuration** | ✅ Yes (via Nix function parameters) | ✅ Yes (via `updateConfig` hook) |
| **Custom fetchers** | ✅ Yes (git, tarball, GitHub) | ✅ Yes (via `fetchers` hook) |
| **Build-time patching** | ✅ Yes (Nix-level) | ❌ No |
| **Runtime patching** | ❌ No | ✅ Yes |
| **Hermetic builds** | ✅ Yes (Nix sandbox) | ❌ No (network access) |
| **Reproducibility** | ✅ Yes (content-addressed storage) | ⚠️  Partial (depends on lockfile) |
| **Workspace support** | ✅ Yes (explicit component list) | ✅ Yes (native) |
| **Offline installation** | ✅ Yes (after Nix fetch) | ❌ No (requires initial fetch) |
| **Language** | Nix expression language | JavaScript/CommonJS |

## When to Use Each

### Use pnpm2nix-honk when:

1. **Reproducible builds are critical**
   - Building production containers/artifacts
   - CI/CD pipelines requiring determinism
   - Security-conscious environments needing provenance

2. **Offline/air-gapped builds needed**
   - No network access during build
   - All dependencies pre-fetched and verified

3. **Integration with Nix ecosystem**
   - Already using NixOS or Nix package manager
   - Want to leverage Nix's dependency management
   - Need to mix pnpm packages with other Nix packages

4. **Immutable deployments**
   - Content-addressed storage requirements
   - Need to guarantee exact same output every time

5. **Custom build phases**
   - Complex build requirements beyond `pnpm install && pnpm build`
   - Need fine-grained control over build environment

**Limitations:**
- Requires Nix knowledge
- Import From Derivation (IFD) can be slow
- Not suitable for day-to-day development
- Changes require Nix rebuild
- Cannot modify dependency code (only build process)

### Use pnpmfile when:

1. **Development-time workarounds needed**
   - Patching broken dependencies temporarily
   - Forcing specific versions due to peer dependency conflicts
   - Working around ecosystem issues

2. **Dependency manifest corrections**
   - Adding missing peerDependencies
   - Fixing incorrect dependency declarations
   - Removing problematic dependencies

3. **Monorepo/workspace customization**
   - Custom resolution logic for workspace protocols
   - Sharing configuration across projects via config dependencies

4. **Testing/experimentation**
   - Trying different dependency versions
   - Testing compatibility scenarios
   - Quick prototyping without rebuild overhead

5. **Runtime package management**
   - Normal development workflows
   - Hot-reloading and incremental changes
   - Collaborative development (team uses pnpm)

**Limitations:**
- Requires deleting lockfile to re-resolve modified packages
- Mutations not saved to filesystem (except via `pnpm patch`)
- Cannot prevent script execution via `readPackage`
- Changes only apply during installation phase
- Network access still required for fetching

## Composition & Interaction

These approaches are **not mutually exclusive**. Potential combinations:

1. **pnpmfile first, pnpm2nix for builds:**
   - Use pnpmfile during development for quick patches
   - Commit the resulting `pnpm-lock.yaml`
   - Use pnpm2nix-honk to build from the patched lockfile in CI/production

2. **Config dependencies + Nix:**
   - Use pnpm's config dependencies to share .pnpmfile.cjs across repos
   - Use pnpm2nix-honk to build those repos reproducibly

3. **Complementary concerns:**
   - pnpmfile: Modify *what* gets resolved
   - pnpm2nix: Control *how* it gets built and deployed

## Philosophical Differences

| Dimension | pnpm2nix-honk | pnpmfile |
|-----------|---------------|----------|
| **Mutability** | Immutable (functional) | Mutable (imperative) |
| **Trust model** | Zero trust (verify all inputs) | Trust registry + integrity hashes |
| **Determinism** | Guaranteed (content-addressed) | Best-effort (lockfile-based) |
| **Scope** | Build system integration | Package manager feature |
| **Abstraction level** | Infrastructure-as-code | Application-level hooks |
| **Learning curve** | Steep (Nix) | Gentle (JavaScript) |

## Performance Characteristics

### pnpm2nix-honk
- **Initial setup:** Slow (IFD evaluation, downloading all deps to Nix store)
- **Incremental builds:** Fast if inputs unchanged (Nix caching)
- **Cache sharing:** Excellent (binary cache infrastructure)
- **Evaluation overhead:** High (parsing YAML, IFD)
- **Network usage:** One-time (fetch to Nix store)

### pnpmfile
- **Initial setup:** Fast (just install)
- **Incremental builds:** Fast (standard pnpm behavior)
- **Cache sharing:** Good (pnpm store)
- **Evaluation overhead:** Low (JavaScript execution)
- **Network usage:** Per install (unless store is populated)

## Real-World Examples

### pnpm2nix-honk Example
```nix
# Building a pnpm workspace with Nix
mkPnpmPackage {
  src = ./.;
  workspace = ./.;
  components = ["packages/frontend" "packages/backend"];
  pnpmLockYaml = ./pnpm-lock.yaml;
  pnpmWorkspaceYaml = ./pnpm-workspace.yaml;
  nodejs = pkgs.nodejs_20;
  pnpm = pkgs.nodejs_20.pkgs.pnpm;
  noDevDependencies = true;
  installEnv = { PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD = "1"; };
}
```

### pnpmfile Example
```javascript
// .pnpmfile.cjs - Fix broken dependency
module.exports = {
  hooks: {
    readPackage(pkg, context) {
      // Workaround: foo@1.x has wrong peer dependency
      if (pkg.name === 'foo' && pkg.version.startsWith('1.')) {
        pkg.peerDependencies = {
          ...pkg.peerDependencies,
          react: '^18.0.0' // was incorrectly ^17.0.0
        };
        context.log('Fixed foo peer dependency');
      }

      // Remove problematic optional dependency
      if (pkg.optionalDependencies?.['fsevents']) {
        delete pkg.optionalDependencies.fsevents;
      }

      return pkg;
    },

    afterAllResolved(lockfile, context) {
      // Custom lockfile manipulation
      return lockfile;
    }
  }
};
```

## Conclusion

**pnpm2nix-honk** and **pnpmfile** serve different purposes in the package management lifecycle:

- **pnpmfile** is a *development tool* for patching dependency metadata during the pnpm installation process
- **pnpm2nix-honk** is a *build tool* for creating hermetic, reproducible artifacts from pnpm projects using Nix

Choose based on your primary concern:
- Need reproducibility/immutability? → pnpm2nix-honk
- Need quick development patches? → pnpmfile
- Need both? → Use pnpmfile in development, pnpm2nix-honk in production

The approaches complement rather than compete with each other.

## Sources

- [pnpm2nix-honk README](README.md)
- [pnpm2nix-honk derivation.nix](derivation.nix)
- [pnpm2nix-honk lockfile.nix](lockfile.nix)
- [.pnpmfile.cjs | pnpm](https://pnpm.io/pnpmfile)
- [Config Dependencies | pnpm](https://pnpm.io/config-dependencies)
- [`afterAllResolved` pnpmfile hook · Issue #1088 · pnpm/pnpm](https://github.com/pnpm/pnpm/issues/1088)
- [Why package managers need hook systems | by Zoltan Kochan | pnpm | Medium](https://medium.com/pnpm/why-package-managers-need-hook-systems-b8125d8b3dc7)
