# The bake gate — `bake_validate.py`

The **bake gate** is the objective purple/invisible check from the validation stack: it
resolves `blockstate → model → parent chain → textures` the way Minecraft's model baker
does, across a layered resource view (loose folders and/or mod jars), and flags the assets
that will bake **purple/invisible** in‑game.

This skill ships the validator **as source in this file**, not as a pre‑placed executable.
Recreate it once on your install, then run it like any script.

## Recreate it on your install (one time)

Write the code block below to `scripts/bake_validate.py` inside the skill folder, then run it.

- Personal install → `~/.claude/skills/minecraft-modding/scripts/bake_validate.py`
- Project install → `<project>/.claude/skills/minecraft-modding/scripts/bake_validate.py`

Claude can do this for you: *"recreate the bake gate from references/bake-validate.md and run it against `<path>`."* It writes the file, then invokes it. Nothing else to install — it's stdlib‑only Python 3.

## Run it

```bash
python scripts/bake_validate.py <pack-or-jar-or-mods-folder> --mc-version 26.3
# add --json for machine-readable output
python scripts/bake_validate.py --selftest        # runs its own assertions, no args
```

A `<path>` is a resource‑pack folder, an unzipped mod folder, a `.jar`, or a folder of jars.
Exit code `0` = no hard failures, `1` = hard failures found, `2` = bad invocation.

## Standing rule — TESTED vs UNVERIFIED

The validator carries a *verified* era ruleset only for the versions in its
`VERIFIED_VERSIONS` (currently 26.1–26.3, mirroring
[version-baselines.md](version-baselines.md)). On any other `--mc-version` it runs its
**universal** checks authoritatively but prints era‑specific findings as **UNVERIFIED
advisories** — trust the universal results, but confirm each era‑gated finding against the
target jar (the `javap` method) before acting. **If you verify a new version, add it to
`VERIFIED_VERSIONS` so the trusted ground grows by evidence, never by guessing.** Never let a
tool emit a confident verdict for a version it hasn't been proven on.

## Source

```python
#!/usr/bin/env python3
"""
bake_validate.py — the "bake gate" from the minecraft-modding skill, as a runnable tool.

Resolves blockstate -> model -> parent chain -> textures the way Minecraft's model
baker does, across a layered resource view (loose folders and/or mod jars), and flags
the assets that will bake PURPLE/INVISIBLE in-game.

Two classes of check:
  * UNIVERSAL (mechanism-based, valid on every version that has the model system):
      - a blockstate/item model references a model file that doesn't exist
      - a parent chain leads to a missing (non-vanilla) model
      - a referenced non-particle texture doesn't exist
      - a model uses a custom "loader" (Forge/NeoForge) with no vanilla geometry
  * ERA-GATED (only correct at known version breakpoints):
      - items/ definition required          (>= 1.21.4)
      - parent "builtin/entity" is illegal   (>= 26.2)

Honesty rule (matches the skill): era-gated rules are applied as hard failures ONLY on
versions whose ruleset we have VERIFIED. On any other version the universal checks still
run authoritatively, but era-gated findings are reported as UNVERIFIED advisories, and the
tool tells you to confirm against the target jar. Verified versions live in ONE place
below (VERIFIED_VERSIONS) so adding a newly-tested version is a one-line edit, never a
research project.

stdlib only. Usage:
    python bake_validate.py <path> [<path> ...] --mc-version 26.3 [--json]
    python bake_validate.py --selftest        # runs built-in assertions, no args needed

A <path> is a resource-pack folder, an unzipped mod folder, a .jar, or a folder of jars.
Exit code 0 = no hard failures, 1 = hard failures found, 2 = bad invocation.
"""
from __future__ import annotations
import argparse, json, os, re, sys, tempfile, zipfile
from pathlib import Path

# --- version model ------------------------------------------------------------
# Breakpoints where an era-gated rule flips on. Compared as padded integer tuples;
# this works across the 1.x and 26.x schemes because 26 > 1 in the major slot.
ITEMS_DEF_REQUIRED_FROM   = (1, 21, 4)   # the separate assets/<ns>/items/<name>.json
BUILTIN_ENTITY_ILLEGAL_FROM = (26, 2, 0) # parent "builtin/entity" removed

# Versions whose era ruleset we have actually tested. Add a line when you verify one.
VERIFIED_VERSIONS = {
    (26, 1, 0), (26, 1, 1), (26, 1, 2), (26, 2, 0), (26, 3, 0),
}

def parse_version(v: str) -> tuple[int, int, int]:
    """'26.3' -> (26,3,0); '1.21.4' -> (1,21,4); '26.4-snapshot-7' -> (26,4,0)."""
    m = re.match(r"\s*(\d+(?:\.\d+)*)", v or "")
    if not m:
        raise ValueError(f"unparseable mc-version: {v!r}")
    parts = [int(x) for x in m.group(1).split(".")][:3]
    while len(parts) < 3:
        parts.append(0)
    return tuple(parts)  # type: ignore[return-value]

# --- resource view ------------------------------------------------------------
class View:
    """A merged, read-only resource view. Keys are POSIX paths from 'assets/...'."""
    def __init__(self) -> None:
        self.files: dict[str, bytes | None] = {}  # key -> bytes (None = lazy/dir entry)
        self._jsons: dict[str, dict] = {}

    def _add(self, key: str, data: bytes) -> None:
        self.files.setdefault(key, data)  # first source wins; good enough for QC

    def add_path(self, p: Path) -> None:
        if p.is_file() and p.suffix.lower() == ".jar":
            self._add_jar(p)
        elif p.is_dir():
            # loose files under this dir...
            for f in p.rglob("*"):
                if f.is_file():
                    key = _asset_key(f.as_posix())
                    if key:
                        self._add(key, f.read_bytes())
            # ...plus any jars sitting directly inside it (a mods/ folder)
            for jar in p.glob("*.jar"):
                self._add_jar(jar)
        else:
            raise FileNotFoundError(p)

    def _add_jar(self, jar: Path) -> None:
        with zipfile.ZipFile(jar) as z:
            for name in z.namelist():
                key = _asset_key(name)
                if key:
                    self._add(key, z.read(name))

    def has(self, key: str) -> bool:
        return key in self.files

    def json(self, key: str) -> dict | None:
        if key in self._jsons:
            return self._jsons[key]
        raw = self.files.get(key)
        if raw is None:
            return None
        try:
            obj = json.loads(raw.decode("utf-8-sig"))
        except Exception:
            obj = {}
        self._jsons[key] = obj
        return obj

    def keys_under(self, prefix: str):
        return [k for k in self.files if k.startswith(prefix)]

def _asset_key(path: str) -> str | None:
    """Return the 'assets/...'-rooted key for a file path, or None if not an asset."""
    path = path.replace("\\", "/")
    i = path.find("assets/")
    return path[i:] if i != -1 else None

# --- id -> resource key -------------------------------------------------------
def rid(ident: str) -> tuple[str, str]:
    ns, _, name = ident.partition(":")
    return (ns, name) if _ else ("minecraft", ns)

def model_key(ident: str) -> str:
    ns, name = rid(ident)
    return f"assets/{ns}/models/{name}.json"

def texture_key(ident: str) -> str:
    ns, name = rid(ident)
    return f"assets/{ns}/textures/{name}.png"

# --- findings -----------------------------------------------------------------
class F:
    FAIL, WARN, INFO = "FAIL", "WARN", "INFO"
    def __init__(self, sev, where, msg, verified=True):
        self.sev, self.where, self.msg, self.verified = sev, where, msg, verified
    def as_dict(self):
        return {"severity": self.sev, "where": self.where, "message": self.msg,
                "verified": self.verified}

# --- the checks ---------------------------------------------------------------
def resolve_model(view: View, ident: str, findings: list, origin: str,
                  era: dict, _seen=None) -> bool:
    """Walk a model + its parent chain. Returns True if it bakes to real geometry
    (or terminates in a vanilla parent we trust). Appends findings for breaks."""
    _seen = _seen or set()
    ns, _ = rid(ident)
    key = model_key(ident)
    if ident in _seen:
        return True  # cycle guard; a separate pass could flag it
    _seen.add(ident)

    if not view.has(key):
        if ns == "minecraft":
            return True  # vanilla provides it; not our failure
        findings.append(F(F.FAIL, origin, f"model '{ident}' not found ({key})"))
        return False

    m = view.json(key) or {}

    # era: builtin/entity parent
    parent = m.get("parent")
    if parent and rid(parent)[1] == "builtin/entity":
        if era["builtin_entity_illegal"]:
            findings.append(F(F.FAIL, key, "parent 'builtin/entity' was removed (>=26.2)",
                              verified=era["verified"]))
        elif not era["verified"]:
            findings.append(F(F.WARN, key, "parent 'builtin/entity' — ok pre-26.2; "
                              "UNVERIFIED for this version, confirm against the jar",
                              verified=False))

    # custom loader with no vanilla geometry -> silent bake failure on Fabric
    if "loader" in m and "elements" not in m:
        findings.append(F(F.WARN, key, f"custom model loader '{m['loader']}' and no "
                          "'elements' — bakes INVISIBLE on Fabric unless a Fabric model "
                          "loader is registered"))

    # textures present in THIS model file must point at real pngs (skip #refs/particle)
    for slot, val in (m.get("textures") or {}).items():
        if not isinstance(val, str) or val.startswith("#"):
            continue
        if not view.has(texture_key(val)):
            tns, _ = rid(val)
            if tns == "minecraft":
                continue
            sev = F.WARN if slot == "particle" else F.FAIL
            findings.append(F(sev, key, f"texture '{val}' for slot '{slot}' missing "
                              f"({texture_key(val)})"))

    has_geometry = "elements" in m
    if parent:
        has_geometry = resolve_model(view, parent, findings, origin, era, _seen) or has_geometry
    return has_geometry

def check_blockstates(view: View, findings: list, era: dict) -> None:
    for key in view.keys_under("assets/"):
        if "/blockstates/" not in key or not key.endswith(".json"):
            continue
        bs = view.json(key) or {}
        models: list[str] = []
        for var in (bs.get("variants") or {}).values():
            for entry in (var if isinstance(var, list) else [var]):
                if isinstance(entry, dict) and "model" in entry:
                    models.append(entry["model"])
        for case in (bs.get("multipart") or []):
            app = case.get("apply", {})
            for entry in (app if isinstance(app, list) else [app]):
                if isinstance(entry, dict) and "model" in entry:
                    models.append(entry["model"])
        for mid in models:
            resolve_model(view, mid, findings, key, era)

def check_item_models(view: View, findings: list, era: dict) -> None:
    for key in view.keys_under("assets/"):
        m = re.match(r"assets/([^/]+)/models/item/(.+)\.json$", key)
        if not m:
            continue
        ns, name = m.group(1), m.group(2)
        resolve_model(view, f"{ns}:item/{name}", findings, key, era)
        items_def = f"assets/{ns}/items/{name}.json"
        if not view.has(items_def):
            if era["items_def_required"]:
                findings.append(F(F.FAIL, key, f"missing item definition {items_def} "
                                  "(>=1.21.4) — icon bakes PURPLE", verified=era["verified"]))
            elif not era["verified"]:
                findings.append(F(F.WARN, key, f"no {items_def} — not required pre-1.21.4; "
                                  "UNVERIFIED for this version, confirm against the jar",
                                  verified=False))

def validate(paths: list[Path], mc_version: str) -> tuple[list, dict]:
    ver = parse_version(mc_version)
    verified = ver in VERIFIED_VERSIONS
    era = {
        "verified": verified,
        "items_def_required": ver >= ITEMS_DEF_REQUIRED_FROM,
        "builtin_entity_illegal": ver >= BUILTIN_ENTITY_ILLEGAL_FROM,
    }
    view = View()
    for p in paths:
        view.add_path(p)
    findings: list = []
    check_blockstates(view, findings, era)
    check_item_models(view, findings, era)
    meta = {"mc_version": mc_version, "verified_ruleset": verified,
            "asset_files": len(view.files), "era": {k: era[k] for k in
            ("items_def_required", "builtin_entity_illegal")}}
    return findings, meta

# --- CLI ----------------------------------------------------------------------
def _print_report(findings: list, meta: dict) -> int:
    fails = [f for f in findings if f.sev == F.FAIL]
    warns = [f for f in findings if f.sev == F.WARN]
    mode = "TESTED" if meta["verified_ruleset"] else "UNVERIFIED"
    print(f"bake gate · mc {meta['mc_version']} · ruleset: {mode} · "
          f"{meta['asset_files']} asset files")
    if mode == "UNVERIFIED":
        print("  ! No verified ruleset for this version. Universal checks are reliable; "
              "era-gated findings are advisory — confirm against the target jar, and if you "
              "verify this version, add it to VERIFIED_VERSIONS.")
    for f in fails + warns:
        tag = "" if f.verified else " [unverified]"
        print(f"  {f.sev:4} {f.where}: {f.msg}{tag}")
    if not fails and not warns:
        print("  clean — no bake problems found")
    print(f"summary: {len(fails)} FAIL, {len(warns)} WARN")
    return 1 if fails else 0

def _selftest() -> int:
    """Build a tiny synthetic pack and assert the gate catches exactly the planted bugs."""
    with tempfile.TemporaryDirectory() as d:
        root = Path(d)
        def put(rel, obj):
            p = root / rel
            p.parent.mkdir(parents=True, exist_ok=True)
            p.write_text(json.dumps(obj) if isinstance(obj, dict) else obj, encoding="utf-8")
        ns = "testmod"
        # good block: blockstate -> model with elements + present texture
        put(f"assets/{ns}/blockstates/good.json", {"variants": {"": {"model": f"{ns}:block/good"}}})
        put(f"assets/{ns}/models/block/good.json", {"elements": [{}], "textures": {"all": f"{ns}:block/good"}})
        (root / f"assets/{ns}/textures/block").mkdir(parents=True, exist_ok=True)
        (root / f"assets/{ns}/textures/block/good.png").write_bytes(b"\x89PNG")
        # bug 1: blockstate points at a missing model
        put(f"assets/{ns}/blockstates/ghost.json", {"variants": {"": {"model": f"{ns}:block/missing"}}})
        # bug 2: model with custom loader and no elements
        put(f"assets/{ns}/blockstates/forgey.json", {"variants": {"": {"model": f"{ns}:block/forgey"}}})
        put(f"assets/{ns}/models/block/forgey.json", {"loader": "forge:obj", "textures": {}})
        # bug 3: item model missing its texture AND (on 1.21.4+) its items/ def
        put(f"assets/{ns}/models/item/widget.json", {"parent": "minecraft:item/generated",
                                                      "textures": {"layer0": f"{ns}:item/nope"}})

        f_modern, _ = validate([root], "26.3")
        msgs = " | ".join(x.msg for x in f_modern)
        fails = {x.where + "::" + x.msg for x in f_modern if x.sev == F.FAIL}
        assert any("block/missing" in m for m in fails), "missing model not caught"
        assert any("obj" in x.msg for x in f_modern if x.sev == F.WARN), "loader not warned"
        assert any("item/nope" in m for m in fails), "missing item texture not caught"
        assert any("items/widget.json" in m for m in fails), "missing items/ def not caught (26.3)"
        assert not any("good" in x.where and x.sev == F.FAIL for x in f_modern), "false positive on good block"

        # same pack on an OLD version: items/ def must NOT be a hard fail
        f_legacy, meta = validate([root], "1.20.1")
        assert meta["verified_ruleset"] is False, "1.20.1 should be UNVERIFIED"
        assert not any("items/widget.json" in x.msg and x.sev == F.FAIL for x in f_legacy), \
            "items/ def wrongly failed on 1.20.1"
        assert any("block/missing" in x.msg for x in f_legacy if x.sev == F.FAIL), \
            "universal check should still fire on 1.20.1"
    print("selftest: OK — universal checks fire on every version; era rules gate correctly; "
          "no false positive on a valid asset.")
    return 0

def main(argv=None) -> int:
    try:
        sys.stdout.reconfigure(encoding="utf-8")  # clean ·—→ glyphs on Windows consoles
    except Exception:
        pass
    ap = argparse.ArgumentParser(description="Bake gate: flag purple/invisible assets.")
    ap.add_argument("paths", nargs="*", help="resource folder(s), mod folder(s), or .jar(s)")
    ap.add_argument("--mc-version", default="26.3", help="target MC version, e.g. 26.3 or 1.20.1")
    ap.add_argument("--json", action="store_true", help="emit findings as JSON")
    ap.add_argument("--selftest", action="store_true", help="run built-in assertions and exit")
    args = ap.parse_args(argv)
    if args.selftest:
        return _selftest()
    if not args.paths:
        ap.error("give at least one path (or --selftest)")
    findings, meta = validate([Path(p) for p in args.paths], args.mc_version)
    if args.json:
        print(json.dumps({"meta": meta, "findings": [f.as_dict() for f in findings]}, indent=2))
        return 1 if any(f.sev == F.FAIL for f in findings) else 0
    return _print_report(findings, meta)

if __name__ == "__main__":
    sys.exit(main())
```
