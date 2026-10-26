# OSM-Based Traffic Sign Localization Agent

## 1. Project goal

Build an agent system that turns raw traffic-sign detections from
street-view imagery into validated geospatial objects. The system
combines:

-   **OSM street network** --- candidate road links, topology, road
    classification, and direction context.
-   **Segmentation-model detections** --- the raw output from the model,
    including masks, bounding boxes, class labels, and confidence.
-   **360° street-view imagery and camera metadata** --- visual evidence
    and camera pose needed to estimate where a detected object is
    located and which direction its face points.
-   **Sidewalk, curb, road-edge, or other boundary geometry** ---
    spatial constraints that help distinguish a sign beside the road
    from one on a parallel road or frontage road.
-   **Reviewed results and QA outcomes** --- evidence for improving
    future processing and memory.

The output should contain a traceable estimate of each object's **class,
position, heading, road association, uncertainty, evidence, and QA
status**.

> **Core principle:** a model detection is an observation, not yet a
> geospatial object. Keep raw evidence immutable and store the estimated
> physical object and its map association separately.

## 2. Recommended architecture

Use a deterministic Python processing core coordinated by ordinary
Python modules and services. **The default architecture does not require
an LLM API and must not make paid LLM calls at runtime.**

Use Claude Code through your existing Claude Pro subscription as a
development assistant in VS Code. Claude Code helps you write, review,
and test the code; it is not a runtime dependency of the finished
application. The application should continue to run when Claude Code is
closed and when there is no internet connection, apart from any external
data services that you deliberately use.

Use Python functions, geospatial libraries, SQL, configured rules, and
scheduled jobs for runtime processing. Keep the system modular so that
an LLM integration could be considered separately in the future, but do
not include one in the default implementation.

``` mermaid
flowchart TD
    A[Raw segmentation detections] --> B[Observation normalization]
    I[360 street-view images + camera metadata] --> B
    O[OSM road network] --> C[Candidate road generation]
    S[Sidewalk / curb / road boundaries] --> C
    B --> D[Cross-view matching]
    C --> D
    D --> E[Localization and uncertainty engine]
    E --> F[Heading estimation]
    F --> G[Spatial and semantic QA]
    G -->|Pass| H[Validated traffic-sign records]
    G -->|Needs review| R[Human review queue]
    G -->|Fail| X[Rejected / unresolved records]
    R --> H
    H --> M[Session traces and QA outcomes]
    M --> N[Dreaming / memory update proposal]
    N --> V[Evaluation against reviewed cases]
    V -->|Approved| W[Versioned shared memory]
    V -->|Rejected| M
    W -. informs future runs .-> B
    W -. informs future runs .-> C
    W -. informs future runs .-> G
```

### Main processing stages

1.  **Ingestion** --- read raw detections, image/panorama identifiers,
    camera metadata, OSM features, and boundary geometries.
2.  **Observation normalization** --- standardize classes, confidence
    values, masks, pixel coordinates, panorama angles, and metadata.
3.  **Candidate generation** --- find plausible roads and boundary
    segments around each observation; retain multiple candidates when
    ambiguity exists.
4.  **Cross-view matching** --- decide whether detections from different
    images are observations of the same physical sign.
5.  **Localization** --- estimate a defined reference point on the
    object, such as pole base or sign-face center, and quantify
    uncertainty.
6.  **Heading estimation** --- estimate the sign-face normal in a
    documented reference convention.
7.  **QA and decision** --- accept, reject, or route the estimate for
    review with evidence and rule versions.
8.  **Persistence and provenance** --- store outputs, source
    observations, processing versions, and review decisions.
9.  **Dreaming** --- periodically analyze outcomes and propose
    improvements to shared memory; evaluate proposals before
    publication.

## 3. Separate the concepts in your data model

Keep these three concepts distinct:

-   **Observation:** what a particular image and model run observed.
-   **Physical object estimate:** the estimated location and orientation
    of the real sign.
-   **Map association:** the road link or road-side context related to
    that object.

One physical sign may have several observations. Several observations
should not automatically become several physical signs. A road link is a
contextual association, not proof of the sign's exact location.

## 4. Suggested repository structure

``` text
traffic-sign-agent/
├── README.md
├── pyproject.toml
├── .env.example
├── config/
│   ├── agent.yaml
│   ├── geometry.yaml
│   ├── qa_rules.yaml
│   └── dreaming.yaml
├── data_contracts/
│   ├── raw_detection.schema.json
│   ├── localized_sign.schema.json
│   └── qa_result.schema.json
├── src/
│   └── traffic_sign_agent/
│       ├── main.py
│       ├── ingestion/
│       │   ├── raw_detection_reader.py
│       │   ├── panorama_reader.py
│       │   └── camera_metadata.py
│       ├── perception/
│       │   ├── detection_adapter.py
│       │   ├── mask_geometry.py
│       │   └── cross_view_matching.py
│       ├── geospatial/
│       │   ├── crs_transform.py
│       │   ├── camera_projection.py
│       │   ├── road_candidates.py
│       │   ├── sidewalk_constraints.py
│       │   ├── triangulation.py
│       │   ├── heading_estimator.py
│       │   └── uncertainty.py
│       ├── agents/
│       │   ├── orchestrator.py
│       │   ├── observation_agent.py
│       │   ├── association_agent.py
│       │   ├── localization_agent.py
│       │   └── qa_agent.py
│       ├── validation/
│       │   ├── spatial_rules.py
│       │   ├── duplicate_detection.py
│       │   ├── confidence_policy.py
│       │   └── review_queue.py
│       ├── storage/
│       │   ├── postgis_repository.py
│       │   ├── object_store.py
│       │   └── provenance.py
│       ├── memory/
│       │   ├── store.py
│       │   ├── retrieval.py
│       │   ├── lesson_extractor.py
│       │   └── versioning.py
│       └── dreaming/
│           ├── scheduler.py
│           ├── orchestrator.py
│           ├── analyze_failures.py
│           ├── verify_lessons.py
│           └── publish_memory.py
├── workspace/
│   ├── sessions/
│   ├── memory/
│   │   ├── index.md
│   │   ├── sign_classes.md
│   │   ├── localization_rules.md
│   │   ├── road_association.md
│   │   └── known_failure_modes.md
│   ├── dreaming/
│   │   ├── candidates/
│   │   ├── reports/
│   │   └── snapshots/
│   └── exports/
└── tests/
    ├── fixtures/
    ├── test_camera_geometry.py
    ├── test_road_matching.py
    ├── test_heading.py
    ├── test_qa_rules.py
    └── test_dreaming.py
```

This is a target structure, not a requirement to create every file on
day one. Start with a small runnable pipeline and add modules as each
responsibility becomes real.

## 5. Components and responsibilities

Use small, testable Python modules. They do not need to be autonomous
LLM agents.

| Component | Responsibility | Default implementation |
|---|---|---|
| Pipeline orchestrator | Schedule stages, retries, dependencies, and job status | Python functions or a lightweight task runner |
| Observation normalizer | Validate and standardize detection payloads | Python and schema validation |
| Road association | Rank candidate OSM links using geometry and topology | GeoPandas, Shapely, PostGIS queries, scoring rules |
| Localization engine | Project camera rays, combine views, estimate position and uncertainty | Calibrated geometry and numerical optimization |
| Heading estimator | Estimate sign-face normal and confidence | Geometric calculations and explicit conventions |
| QA engine | Accept, flag, or reject estimates and record evidence | Versioned deterministic rules |
| Review workflow | Capture human corrections and reason codes | Database table and a simple review interface |
| Memory and Dreaming jobs | Analyze recurring failures and propose rule improvements | SQL, Python statistics, clustering, versioned Markdown/JSON |
| Memory retrieval | Load approved rules relevant to a processing job | Local files or database queries |

**Design rule:** calculations and acceptance decisions must be reproducible
from recorded inputs, algorithm versions, configuration, and rule versions.
No runtime LLM call should be required to process or validate a sign.

## 6. Data contracts

### 6.1 Raw detection

Treat the model output as immutable evidence. Store the original payload
or a durable reference to it.

``` json
{
  "detection_id": "det_000123",
  "image_id": "pano_0042",
  "model_version": "segmentation_v3",
  "class_label": "speed_limit",
  "class_confidence": 0.96,
  "bbox": {
    "x_min": 812,
    "y_min": 210,
    "x_max": 884,
    "y_max": 306
  },
  "mask_ref": "object-store://masks/det_000123",
  "raw_payload_ref": "object-store://raw/det_000123"
}
```

Add fields that your source actually provides, such as image dimensions,
panorama heading, camera location, pitch/roll, capture time, and camera
calibration identifier. Do not invent camera metadata when it is
unavailable.

### 6.2 Image observation

An observation is one view of a potential object. It may reference a raw
detection and the camera pose used for geometric calculations.

``` json
{
  "observation_id": "obs_000123",
  "detection_id": "det_000123",
  "image_id": "pano_0042",
  "track_id": "track_000045",
  "camera_pose_ref": "camera-pose://pano_0042",
  "pixel_or_panorama_angles": {
    "horizontal_deg": 18.4,
    "vertical_deg": -3.2
  },
  "observation_quality": "usable"
}
```

The angles above are illustrative. Define whether angles are relative to
the image, panorama, or true north, and document sign conventions.

### 6.3 Localized physical object

Choose and document the position reference. For example,
`estimated_pole_base` is useful when the goal is to place a sign on the
map; `sign_face_center` may be more appropriate for image-based
measurement.

``` json
{
  "object_id": "sign_000045",
  "class_label": "speed_limit",
  "position": {
    "longitude": 106.7001,
    "latitude": 10.7765,
    "reference_point": "estimated_pole_base",
    "crs": "EPSG:4326"
  },
  "heading": {
    "degrees": 274.0,
    "reference": "sign_face_normal_true_north_clockwise",
    "confidence": 0.87
  },
  "road_association": {
    "osm_way_id": "123456789",
    "side_of_road": "right",
    "association_confidence": 0.91
  },
  "uncertainty": {
    "horizontal_radius_m": 2.5,
    "definition": "illustrative_radius; replace with calibrated semantics"
  },
  "localization_method": "multi_view_ray_intersection_with_boundary_constraints",
  "source_observation_ids": ["obs_000123", "obs_000124"],
  "status": "needs_review"
}
```

All coordinates, IDs, confidence values, and uncertainty values in this
example are illustrative---not measured results. Define uncertainty
precisely, such as a 95% horizontal confidence radius or covariance
ellipse. Do not mix a standard deviation, a maximum error, and a
confidence radius under one field name.

### 6.4 QA result

``` json
{
  "qa_id": "qa_000045",
  "object_id": "sign_000045",
  "decision": "needs_review",
  "rule_version": "qa_rules_v1.0.0",
  "checks": [
    {
      "rule_id": "GEO-003",
      "result": "warning",
      "severity": "medium",
      "evidence": {
        "nearest_boundary_distance_m": 1.2
      }
    }
  ],
  "review_status": "pending"
}
```

Keep QA evidence machine-readable so it can be analyzed later by the
Dreaming process.

## 7. Geospatial localization strategy

### 7.1 Use a defined coordinate and orientation convention

-   Keep exchange coordinates in WGS84 longitude/latitude (EPSG:4326)
    when appropriate.
-   Perform distances, buffers, offsets, and local geometry calculations
    in a suitable projected CRS measured in metres. Select a CRS
    appropriate for the working area.
-   Store the source CRS and transformation method.
-   Define heading as degrees clockwise from true north, or explicitly
    choose another convention and convert consistently.
-   Store **sign-face normal**, **road-link bearing**, and **direction
    of traffic affected** as separate fields. They are not
    interchangeable.

### 7.2 Generate road candidates instead of selecting the nearest road blindly

For each observation, build a candidate set from a search region around
the camera ray or preliminary location. Score each candidate using
evidence such as:

-   distance from the estimated object position to the road or road
    edge;
-   camera viewing-ray compatibility;
-   road geometry and topology;
-   road class and plausible sign usage;
-   sidewalk/curb/road-boundary relationship;
-   sign-face orientation relative to candidate traffic directions;
-   consistency across multiple panoramas.

Keep several candidates when scores are close. Do not force a single
association if evidence is ambiguous.

### 7.3 Use boundaries as constraints, not ground truth

Sidewalk and curb geometries can help identify the likely roadside and
rule out implausible placements. However, a sign pole is not necessarily
located on the sidewalk boundary itself. The sign may be set back from
the curb, mounted on a median, attached to a building, or positioned on
a traffic island.

Model the relationship explicitly---for example, `roadside`, `median`,
`island`, `building-mounted`, `unknown`---and use locally validated
offsets or rules rather than one universal distance.

### 7.4 Estimate position from camera evidence

A segmentation mask gives an image region, not a 3D coordinate. A robust
localization approach can combine:

1.  Camera pose and calibrated image/panorama geometry.
2.  A ray or viewing direction derived from the detected pixels.
3.  Multiple observations of the same sign from different camera
    positions.
4.  Known scene geometry, depth, or surveyed reference points when
    available.
5.  Road-edge and sidewalk constraints.
6.  Optimization that minimizes reprojection error and penalizes
    geometrically implausible placements.

A single camera ray does not uniquely determine a 3D point. If there is
only one usable view and no depth or other constraint, return a coarse
estimate or `needs_review`; do not manufacture precision.

Separate the estimated sign-face location from the pole base when the
intended output needs a ground position. The bottom of a visible sign
mask is not guaranteed to be the pole base.

### 7.5 Estimate heading separately

Possible heading evidence includes:

-   sign-face plane or orientation inferred from image geometry;
-   multiple views of the same sign;
-   known sign shape and visible face;
-   relationship to the road and the traffic direction the sign
    controls.

Record the method and confidence. A road's OSM node order or way
direction alone does not establish the direction the sign faces.
Preserve alternatives if front/back classification is uncertain.

### 7.6 Cross-view matching and deduplication

Match detections across images before treating them as separate physical
objects. Potential features include:

-   distance and angle between camera rays;
-   compatible class and sign-face appearance;
-   reprojection consistency;
-   estimated roadside location;
-   capture sequence and camera trajectory;
-   existing localized-object candidates.

Use explicit matching scores and retain links to all raw detections.
Avoid deleting raw records when merging duplicate observations.

## 8. Processing pseudocode

This is an architectural sketch. Replace interfaces with your actual
storage and geometry implementations.

``` python
async def process_detection_batch(batch_id):
    raw_detections = await ingestion.load_batch(batch_id)

    observations = await observation_agent.normalize(raw_detections)

    candidates = await road_association.generate_candidates(
        observations=observations,
        road_network=road_network,
        sidewalk_boundaries=sidewalks,
    )

    tracks = await cross_view_matcher.match(
        observations=observations,
        road_candidates=candidates,
    )

    results = []

    for track in tracks:
        estimate = localization_engine.estimate(track)
        heading = heading_engine.estimate(track, estimate)

        qa_result = qa_agent.evaluate(
            estimate=estimate,
            heading=heading,
            observations=track.observations,
            road_candidates=track.road_candidates,
        )

        result = {
            "estimate": estimate,
            "heading": heading,
            "qa": qa_result,
        }
        await repository.save_result(result)
        results.append(result)

    await session_recorder.record_batch_outcomes(
        batch_id=batch_id,
        results=results,
    )

    return results
```

Keep geometry functions synchronous or asynchronous as appropriate for
the actual implementation. The important part is that every stage has an
explicit input/output contract and that failures are recorded.

## 9. QA rules and decision policy

Use rule IDs, versioned thresholds, and structured evidence. Thresholds
must be calibrated against reviewed ground-truth data; the examples
below define rule intent, not universal numeric cutoffs.

  -------------------------------------------------------------------------
  Rule ID                 Check                     Example failure or
                                                    warning
  ----------------------- ------------------------- -----------------------
  `DATA-001`              Required fields and CRS   Missing camera pose or
                          metadata exist            unknown coordinate
                                                    reference

  `GEO-001`               Estimate falls within a   Position too far from
                          plausible search region   all supporting
                                                    observations

  `GEO-002`               Projection/reprojection   Estimated object poorly
                          residual is acceptable    explains the observed
                                                    pixels

  `GEO-003`               Position has a plausible  Sign appears in the
                          relationship to road and  carriageway without
                          boundaries                supporting evidence

  `HEAD-001`              Heading is supported by   Heading depends only on
                          sign-face geometry        OSM way direction

  `ROAD-001`              Road association is       Candidate belongs to a
                          geometrically and         parallel road with weak
                          topologically plausible   evidence

  `DUP-001`               Repeated observations are Same physical sign
                          grouped consistently      exported as multiple
                                                    objects

  `EVID-001`              Evidence is traceable     Output cannot be linked
                                                    to source images or
                                                    detections
  -------------------------------------------------------------------------

Suggested decisions:

-   **Accept:** required evidence is present, uncertainty is within the
    calibrated limit, and no blocking rule fails.
-   **Needs review:** multiple candidates remain plausible, camera
    metadata is weak, or uncertainty is too large for automatic
    acceptance.
-   **Reject/unresolved:** evidence is inconsistent or required
    information is missing. Preserve the record and reason; do not
    silently discard it.

## 10. Shared memory and Dreaming

The system should learn from repeated sessions and reviewed outcomes,
but it should not automatically convert every model output or
unverified observation into a rule.

### 10.1 Memory layers

Use separate memory areas for different purposes:

-   **Stable project memory:** schema, coordinate conventions, data
    contracts, known system constraints.
-   **Approved operational rules:** reviewed rules for sign classes,
    road association, localization, and QA.
-   **Session traces:** what ran, inputs, outputs, tool calls, errors,
    and decisions.
-   **Dreaming proposals:** candidate lessons extracted from recurring
    patterns.
-   **Evaluation reports:** evidence that a proposed lesson improves
    performance on reviewed cases.

Example memory files:

``` text
workspace/memory/
├── index.md
├── sign_classes.md
├── localization_rules.md
├── road_association.md
├── heading_rules.md
├── qa_policy.md
└── known_failure_modes.md
```

### 10.2 Example memory content

`workspace/memory/road_association.md`

``` markdown
# Road association rules

## Parallel roads and frontage roads

- Do not assign a sign to the nearest road solely by Euclidean distance.
- Generate multiple candidate links when parallel roads are within the search region.
- Compare camera viewing-ray compatibility and cross-view consistency.
- Use sidewalk, curb, median, and road-edge geometry as constraints.
- Check road topology and plausible sign/traffic relationships.
- Keep candidate scores and unresolved alternatives in the QA record.
- Route ambiguous cases to review rather than forcing an association.
```

### 10.3 Dreaming cycle

``` mermaid
flowchart TD
    A[Session traces + QA outcomes] --> B[Group failures by root cause]
    B --> C[Draft candidate lesson]
    C --> D[Check evidence and duplication]
    D --> E[Evaluate on reviewed cases]
    E -->|Improves and passes safeguards| F[Publish versioned memory]
    E -->|Insufficient or harmful| G[Reject or revise proposal]
    F --> H[Future sessions retrieve approved lesson]
```

A practical Dreaming pass:

1.  Collect session traces, localization outcomes, QA results, and human
    correction reasons.
2.  Group recurring failures by likely root cause: camera calibration,
    low image quality, parallel roads, incorrect cross-view matching,
    heading ambiguity, or missing boundaries.
3.  Ask a lesson-extraction process to propose a concise rule and
    identify supporting cases.
4.  Check whether the lesson already exists and whether the evidence
    supports it.
5.  Evaluate the proposed rule against a fixed set of reviewed cases.
6.  Publish only approved memory updates with a version, date,
    author/process, evidence, and rollback path.
7.  Monitor subsequent results to verify the lesson continues to help.

### 10.4 Example Dreaming proposal

``` json
{
  "proposal_id": "dream_2026_001",
  "topic": "parallel_road_association",
  "observation": "A subset of sign estimates is repeatedly associated with the wrong road when two roads run in parallel.",
  "proposed_rule": "Retain multiple road candidates and score them using camera-ray fit, cross-view consistency, boundary context, and topology before selecting an association.",
  "supporting_case_ids": ["review_102", "review_119", "review_208"],
  "evaluation_status": "pending",
  "memory_target": "road_association.md"
}
```

The IDs and content above are illustrative. The proposal must point to
real cases in a production system.

### 10.5 Memory updates are not model or algorithm updates

Keep these changes separate:

-   **Memory update:** improves how the agent selects tools, applies
    approved rules, or routes ambiguous cases.
-   **Geometry/configuration update:** changes parameters, coordinate
    handling, thresholds, or optimization.
-   **Model update:** changes segmentation or matching model
    weights/version.

Geometry and model changes require reproducible experiments, regression
tests, and version tracking. A Dreaming process should not silently
modify production thresholds or rewrite historical outputs.

## 11. Provenance, reproducibility, and versioning

For each localized object, record at least:

-   IDs of all source detections and images;
-   source image and camera metadata version;
-   segmentation model version;
-   OSM extract timestamp or dataset version;
-   boundary dataset version;
-   localization algorithm/version and configuration hash;
-   heading algorithm/version;
-   QA rule version and results;
-   memory version used for the run;
-   human reviewer decision and correction reason, if applicable.

This makes it possible to explain why an object was placed at a given
position and to reproduce or compare a later result.

## 12. Storage design

A practical initial setup is:

-   **PostGIS:** objects, geometries, candidate road associations, QA
    results, and spatial queries.
-   **Object storage or a file store:** original images or references,
    masks, raw model payloads, and large artifacts.
-   **JSONL or structured event logs:** session traces and stage
    outcomes.
-   **Version-controlled Markdown/YAML:** approved memory,
    configuration, and QA policy.
-   **A review table or queue:** uncertain and rejected estimates,
    evidence links, reviewer decisions, and reason codes.

Do not store large image/mask payloads directly in every relational row
if references are sufficient. Preserve immutable raw records and write
new versions of derived estimates rather than overwriting source
evidence.

## 13. Testing and evaluation

Build a reviewed evaluation set before tuning thresholds. Include
difficult cases such as:

-   parallel roads and frontage roads;
-   divided roads and medians;
-   signs near intersections or roundabouts;
-   signs partially occluded by trees, vehicles, or buildings;
-   tilted cameras and imperfect panorama metadata;
-   signs seen from only one direction;
-   repeated views of the same sign;
-   signboards mounted on buildings rather than poles;
-   incomplete or misaligned sidewalk boundaries.

Track separate metrics rather than a single overall accuracy score:

-   horizontal position error (metres);
-   heading angular error (degrees);
-   road-association accuracy;
-   duplicate/merge precision and recall;
-   percentage automatically accepted;
-   review rate and unresolved rate;
-   error and coverage by sign class, road type, and image-quality band.

Evaluate memory proposals on the same fixed cases and a separate holdout
set where possible. A proposal should not pass only because it improves
the examples used to invent it.

## 14. Suggested implementation phases

### Phase 1 --- Data contract and baseline

-   Define immutable raw-detection and image-observation schemas.
-   Standardize camera pose and angle conventions.
-   Choose the object position reference (pole base, sign-face center,
    or another defined point).
-   Store a small set of examples in PostGIS or structured files.
-   Build a deterministic baseline that reports position candidates and
    uncertainty.

### Phase 2 --- Candidate roads and boundaries

-   Load the required OSM road network and boundary layers.
-   Transform geometry to a suitable metric CRS.
-   Generate multiple candidate roads rather than selecting nearest
    only.
-   Store candidate scores and evidence.

### Phase 3 --- Cross-view localization and heading

-   Match detections across panoramas.
-   Implement calibrated camera-ray geometry and multi-view estimation
    where metadata permits.
-   Estimate sign-face heading separately from road bearing.
-   Return uncertainty and an explicit unresolved status when evidence
    is insufficient.

### Phase 4 --- QA and review loop

-   Implement versioned spatial and semantic QA rules.
-   Build a review queue for uncertain cases.
-   Capture reviewer corrections as structured feedback.
-   Track metrics against reviewed ground truth.

### Phase 5 --- Memory and Dreaming

-   Record session traces and QA outcomes.
-   Extract recurring failure patterns.
-   Generate candidate memory changes.
-   Evaluate candidates on reviewed cases and publish approved versions
    only.

### Phase 6 --- Scale and monitoring

-   Add job retries, idempotency, run-level observability, and batch
    tracking.
-   Monitor error rates by geography, road type, class, and model
    version.
-   Keep regression tests and a rollback mechanism for
    rule/configuration releases.

## 15. Minimal first version

You do not need a multi-agent framework or vector database at the
beginning. A useful first version can be built with:

-   Python orchestration;
-   GeoPandas/Shapely or equivalent tested geospatial functions;
-   PostGIS for spatial data;
-   JSONL for execution traces;
-   version-controlled Markdown/YAML for approved memory and
    configuration;
-   a simple review interface.

Start with one end-to-end batch that produces a traceable object
estimate and a QA decision. Use Claude Code as a development assistant,
not as a service that the deployed application must call.

## 16. Key design decisions to settle early

Before implementing production localization, explicitly decide:

1.  What exactly does `position` mean: pole base, sign-face center, or
    another point?
2.  What camera metadata is available: camera GPS, heading, pitch, roll,
    field of view, panorama layout, and calibration?
3.  Are multiple images of the same sign available, and can their camera
    positions be trusted?
4.  Which boundary layers exist, what do they represent, and how
    accurate are they?
5.  What is the required positional and heading accuracy for automatic
    acceptance?
6.  How should ambiguous road associations be represented and reviewed?
7.  What source data and algorithm versions must be preserved for
    auditability?
8.  Which lessons may enter shared memory automatically, and which
    require human approval?

These choices determine whether the system can calculate a reliable
coordinate or only produce a candidate estimate.

## 17. Cost-conscious runtime policy

The default system is designed to run without a paid LLM API.

- **Development:** use Claude Code with your existing Claude Pro subscription to help write, refactor, and test the project. Subscription limits and terms still apply.
- **Runtime:** the deployed Python application must not call Anthropic's API, OpenAI's API, or another paid LLM API.
- **Core processing:** use Python, PostGIS, geospatial libraries, the segmentation model you already have access to, and deterministic QA rules.
- **Dreaming:** run scheduled SQL/Python analyses to find patterns, create reports, and propose changes. Require human approval before changing production rules or configuration.
- **Other possible costs:** this avoids a separate LLM API bill, but imagery services, hosting, storage, compute, or other third-party services may still charge fees.
- **No hidden dependency:** tests and deployment should work without API keys for an LLM provider. Do not add LLM API keys to `.env` unless you later approve a separate feature that requires them.

## 18. Implementation checklist

- [ ] Open the project in VS Code and ask Claude Code to inspect the existing repository before editing.
- [ ] Create or update `CLAUDE.md` with project conventions and the no-paid-LLM-runtime requirement.
- [ ] Confirm Python, PostGIS, OSM extracts, camera metadata, and boundary datasets.
- [ ] Define schemas for raw detections, image observations, localized objects, and QA results.
- [ ] Implement one end-to-end batch using deterministic Python processing.
- [ ] Add tests for CRS transformations, candidate-road ranking, localization, heading, and QA.
- [ ] Record provenance and processing versions.
- [ ] Add a review workflow and capture correction reasons.
- [ ] Implement scheduled SQL/Python failure reports for Dreaming.
- [ ] Verify the application runs without Anthropic/OpenAI API keys or runtime LLM calls.

## 19. References and implementation notes

-   OpenStreetMap elements and ways provide map geometry and node order;
    way direction should not be interpreted as the sign's facing
    direction. Consult the [OpenStreetMap Elements
    documentation](https://wiki.openstreetmap.org/wiki/Elements) and
    [forward/backward
    documentation](https://wiki.openstreetmap.org/wiki/Forward_%26_backward,_left_%26_right).
-   Use calibrated camera geometry and explicit CRS transformations
    rather than treating pixel coordinates as map coordinates.
-   Keep all sample IDs, coordinates, confidence values, and uncertainty
    values in this document as illustrative examples only.
-   The architecture is a starting point. Validate every production
    threshold and automated decision against reviewed data from the
    target geography.
