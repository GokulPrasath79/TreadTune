TreadTune — Automated Tyre Tread Pattern Tuning
Apollo Tyres Ltd × Vredestein Tyres — Pre-Development Project Status: Prototype / Proof of Concept · Target: FY27

A tool that automatically adjusts a tyre tread pattern to meet a target void (land:sea) ratio, then converts the tuned pattern into a multi-pitch tyre-noise sequence — removing a manual, iterative CAD task from the tread designer's workflow.

Table of Contents
Problem Statement
Objectives
Key Concepts
System Architecture
Process Flow
Algorithms
Prototype (This Repository)
Production Technology Stack
Feasibility Analysis
Risks & Mitigations
Roadmap
Deliverables Checklist
Limitations of the Current Prototype
Repository Structure
Getting Started
Glossary
1. Problem Statement
Tread pattern design at Apollo/Vredestein currently requires a designer to manually adjust groove geometry in CAD until the pattern's void ratio (the proportion of "sea" — grooves — to "land" — rubber contact area) meets a target specification, and then manually rebuild the pattern across a multi-pitch sequence (multiple pitch lengths scaled around a reference pitch, arranged to reduce tyre noise harmonics). Both steps are repetitive, time-consuming, and sensitive to human error, and they sit on the critical path of every new tread pattern program.

There is currently no tool that:

Automatically measures land:sea ratio from a tread pattern drawing.
Suggests or applies groove-geometry adjustments to hit a target ratio.
Automatically propagates an approved pattern across a pitch sequence with correct scaling.
2. Objectives
Main objective (from project brief):

Develop a tool that will help the tyre designer to automatically adjust a tread pattern based on the void ratio requirement and convert it into a multi-pitch output according to the required pitch ratio.

Supporting objectives:

Reduce manual CAD iteration time for void-ratio tuning.
Provide a repeatable, auditable process (inputs, thresholds, and outputs are logged).
Produce outputs designers already work with: CAD file + bitmap image.
Lay groundwork for a future computer-vision-based feature identification module (image processing / edge detection).
3. Key Concepts
Term	Definition
Tread Pattern	The full repeating groove/rib design molded into a tyre's tread.
Tread Pitch	One repeating unit of the tread pattern along the circumference.
Land	Rubber contact area of the tread (bears load, grips road).
Sea	Groove / void area of the tread (channels water, reduces noise resonance at specific frequencies).
Void Ratio	Sea area ÷ total tread area, expressed as a percentage. Typical passenger tyre range: 25–35%.
Multi-Pitch	A sequence of pitches at different scaled lengths (e.g., P1 = 0.90×P3, P3 = reference, P5 = 1.20×P3) arranged around the tyre circumference to spread noise energy across frequencies instead of concentrating it at one harmonic — reduces perceived tyre noise ("pitch sequencing" / "harmonic tuning").
Reference Pitch (P3)	The baseline/unscaled pitch that all other pitches in the sequence are multiples of.
4. System Architecture
4.1 High-Level Component Diagram
Output
Core Engine
Input
not within tolerance
within tolerance
CAD Drawing.dwg / .dxf
CAD Import & Parsing
Pattern FeatureIdentification
Land:SeaAnalysis Engine
Tuning / OptimizationModule
MultipitchScaling Engine
CAD Export.dwg / .dxf
Bitmap Export.png
Report / Log.json
4.2 Component Responsibilities
Component	Responsibility	Prototype Implementation	Production Implementation
CAD Import & Parsing	Read tread pattern geometry from CAD file, normalize into internal geometry model	N/A — accepts bitmap image directly	AutoCAD API / ObjectARX or ezdxf reading closed polylines/hatches tagged as land or sea layers
Pattern Feature Identification	Classify geometry (or pixels) into land vs. sea	Luminance thresholding on canvas pixel data	Layer/entity-based classification from CAD metadata, with vision-based fallback (see §11 Roadmap) for scanned/legacy drawings
Land:Sea Analysis Engine	Compute current void ratio; compare to target ± tolerance	Pixel counting over a binary mask	Polygon area summation (shoelace formula) over land/sea entity sets
Tuning / Optimization Module	Adjust groove geometry to close the gap to target ratio	Morphological dilation/erosion of the binary mask (simulates uniform groove widening/narrowing)	Parametric offset of groove boundary curves (CAD offset/fillet operations) driven by a solver (bisection or gradient-free search on offset distance vs. resulting ratio)
Multipitch Scaling Engine	Generate a full pitch sequence from one tuned reference pitch	Horizontal raster scaling of the tuned tile per multiplier	Vector scaling of pitch boundary curves along the circumferential axis, preserving groove depth/width proportions, with pitch-noise-sequence validation
Export	Produce CAD + bitmap + report outputs	PNG canvas export + JSON metadata	DWG/DXF write-back via CAD API, rasterized PNG, structured report (PDF/JSON)
4.3 Data Model (conceptual)

TreadPattern
 ├─ referencePitch: Geometry (land polygons[], sea polygons[])
 ├─ voidRatioTarget: float
 ├─ tolerance: float
 └─ pitchSequence: [
      { id, multiplier, geometry, isReference }
    ]
5. Process Flow
This mirrors the flow chart in the original project brief:

No
Yes
Not OK
OK
Tread Pattern DrawingI/P: CAD file
Pattern FeatureIdentification
AnalysisLand : Sea
Need Land:Seatuning?
Proceed toMultipitch Scaling
Tune Land:Sea
Land:Sea %OK?
Finish & ExportO/P: CAD file, Bitmap
6. Algorithms
6.1 Feature Identification (thresholding)
For each pixel/entity, classify as sea if luminance/attribute falls below a designer-set threshold, else land:


luminance = 0.299·R + 0.587·G + 0.114·B
class = SEA if luminance < threshold else LAND
In production, this is replaced by direct CAD layer/entity classification, since pattern drawings already separate land and sea geometry on distinct layers — thresholding is a fallback for raster/scanned input only.

6.2 Void Ratio Calculation

void_ratio (%) = (sea_area / total_area) × 100
Prototype: pixel counting over the analysis raster. Production: shoelace-formula polygon area summed over all sea entities, divided by total pitch bounding area.

6.3 Tuning (groove width adjustment)
Prototype uses morphological dilation/erosion: growing the sea mask by n pixels widens grooves (raises void ratio); eroding narrows them (lowers void ratio). An auto-tune routine sweeps candidate adjustment values and selects the one minimizing |actual − target|.

Production equivalent: offset every sea-boundary curve outward (dilate) or inward (erode) by a computed distance d, recompute area, and iterate — a 1-D root-finding problem (bisection or golden-section search on d vs. resulting void ratio), since void ratio is monotonic in offset distance for a fixed pattern topology.

6.4 Multi-Pitch Scaling
Given a reference pitch P₃ and a set of multipliers (e.g., 0.90 / 0.95 / 1.00 / 1.10 / 1.20), each pitch Pᵢ is generated by scaling the reference geometry along the circumferential axis by its multiplier, preserving lateral (radial) geometry unchanged:


Pᵢ.width  = P₃.width  × multiplierᵢ
Pᵢ.height = P₃.height              (unchanged)
Pitch order within the sequence affects noise performance; production tooling should support both designer-defined ordering and randomized/optimized ordering evaluated against a pitch-noise-sequence (PNS) model — outside the scope of this pre-development phase but flagged for the acoustics team.

7. Prototype (This Repository)
The included prototype (treadtune_prototype.html) is a single-file, browser-based demonstrator covering the full pipeline end-to-end on raster (bitmap) input, so the workflow and UX can be validated before committing to CAD-API integration effort.

What it does:

Accepts an uploaded tread pattern image, or generates a synthetic sample pattern.
Runs live threshold-based land/sea segmentation.
Displays void ratio on a live gauge against a designer-set target and tolerance.
Provides a groove-width tuning slider and an auto-tune search.
Generates a 3/5/7-pitch multi-pitch strip once the ratio is within tolerance, with a shuffle control for pitch ordering.
Exports a PNG bitmap and a JSON run report.
What it does not do (by design — see §13):

Read or write native .dwg / .dxf files.
Operate on true vector groove geometry (it operates on a rasterized approximation).
Model acoustic/pitch-noise-sequence performance.
8. Production Technology Stack
Layer	Recommended Options	Notes
CAD I/O	AutoCAD .NET API / ObjectARX (in-app plugin) or ezdxf / ODA File Converter (standalone service)	In-app plugin gives designers a native Ribbon/Palette experience inside AutoCAD; standalone service is easier to deploy/scale and version-control but requires a round trip out of AutoCAD
Geometry & analysis	Python (shapely, numpy) or C#/.NET geometry libraries	Shapely handles polygon boolean ops, offsets, and area calc cleanly
Optimization	scipy.optimize (bisection/brentq) for 1-D offset search	Void-ratio-vs-offset is well-behaved and monotonic — no heavy ML needed
UI	AutoCAD Ribbon/Palette (if in-app) or a lightweight desktop/web app (Electron, or internal web portal)	Match how designers already work; avoid introducing a second CAD-like tool if avoidable
Image processing (future, exploratory)	OpenCV (Canny edge detection, contour extraction)	For scanned/legacy drawings lacking CAD layer structure — see Roadmap
Storage/versioning	Existing PLM/CAD vault	Reuse Apollo's existing document management rather than introducing a new one
9. Feasibility Analysis
9.1 Technical Feasibility — High
Void ratio calculation from polygon geometry is a solved, well-understood computation (polygon area summation); no research risk.
Groove-width tuning as a 1-D monotonic search problem is tractable with standard numerical methods; no ML/AI required for the core loop.
Multi-pitch scaling is a geometric transform (uniform scale along one axis) — low complexity.
The main technical unknown is the depth of CAD API integration required (layer conventions, block structures, and how consistently existing tread drawings separate land/sea geometry across the design library). This should be scoped with a small sample of real production drawings before committing to a build estimate.
9.2 Data Feasibility — Medium-High
Requires tread pattern CAD files to have a consistent, machine-readable convention for distinguishing land vs. sea geometry (e.g., dedicated layers or closed-polyline hatches). If legacy drawings are inconsistent, a one-time cleanup pass or the image-processing fallback (§11) will be needed.
Void ratio targets and tolerances are already defined by design/compound engineering — no new data collection needed to define "correct" outputs.
9.3 Organizational / Workflow Feasibility — High
The tool augments, rather than replaces, the tread designer's existing CAD workflow — output remains a CAD file the designer reviews and finalizes, minimizing change-management risk.
Fits naturally as a plugin/step inside the existing pattern-development process; no new approval workflow required.
9.4 Resource & Timeline Feasibility
Rough phase-level estimate (subject to real scoping once CAD drawing conventions are confirmed):

Phase	Scope	Indicative Effort
Pilot (this prototype)	Validate UX and algorithm logic on raster input	Complete
Phase 1	CAD read/parse + land:sea analysis on real drawings	4–6 weeks
Phase 2	Tuning module (offset solver) + designer review UI	4–6 weeks
Phase 3	Multi-pitch scaling + CAD export	3–4 weeks
Phase 4	Pilot with 1–2 designers, refine, document	3–4 weeks
(Effort assumes one CAD/geometry engineer plus part-time design-team liaison; excludes procurement/licensing lead time for CAD SDK access.)

9.5 Cost Considerations
No new infrastructure required if delivered as an in-app AutoCAD plugin (uses designers' existing licenses).
Primary cost is engineering time; no specialized hardware, and no ML training cost since the core loop is deterministic geometry/optimization, not learned.
10. Risks & Mitigations
Risk	Impact	Mitigation
Inconsistent layer/entity conventions across legacy tread drawings	Feature identification fails or requires per-drawing cleanup	Audit a representative drawing sample early (Phase 1); define a minimum drawing standard going forward; keep raster fallback for outliers
Uniform groove-width offset doesn't preserve designer intent for complex/asymmetric patterns	Tuned pattern looks mechanically correct but aesthetically/functionally wrong	Keep the tuning step advisory — designer reviews and can hand-adjust before export; support region-limited (not whole-pattern) tuning in Phase 2
Multi-pitch scaling ignores acoustic (pitch-noise-sequence) performance	Generated sequence may not be optimal for noise	Flag explicitly as geometric-only in this phase; involve NVH/acoustics team before treating output as final
CAD API access/licensing delays	Timeline slip	Start Phase 1 spike early to confirm SDK access; standalone ezdxf-based service is a lower-risk fallback if in-app API access is delayed
Scope creep toward full computer-vision pattern recognition	Timeline/complexity blowout	Treat image processing/edge detection explicitly as an exploratory future feature (§11), not a Phase 1–4 dependency
11. Roadmap
Now — Pre-development: This prototype; validate workflow/UX with design team.
Phase 1–4: As per §9.4 — CAD-native pipeline, tuning, multi-pitch, export, pilot.
Future / Exploratory — Image Processing & Edge Detection: Investigate CV-based feature identification (e.g., Canny edge detection + contour extraction) for scanned or non-standard drawings where CAD layer conventions are unavailable, so the tool can ingest a wider range of legacy source material. Called out explicitly as exploratory in the original project brief and not required for the core deliverable.
Future: Pitch-noise-sequence-aware ordering (in collaboration with NVH/acoustics), batch processing across a full pattern family, direct integration with the design review/approval system.
12. Deliverables Checklist
Per the original project brief:

 Tool with user-friendly interface (input & output) — prototype delivered
 Native AutoCAD drawing file input/output — pending CAD-API integration (Phase 1–3)
 Bitmap file output — implemented in prototype
 System architecture diagram — this document, §4
 Algorithm documentation — this document, §6
 User manual — to be written alongside Phase 4 pilot
 Exploration of image processing / edge detection — scoped as future work, §11
13. Limitations of the Current Prototype
Operates on a rasterized approximation of the pattern, not true vector CAD geometry — groove edges are not mathematically exact and cannot be exported back into a CAD-native, editable form.
Threshold-based feature identification is a stand-in for CAD layer/entity classification; it is sensitive to image quality and lighting/contrast when a real photo is used.
Uniform groove-width tuning (dilate/erode) does not account for local pattern intent — a production tuning module needs region-aware or curve-specific offsetting.
Multi-pitch generation is geometric only — it does not evaluate or optimize acoustic (pitch noise sequence) performance.
No persistence layer — the prototype does not save/version pattern iterations; production tooling should integrate with Apollo's existing PLM/CAD vault.
14. Repository Structure

.
├── README.md                     # this file
├── treadtune_prototype.html      # single-file interactive prototype (open in any browser)
└── docs/
    └── (future: user manual, drawing-convention audit notes, API integration spec)
15. Getting Started
Clone this repository.
Open treadtune_prototype.html directly in a modern browser (Chrome/Edge/Firefox) — no build step or server required.
Upload a tread pattern image, or click "Use synthetic sample".
Work through Steps 01–06 in the left panel: detect features → analyze ratio → tune → scale to multi-pitch → export.
No installation, dependencies, or backend are required for the prototype; it runs entirely client-side.

16. Glossary
Term	Meaning
DWG / DXF	Native/interchange CAD file formats used by AutoCAD
PNS	Pitch Noise Sequence — the ordering of scaled pitches around a tyre to manage noise harmonics
NVH	Noise, Vibration, Harshness — the engineering discipline concerned with tyre acoustic performance
PLM	Product Lifecycle Management — Apollo's existing design document/version management system
Prepared as a pre-development artifact for internal review. Reflects the scope discussed in "Automated Tyre Tread Pattern Tuning" (10.04.2026).
