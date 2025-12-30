# CLAUDE.md - AI Assistant Guide for pnpm2nix-honk

> Comprehensive documentation for AI assistants working on the pnpm2nix-honk codebase

## Project Overview

**pnpm2nix** is a Nix library that provides build tooling for pnpm packages. It enables building pnpm-based JavaScript/TypeScript projects reproducibly within the Nix ecosystem.

### Key Features
- Supports pnpm 10 and pnpm workspaces
- Provides `mkPnpmPackage` function for building pnpm packages with Nix
- Handles lockfile processing with Import From Derivation (IFD)
- Supports monorepo/workspace builds
- Integrates with nixpkgs ecosystem

### License
ISC License (permissive open-source)

## Repository Structure

```
pnpm2nix-honk/
├── derivation.nix       # Main mkPnpmPackage function implementation
├── lockfile.nix         # Lockfile parsing and processing logic
├── flake.nix            # Nix flake entry point and overlay definitions
├── flake.lock           # Flake input lock file
├── README.md            # User-facing documentation
├── LICENSE              # ISC license
├── .gitignore           # Git ignore patterns
├── .github/
│   ├── workflows/
│   │   └── main.yml     # CI workflow (runs checks via nzbr/actions)
│   └── renovate.json5   # Renovate dependency update configuration
└── example/             # Example project using mkPnpmPackage
    ├── default.nix      # Example Nix derivation
    ├── package.json     # Example pnpm package
    ├── pnpm-lock.yaml   # Example lockfile
    ├── index.html       # Example HTML entry point
    └── style.scss       # Example SCSS file
```

## Core Architecture

### 1. Main Entry Points

#### **flake.nix** (Primary Interface)
- Defines the flake interface for modern Nix usage
- Exports `packages.mkPnpmPackage` and `packages.example`
- Provides `overlays.default` for integrating with nixpkgs
- Uses nixpkgs 25.05 and flake-utils

#### **derivation.nix** (Core Logic)
- Implements the `mkPnpmPackage` function
- Accepts 40+ parameters for customizing builds
- Creates a `stdenv.mkDerivation` with pnpm-specific logic
- Handles both single packages and workspace builds

#### **lockfile.nix** (Dependency Resolution)
- `parseLockfile`: Converts YAML lockfile to JSON/Nix
- `processLockfile`: Processes lockfile and fetches all dependencies
- `findTarball`: Resolves various package sources (npm registry, git, tarballs)
- Patches lockfile to use local Nix store paths

### 2. Build Process Flow

```
1. Parse pnpm-lock.yaml → JSON structure
2. Extract all package dependencies from lockfile
3. Fetch each dependency tarball into Nix store
4. Patch lockfile to point to store paths (file:// URLs)
5. Run pnpm install with patched lockfile
6. Execute build scripts
7. Copy dist directories to $out
```

### 3. Key Mechanisms

#### Import From Derivation (IFD)
- `parseLockfile` uses IFD to convert YAML to JSON at evaluation time
- This is **slow** - recommend passing lockfile paths explicitly:
  ```nix
  pnpmLockYaml = ./pnpm-lock.yaml;
  ```

#### Dependency Fetching
Supports multiple source types:
- **npm registry**: Standard `registry.npmjs.org` URLs
- **git repositories**: Via `resolution.type = "git"`
- **GitHub tarballs**: Via `codeload.github.com` URLs
- **Custom tarballs**: Via `resolution.tarball` with integrity hashes

#### Workspace Support
- Detects workspace via `workspace` and `components` parameters
- Copies `pnpm-workspace.yaml` during build
- Uses `--filter` flags to build specific workspace members
- Handles multiple `node_modules` directories per component

## Development Conventions

### Nix Style Guidelines

1. **Formatting**: Use `nixpkgs-fmt` (checked in CI)
   ```bash
   nix run nixpkgs#nixpkgs-fmt -- --check .
   ```

2. **Function Parameters**:
   - Use `@attrs` pattern to capture all arguments
   - Provide defaults for all optional parameters
   - Document parameters in README.md table

3. **Let Bindings Organization**:
   ```nix
   let
     # Flags/computed indicators
     isWorkspace = workspace != null && components != [];

     # Utility functions
     forEachConcat = f: xs: ...;

     # Single-value computations
     nativeBuildInputs = [...];

     # Loop-based computations
     computedDistFiles = ...;
   in
   ```

4. **String Interpolation**:
   - Use `optionalString` for conditional strings
   - Use `concatStringsSep` for joining lists
   - Prefer `''` multi-line strings for bash scripts

5. **Attribute Updates**:
   - Use `recursiveUpdate` to merge derivation attributes
   - Clean up internal-only attributes in final result

### Git Workflow

- **Main branch**: `main` (not `master`)
- **CI**: Runs on push to main and all PRs
- **Checks**: Uses external `nzbr/actions` workflow
- **Branch naming**: Feature branches use `claude/` prefix (for AI assistants)

### Testing Changes

1. **Local flake checks**:
   ```bash
   nix flake check
   ```
   Runs:
   - `nixpkgs-fmt` formatting check
   - `build-example` builds the example package

2. **Build example manually**:
   ```bash
   nix build .#example
   ```

3. **Test with custom package**:
   ```bash
   nix build --impure --expr '
     with import <nixpkgs> {};
     callPackage ./derivation.nix {} {
       src = /path/to/your/package;
     }
   '
   ```

## Common Development Tasks

### Adding New Parameters

1. Update `derivation.nix` function signature
2. Document in `README.md` table
3. Use in appropriate build phase
4. Test with example or custom package
5. Update tests if needed

### Modifying Lockfile Processing

Files to update:
- `lockfile.nix`: Core parsing/processing logic
- `derivation.nix`: Usage of `processLockfile` results
- Test with various lockfile formats (pnpm 9, 10)

### Supporting New Dependency Sources

1. Add new case to `findTarball` switch in `lockfile.nix`
2. Pattern match on `resolution` fields
3. Return appropriate `fetchurl` or `fetchGit` call
4. Test with real-world package using that source

### Debugging Build Failures

1. **Check configurePhase**: pnpm install issues
   - Verify lockfile patching worked
   - Check store path is correct
   - Ensure all tarballs were fetched

2. **Check buildPhase**: Build script failures
   - Verify node_modules is populated
   - Check environment variables (`buildEnv`)
   - Run build script manually to diagnose

3. **Check installPhase**: Output copying issues
   - Verify `distDir` exists
   - Check `distDirs` list is correct
   - Ensure workspace paths are relative

## Key Implementation Details

### Parameter Defaults

Critical defaults to understand:
- `packageJSON = ${src}/package.json` - Strongly recommend overriding
- `pnpmLockYaml = ${src}/pnpm-lock.yaml` - Strongly recommend overriding
- `nodejs = pkgs.nodejs` - Recommend pinning specific version
- `pnpm = nodejs.pkgs.pnpm` - Recommend pinning specific version
- `registry = "https://registry.npmjs.org"` - Sometimes ignored by pnpm
- `script = "build"` - The npm script to run
- `distDir = "dist"` - Where build outputs go

### Workspace-Specific Behavior

When `workspace != null && components != []`:
- `src` defaults to `workspace`
- `distDirs` becomes `["component1/dist", "component2/dist", ...]`
- Build uses `pnpm run --filter ./component1 --filter ./component2 ...`
- Multiple `node_modules` directories are handled

### Environment Variables

Two separate environments:
1. **installEnv**: Set during `pnpm install` (configurePhase)
2. **buildEnv**: Set during build script execution (buildPhase)

Use these for:
- Setting `NODE_ENV`
- Configuring build tools
- Passing secrets/tokens (though avoid in public builds)

### Passthru Attributes

`passthru` is exposed for debugging:
- `passthru.attrs`: Original function arguments
- `passthru.patchedLockfileYaml`: The patched lockfile
- `passthru.processResultAllDeps`: List of dependency tarballs

Access via: `(mkPnpmPackage {...}).passthru.attrs`

## Advanced Usage Patterns

### Monorepo/Workspace Build

```nix
mkPnpmPackage {
  workspace = ./.;
  components = [ "packages/app" "packages/lib" ];
  pnpmWorkspaceYaml = ./pnpm-workspace.yaml;
  # ... other options
}
```

### Custom Build Script

```nix
mkPnpmPackage {
  src = ./.;
  scriptFull = ''
    pnpm run lint
    pnpm run test
    pnpm run build
  '';
}
```

### Installing node_modules in Output

```nix
mkPnpmPackage {
  src = ./.;
  installNodeModules = true;  # Includes node_modules in $out
}
```

### Production-Only Dependencies

```nix
mkPnpmPackage {
  src = ./.;
  noDevDependencies = true;  # Skip devDependencies
}
```

## Troubleshooting Guide

### Problem: "No matching case found" error

**Cause**: Unsupported package resolution format in lockfile

**Solution**:
1. Examine the error for `n=` (package name) and `v=` (version object)
2. Add new case to `findTarball` in `lockfile.nix`
3. Handle the specific `resolution` format

### Problem: Build works locally but fails in Nix

**Cause**: Missing system dependencies or impure operations

**Solution**:
- Add native dependencies to `extraNativeBuildInputs`
- Add runtime dependencies to `extraBuildInputs`
- Check for hardcoded paths or network access
- See `example/default.nix` which adds `vips` for `sharp` package

### Problem: IFD is too slow

**Cause**: Lockfile parsing happens at eval time

**Solution**: Always explicitly pass lockfile paths:
```nix
mkPnpmPackage {
  src = ./.;
  packageJSON = ./package.json;
  pnpmLockYaml = ./pnpm-lock.yaml;
}
```

### Problem: Workspace components not building

**Cause**: Incorrect paths or missing configuration

**Solution**:
- Ensure `components` are relative paths from `workspace`
- Provide explicit `pnpmWorkspaceYaml`
- Check that `pnpm-workspace.yaml` matches component paths

## Integration Examples

### As Flake Input

```nix
{
  inputs.pnpm2nix.url = "github:hawkrives/pnpm2nix-honk";

  outputs = { nixpkgs, pnpm2nix, ... }: {
    packages.x86_64-linux.myapp =
      pnpm2nix.packages.x86_64-linux.mkPnpmPackage {
        src = ./.;
      };
  };
}
```

### Via callPackage

```nix
let
  pkgs = import <nixpkgs> {};
  pnpm2nix = pkgs.callPackage /path/to/pnpm2nix/derivation.nix {};
in
  pnpm2nix.mkPnpmPackage {
    src = ./.;
  }
```

## Related Resources

- **Upstream**: Original project by nzbr
- **pnpm Documentation**: https://pnpm.io/
- **Nix Pills**: https://nixos.org/guides/nix-pills/
- **nixpkgs Manual**: https://nixos.org/manual/nixpkgs/

## AI Assistant Guidelines

When working on this codebase:

1. **Always read before editing**: Use Read tool on files before making changes
2. **Test changes**: Run `nix flake check` after modifications
3. **Format code**: Ensure nixpkgs-fmt passes
4. **Update documentation**: Keep README.md in sync with code changes
5. **Consider IFD performance**: Avoid adding more IFD if possible
6. **Handle errors gracefully**: Use proper Nix error messages
7. **Maintain compatibility**: This is a library, breaking changes affect users
8. **Test with example**: Always build the example package after changes

### Commit Message Style

Based on recent commits:
- Imperative mood: "Update", "Fix", "Add" (not "Updates", "Fixed", "Added")
- Be specific: "Update nixpkgs input" not "Update dependencies"
- Reference what changed: "Simplify passthru code" not "Code cleanup"

### Branch Strategy

- Work on feature branches prefixed with `claude/` for AI assistants
- Push to `claude/add-claude-documentation-XHWzK` for this task
- Create descriptive branch names for other features
