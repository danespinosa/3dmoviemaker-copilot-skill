# 3D Movie Maker Copilot Skill

A [GitHub Copilot CLI skill](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills) that guides you through building and running Microsoft 3D Movie Maker from source on modern Windows.

## What it does

This skill teaches Copilot how to:
- Clone and build the [3DMMForever](https://github.com/foone/3DMMForever) community fork using CMake + MSVC
- Fix SDK header conflicts on modern Windows
- Set up the runtime directory layout and registry entries
- Troubleshoot common launch errors

## Installation

**Personal skill** (all projects):
```
cp -r . ~/.copilot/skills/build-3dmm/
```

**Repository skill** (single project):
```
cp -r . .github/skills/build-3dmm/
```

## Usage

Just ask Copilot to build 3D Movie Maker:
```
Build and run 3D Movie Maker from source
```

Or invoke explicitly:
```
Use the /build-3dmm skill to compile 3D Movie Maker
```

## License

MIT
