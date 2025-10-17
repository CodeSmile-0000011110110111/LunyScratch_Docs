# Metrics

The split is within 1% of the metrics for Unity: 67% Core vs 33% Godot

It appears Godot version has slightly higher cyclomatic complexity, which may be attributed to me not going over in detail over AI generated code. But also Godot's deep hierarchical OOP inheritance chain and node graph contribute to that. It's certainly more "complex" by providing both a deep inheritance AND a deep node graph, as opposed to a deep node graph, where each node (GameObject) has a list of components (MonoBehaviour).


=== LunyScratch Metrics ===
Package Root         : U:\LunyScratch\LunyScratch_Examples_Godot\addons\lunyscratch
Detected Engine      : Godot

--- Lines of Code (LOC) ---
Core LOC            : 2708
Engine LOC          : 1334
Editor LOC          : 0
Engine+Editor LOC   : 1334
TOTAL LOC           : 4042
Split               : Core 67% | Engine+Editor 33%

--- Complexity (proxies) ---
Core Cyclomatic     : 237  | Cognitive: 215
Core Per-LOC        : 0.088  | Maintainability: 96

Eng+Ed Cyclomatic   : 169  | Cognitive: 162.5
Eng+Ed Per-LOC      : 0.127 | Maintainability: 94

--- Baselines & Ranges ---
Maintainability ranges: >=90 Good, 80-90 Watch, <80 Needs Attention
Cyclomatic density ranges (per LOC): <0.12 Good, 0.12-0.20 Watch, >0.20 High
Core Ratings        : Maintainability=Good | CyclomaticDensity=Good
Eng+Ed Ratings      : Maintainability=Good | CyclomaticDensity=Watch
