# The German (PAL) disc: experimental

SpeedBreaker runs the USA disc and the European (PAL) Spanish disc. The code here
lets a build run the German disc too (*Need for Speed - Most Wanted (Germany)*,
`default.xex` SHA-256 `23ece29d18724ea28971d52f936964d127148bd2359ff24795da7442b9069bf4`),
the same way the Spanish one runs: its own recompiled code beside the USA's, with
the runtime's hooks moved to its addresses. Nothing for it is committed yet: the
config, the address table and the install manifest are generated from your own
discs, by the steps below. Whether they work for this disc is only known once
they have run on it.

## What you need

- **The USA disc's `default.xex` as well** (SHA-256 `aebdf3c1...`). Every
  SpeedBreaker build needs the USA code (`ppc/`), and the German addresses are
  found by comparing the German executable with the USA one.
- The German disc image.
- The toolchain from README step 2 (`scripts/setup_tools.sh`).

Everything derived from the discs goes in `game/` (git-ignored). The files the
steps write in the repository (`config/pal-de/`, `runtime/region/pal_de_*`,
`runtime/install/manifest_pal_de.inc`) hold addresses, names, sizes and hashes
only, as their Spanish counterparts do.

## Steps

From the repository root:

```sh
# 1. Both discs, and the USA code (README step 2).
mkdir -p game/files game/files-pal-de ppc ppc-pal-de
tools/extract-xiso/build/extract-xiso -d game/files -x /path/to/your-usa-backup.iso
tools/extract-xiso/build/extract-xiso -d game/files-pal-de -x "/path/to/Need for Speed - Most Wanted (Germany).iso"
shasum -a 256 game/files/default.xex game/files-pal-de/default.xex
tools/XenonRecomp/build/XenonRecomp/XenonRecomp config/nfsmw.toml tools/XenonRecomp/XenonUtils/ppc_context.h

# 2. Each executable decrypted, with its section table (survey/xexsurvey.cpp).
mkdir -p build game/survey/mw_usa game/survey/mw_de
clang++ -std=c++20 -O2 -include cstdlib -Itools/XenonRecomp/XenonUtils -Itools/XenonRecomp/thirdparty/simde \
  survey/xexsurvey.cpp tools/XenonRecomp/build/XenonUtils/libXenonUtils.a \
  tools/XenonRecomp/build/thirdparty/disasm/libdisasm.a -o build/xexsurvey
build/xexsurvey game/files/default.xex game/survey/mw_usa > game/survey/mw_usa/xexsurvey.txt
build/xexsurvey game/files-pal-de/default.xex game/survey/mw_de > game/survey/mw_de/xexsurvey.txt

# 3. The function map: each USA function's counterpart in the German build.
python3 -I survey/func_map.py game/survey/mw_usa game/survey/mw_de > game/survey/mw_de/funcmap.tsv

# 4. The recompiler config and the runtime's address table, then their check.
S=game/survey
python3 -I survey/region_map.py toml  $S/mw_usa $S/mw_de $S/mw_de/funcmap.tsv --region pal-de
python3 -I survey/region_map.py table $S/mw_usa $S/mw_de $S/mw_de/funcmap.tsv --region pal-de
python3 -I tests/region_addresses_check.py . $S/mw_usa $S/mw_de $S/mw_de/funcmap.tsv --region pal-de

# 5. The German code, namespaced so it links beside the USA's.
tools/XenonRecomp/build/XenonRecomp/XenonRecomp config/pal-de/nfsmw.toml tools/XenonRecomp/XenonUtils/ppc_context.h
python3 scripts/region_ppc.py ppc-pal-de pal_de

# 6. The installer's file list for the German disc.
python3 scripts/gen_install_manifest.py game/files-pal-de --config config/pal-de/nfsmw.toml \
  -o runtime/install/manifest_pal_de.inc
```

Then build as in README step 3: CMake says `PAL (German) release: linked
(ppc-pal-de/)`. Play from the extracted folder with
`NFSMW_GAME_DIR=game/files-pal-de build/main/runtime/SpeedBreaker` (run from the
repository root without it, the USA files in `./game/files` are found first), or
install the image with the installer: the log says `[region] Europe (PAL,
German): its own recompiled code`, and the German disc's saves go to
`save-pal-de/`.

## What to look at on the way

- **Step 3** prints how many functions are the same, changed or missing, and in
  how many pieces the layout moved. The Spanish build: 22,679 functions, all but
  19 the same code, 17 pieces. Far more changed or missing functions mean a
  build further from the USA one than the Spanish one is, and more work below.
- **Step 4** stops with every address it can't map, each with the reason: code
  the German build changed or doesn't have, an `.rdata` item it can't find, or
  `.data moved` (the approach assumes `.data` and `.bss` keep the USA addresses,
  as they do in the Spanish build). Each of those needs a look by hand.
  `region_addresses_check` also fails for any runtime address inside a function
  the German build changed: review those hooks.
- **Step 4** also lists the jump tables only the German build has (the Spanish
  build has two, in leaf functions for its date and number formats), with the
  function bounds it found for them; `NOT a bl target` marks a start to check
  by hand.
- **Step 5:** XenonRecomp's `Found a switch jump table at ... with no switch
  table entry present` or `Switch case at ... is trying to jump outside function`
  means a table or a function's bounds the config still lacks.
- **At startup**, `the Europe (PAL, German) code's layout ... isn't the USA's`
  means its image base, image size or code base differ from the USA build's,
  which the runtime doesn't support.
- **Languages:** the German disc is assumed to be laid out like the Spanish one
  (its executable boots in German, asks for `_pal` movies, and the disc holds
  German movies and speech); German is the Language row's default for it. The
  log's `[language]` line at boot says what the installed files give.

## If two discs' `default.xex` have one size

Installs are told apart by `default.xex`'s size without hashing it. If the
German one has the USA or the Spanish one's size, the install marker's manifest
version decides, and without a marker (a folder set up by hand) the file's
SHA-256 (`install::ManifestForXexFile`).
