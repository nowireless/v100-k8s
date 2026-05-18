# Technical Training Guide for Building an FRC Game Piece Detector

## Architecture and the first engineering decisions

The simplest workflow that is still “real engineering” is this: collect raw robot-camera video and stills to S3-compatible storage; label curated snapshots in a browser tool; materialize a versioned training dataset onto Lustre for training; train a small pretrained detector first on a single GPU in a containerized Kubernetes Job; evaluate on held-out field-realistic videos instead of chasing mAP alone; export to ONNX; compile a TensorRT engine on the Jetson when practical; and publish a compact detection interface to the robot through NetworkTables first, with richer APIs only if you later prove you need them. That flow matches the strengths of Kubernetes Jobs, Ultralytics’ train/val/export lifecycle, TensorRT’s ONNX/engine workflow, Jetson deployment practice, and FRC-native robot networking. citeturn14view6turn28view5turn16view7turn14view4turn15view8

If you want the most opinionated “do this first” stack, use this baseline:

- **Detector:** Ultralytics **YOLO26n** for a greenfield project, with **YOLO11n** as the fallback if you need an older, more familiar baseline. Ultralytics explicitly positions YOLO26 as the edge-focused, NMS-free choice for new projects, while noting YOLO11 remains a strong production option. citeturn29view0turn28view1turn28view2
- **Training:** one **V100**, one **Kubernetes Job**, one reproducible container, one dataset version. Do not start with multi-node DDP. PyTorch recommends DDP with one process per GPU when you do scale, but the first goal is a stable single-GPU pipeline. citeturn20view1turn20view2
- **Storage:** **S3-compatible object storage** for source-of-truth artifacts and **Lustre** for high-throughput training scratch. DVC fits this pattern well because it is designed to synchronize large tracked data to remotes including S3-compatible ones; Lustre is explicitly a parallel filesystem for HPC environments. citeturn23view6turn14view10turn15view7
- **Evaluation target:** held-out robot videos from the actual camera mount, not just image-level validation. Aggregate detection metrics are useful, but FiftyOne’s evaluation docs are right that aggregate metrics alone do not tell the whole story of a detector’s behavior. citeturn16view13turn7search8
- **Deployment:** **TensorRT FP16 first**, then **INT8 only if benchmarking proves it matters** and only with representative calibration data. TensorRT supports FP16 and INT8, but INT8 requires calibration on representative inputs. citeturn14view4turn19view7turn19view5turn19view4
- **Robot interface:** **NetworkTables 4** for first ship. It is WPILib’s publish-subscribe system and is already native to the FRC robot network. citeturn36search6turn15view8

HPC is useful here, but only in a narrow, disciplined way. It is genuinely helpful for central data storage, repeatable containerized jobs, parallel experiment sweeps, and later multi-GPU or multi-node training. It is unnecessary for the first baseline, for exporting to ONNX, and for most Jetson benchmarking. PyTorch distributed and Kubeflow Trainer both exist to scale jobs across workers, but neither substitutes for a clean local training/debug path. citeturn20view2turn32view2turn14view6

### Model choice

For an FRC game piece detector, the main families are all viable, but they do not have equal “competition-week usefulness.”

**YOLO-style detectors** remain the best default for this problem because they combine strong real-time performance, straightforward transfer learning from pretrained weights, a mature export path to ONNX and TensorRT, and a large amount of ecosystem support. Current Ultralytics tooling exposes a clean lifecycle of train, val, predict, export, and benchmark, and its edge-focused families explicitly target ONNX and TensorRT deployment. YOLO11 is designed for efficient real-time use and edge deployment; YOLO26 goes further with end-to-end NMS-free inference, DFL removal, and an edge-first deployment story. citeturn28view5turn28view1turn28view2turn28view3

**RT-DETR** is interesting, and you should know why. The original RT-DETR paper positions it as a real-time end-to-end detector, and RT-DETRv2 brings transformer-style global context and NMS-free detection. That matters in clutter and occlusion. The tradeoff is that transformer detectors are typically more memory-hungry and usually have a weaker “boring deployment path” than the small YOLO families. Ultralytics’ comparison page also frames YOLO as the stronger speed/usability/economy choice for real-world production, while RT-DETR is the more research-forward transformer option. For a first FRC model, I would not start here. citeturn14view2turn33view0

**SSD/MobileNet-like detectors** still matter as the “tiny and cheap” baseline. Google’s edge documentation explicitly describes SSD MobileNetV2 as faster and lighter than EfficientDet-Lite0, but generally less accurate. The MobileNetV2 paper also introduced SSDLite specifically as a mobile object detection framework. If you hit an absolute latency wall on the Jetson or end up on a weaker coprocessor later, this family is a credible fallback. On an Orin Nano, though, it is usually a fallback, not the first choice. citeturn34view0turn16view8

**Classical CV** still has a role, but mostly as a sanity baseline, not the final competition detector. OpenCV contour-based pipelines, thresholding, and morphology can detect simple colored blobs and can be extremely fast. They also fail hard when exposure, white balance, carpet color, or ambient lighting move outside the tuned envelope. WPILib and PhotonVision docs repeatedly emphasize that exposure, brightness, and resolution choices materially change downstream vision behavior. In practice, classical CV is valuable for bootstrapping data collection, for a fallback “green blob exists” indicator, or for highly constrained color targets—but not usually as the final answer for competition-grade game piece detection. citeturn16view9turn16view10turn22view0turn15view16

My practical recommendation is:

- **First shipped model:** **YOLO26n**, 640 input, pretrained initialization, single-class or very small class set, FP16 TensorRT on Jetson. Ultralytics now positions YOLO26 as the greenfield edge choice, especially for latency-critical robotics. citeturn29view0turn29view4turn28view2
- **Fallback baseline:** **YOLO11n** if your team already has tooling or if you want a highly recognized baseline with broad community familiarity. citeturn29view0turn33view0
- **Do not start with:** RT-DETR, multi-task heads, segmentation, or multi-node DDP. Those are second-phase experiments. citeturn33view0turn20view1

### Model size tradeoffs on Jetson Orin Nano

The Jetson Orin Nano Super Developer Kit is an 8 GB system with 67 INT8 TOPS in a 7 W–25 W envelope, which is enough for serious edge inference but not enough to absorb a sloppy pipeline. You need to think of the system as **camera ingest + resize + color conversion + inference + postprocess + serialization + network publish**, not just “model latency.” NVIDIA’s own Jetson power tooling uses `nvpmodel` to select power modes and `jetson_clocks` to inspect clock behavior, so you should benchmark only after standardizing those settings. citeturn15view0turn15view1

Ultralytics’ Jetson guide gives a useful order-of-magnitude reference for a nano model on an **Orin Nano Super**: roughly **15.6 ms** in PyTorch, **7.53 ms** in TensorRT FP32, **4.57 ms** in TensorRT FP16, and **3.80 ms** in TensorRT INT8, while explicitly noting that those inference times **exclude pre/post-processing**. That last caveat matters enormously in robotics, because your real pipeline latency often gets dominated by capture, resize, copy, and serialization once the model itself is fast. citeturn19view1turn19view2

The engineering consequence is straightforward. On Orin Nano, the safe default envelope is:

- **nano model** for first deployment,
- **small model** only if benchmarking demonstrates headroom at your actual camera resolution and frame rate,
- **medium and larger** only if your controls team can prove the extra accuracy matters more than the lost throughput and latency.

That recommendation is an inference from the Jetson hardware envelope and the published per-format timing patterns above. citeturn15view0turn19view1turn19view2

For precision modes, treat them this way:

- **FP16** is the default deployment target. TensorRT supports it directly, Ultralytics exposes it with `half=True`, and it usually gives most of the speedup with much less project risk than INT8. citeturn14view4turn18view1
- **INT8** is an optimization project, not a starting point. TensorRT calibration is only applicable to INT8, requires representative input data, and Ultralytics notes that INT8 calibration should be performed on the same deployment device because calibration results can vary across devices. citeturn19view5turn19view7turn18view0turn18view3

## Data and labeling discipline

### Dataset strategy for FRC

Your dataset needs to capture the robot’s actual visual world, not your team’s best-looking practice image set. Ultralytics’ training guidance is blunt about this: image variety must be representative of the deployed environment, across lighting, angles, sources, and other condition changes, and validation/test images must not appear in training. That general guidance maps almost perfectly to FRC. citeturn31view0turn15view14turn15view15

For FRC, collect from at least these contexts:

- **Practice field and real field** if possible.
- **Shop floor, hallway, and pit-like spaces** to create negative and off-domain examples.
- **Carpet, non-carpet, and multiple carpet colors/textures** to avoid learning “your shop floor” instead of the game piece.
- **Robot-mounted cameras only**, at the exact heights, mounts, lenses, and exposure settings you expect to use in matches.
- **Source / intake / shooter / human-player perspectives**, because those are the robot behaviors that will actually consume the output.
- **Distance bands** from near-intake range to the farthest operationally relevant pickup/vision-assist range.
- **Angles and partial views** caused by strafing, turning, pitch changes, chassis vibration, and approach paths. citeturn31view0turn22view0turn15view17

Lighting and motion are not side details. PhotonVision’s camera tuning docs emphasize that higher resolution can improve accuracy but reduces update rate, and that exposure should be as low as possible while preserving reliable tracking because lower exposure can increase FPS and reduce motion blur. They also explicitly call out motion blur as a robotics problem and recommend global-shutter cameras when possible for fast-moving vision tasks. Even though those docs are written around AprilTags and contour pipelines, the camera physics applies just as much to game-piece detection. citeturn22view0turn22view2turn22view3

Positive and negative examples both matter. Ultralytics recommends including some background-only images to reduce false positives, on the order of roughly 0–10% of the dataset. For FRC, that means frames with carpet, bumpers, human legs, field walls, loading station clutter, reflections, and anything else your detector might hallucinate into a game piece. If your model never sees “empty but confusing” scenes, it will overconfidently invent detections in competition. citeturn31view0

Do not overfit to one shop, one field, one camera, or one carpet. The easiest way to accidentally do this is to record a few beautiful practice sessions, sample every fifth frame, and declare victory. Google’s dataset-splitting guidance is clear that duplicates across train/val/test create overly optimistic evaluation, and scikit-learn’s general leakage guidance says the same thing in broader ML terms. For video, assume adjacent frames are near-duplicates unless proven otherwise. Split by **clip**, **session**, **event**, or **day**, not by random frame. citeturn15view14turn15view15

“How much data is enough?” There is no universal minimum, but there is a sane way to think about it. Ultralytics’ older training guidance gives a “best results” target of roughly **1500 images per class** and **10000 instances per class**, which is a useful ambition marker, not a first-robot gate. For a first FRC detector, I would aim for **hundreds of distinct scenes** and **at least several hundred to a few thousand labeled instances per class**, provided they span the real operation modes above. If performance is unstable, your next move is usually **more diverse field-realistic data**, not more hyperparameter cleverness. citeturn31view0

### Labeling workflow

For a student-heavy team, I recommend **CVAT first**. It is explicitly an image/video annotation tool, supports multiple export formats, and has built-in validation subsets, review mode, and quality analytics. That combination matters more than raw UI polish when you want students to work in parallel without turning labeling into chaos. citeturn12search10turn16view11turn16view12turn14view13turn14view14turn23view8

Good alternates exist. **Roboflow Annotate** is fast and student-friendly, including AI-assisted labeling paths. **Label Studio** is flexible and broad if you already use it for other annotation work. But if the specific goal is a reviewable, controlled object-detection workflow for a small team, CVAT’s QA features make it the better default. citeturn16view15turn16view16turn15view18turn15view19

For **class design**, keep it as small as possible. Start with one class per physical object category you truly need to distinguish in robot behavior. Do **not** create classes like `piece_far`, `piece_near`, `piece_blurry`, or `piece_in_source` unless the downstream robot logic genuinely needs them as separate semantic categories. Geometry, occlusion, and distance should usually be absorbed by model generalization and downstream filtering, not by exploding the class taxonomy. That is an engineering recommendation, but it is strongly supported by the fact that detector toolchains assume a clean class dictionary and depend heavily on consistent labels. citeturn35view2turn31view0

For **bounding boxes**, use one convention and document it. Ultralytics’ dataset format expects normalized `class x_center y_center width height`, and their label-quality guidance says boxes should closely enclose the object with no unnecessary space. For FRC, I recommend boxes that tightly enclose the **visible extent** of the object, not a guessed full object outline behind occlusion. That keeps the labeling rule teachable and repeatable. citeturn35view3turn31view0

A student review loop that actually works looks like this:

- first-pass labeling by one student,
- second-pass review on a fixed sample rate by another student,
- a small **ground-truth subset** maintained by a lead reviewer,
- immediate rejection of labels that violate the style guide,
- periodic model-assisted pre-annotation only **after** the first clean seed set exists.

CVAT directly supports validation subsets, ground-truth comparison, job validation, and review mode, so you do not need to invent this from scratch. citeturn14view13turn14view14turn23view8turn23view9

For **splits**, use **train / val / test** at the **recording-session level**. If you extract from video, assign the entire clip—or at least an entire contiguous segment—to exactly one split before frame extraction. Google’s guidance is explicit that duplicates across splits bias evaluation; with robot video, random frame-level splitting is leakage by default. citeturn15view14turn15view15

### Storage and versioning

Use **S3-compatible object storage** as the durable source of truth for **raw videos, curated image snapshots, labeled datasets, experiment artifacts, exported ONNX models, TensorRT engines, and benchmark reports**. MinIO is explicitly S3-compatible and is positioned for AI/ML and data-intensive workloads; DVC supports S3 and S3-compatible remotes directly. citeturn16view3turn23view6

Use **Lustre** as the fast workspace for training. Lustre describes itself as a parallel filesystem for leadership-class HPC environments, which is exactly why it is a good place to materialize shuffled image trees, dataloaders, cache directories, and transient run outputs during training. Raw truth should still live in object storage; Lustre should be treated as reproducible scratch or staged working data. citeturn15view7

A directory layout that works well is:

```text
frc-vision/
  data/
    raw/
      2026/
        kickoff/
        practice-field/
        scrimmage/
    curated/
      frc-piece-v0/
        images/
        manifests/
    labeled/
      frc-piece-v0.1.0/
        images/
        labels/
        data.yaml
        metadata.json
    calibration/
      front_cam/
      intake_cam/
  artifacts/
    train/
    eval/
    export/
    benchmark/
  deploy/
    jetson/
    robot/
  tools/
  notebooks/
  docs/
```

For metadata, keep a machine-readable manifest per asset group. At minimum, store: dataset version, source clip IDs, camera ID, lens/FOV, resolution, exposure/gain mode, frame extraction rule, annotation tool export version, label guide revision, split assignment, checksum, and who approved it. If you want a single-file dataset representation for Ultralytics, their newer NDJSON option is worth knowing about, but for team readability the classic YOLO image/label tree plus a versioned manifest is still the easiest place to start. citeturn35view0turn35view2

For versioning strategy, my recommendation is:

- **DVC + S3-compatible remote** if you want the best balance of simplicity and real data lineage. DVC is built for synchronizing large files/directories to remotes, and its docs explicitly point to S3-compatible remotes. citeturn14view10turn23view6
- **LakeFS** only if you truly want Git-like branching/merging directly on object storage and your team will actually use zero-copy branching and object-store-native workflows. It is powerful, but often unnecessary for a small FRC team unless you already think in data-platform primitives. citeturn14view11
- **Git LFS** only for relatively small curated assets, schema files, or perhaps tiny example datasets. Git LFS stores pointers in Git and the actual large files elsewhere, which is useful, but it is not how I would manage raw robot video at scale. citeturn14view12
- **Simple manifest-based versioning** is acceptable if the team is disciplined: immutable dataset folders, semantic dataset versions, checksums, explicit changelogs, and no in-place mutation. This is not as feature-rich as DVC, but it ships.

## Storage, containers, and training on HPC Kubernetes

### Reproducible containerized training

Use a **container image** as the immutable training environment. NVIDIA’s framework-container guidance emphasizes containers as reproducible, reliable execution environments, and NGC’s PyTorch containers are the obvious base image family if you want a CUDA-validated starting point. NVIDIA also recommends extending images with Dockerfiles rather than `docker commit`, which aligns with every sane reproducibility practice you already know from infra work. citeturn25view3turn25view1

A clean training image usually has three layers:

1. **Base CUDA/PyTorch image** from NGC or an equivalent pinned CUDA image.
2. **Framework layer** with `ultralytics`, `onnx`, `onnxruntime-gpu`, `pycocotools`, `mlflow`, `boto3`, and your evaluation utilities.
3. **Project layer** with your training code, config schemas, and entrypoints.

Do not bake changing datasets into the image. Mount them or stage them at runtime. That keeps image provenance separate from dataset provenance. This is an engineering recommendation grounded in container best practice and ML reproducibility. citeturn25view3turn25view1

### GPU jobs, scaling, and data staging

Kubernetes Jobs are the right primitive for this work because Jobs are one-off tasks that run to completion. Kubernetes’ GPU scheduling docs also make two practical points you should mirror in every manifest: GPUs are exposed via device plugins such as `nvidia.com/gpu`, and GPU quantities should be specified in `limits` with request/limit equality semantics. If your cluster uses NVIDIA’s GPU Operator, it automates the drivers, device plugin, container toolkit, node labeling, and monitoring stack. citeturn14view6turn32view0turn32view1

For the first phase, run **single-GPU, single-process** training. Only after the entire loop is stable—dataset staging, logs, checkpoints, metrics, export, and repeatability—should you turn on DDP. PyTorch’s guidance is clear: DDP is a multi-process approach with one process per model replica, and each GPU should map to exactly one DDP process. It is normally faster than `DataParallel`, but it is still complexity you do not need on day one. citeturn20view1turn20view2

InfiniBand matters only when you start synchronizing gradients across processes or nodes. In a single-GPU baseline, your bottlenecks are usually data quality, CPU preprocessing, filesystem reads, and model/export iteration speed—not interconnect bandwidth. Once you move into multi-node DDP or Kubeflow Trainer, then high-speed network fabric becomes relevant. Kubeflow’s Trainer docs make clear that distributed PyTorch jobs are ultimately about orchestrating workers and `torchrun` environment setup for gradient synchronization. citeturn32view2turn20view1

Stage data from object storage to Lustre before training. The pattern I would use is:

1. resolve a dataset version identifier;
2. fetch its manifest from S3-compatible storage;
3. sync or copy the exact dataset to a Lustre scratch path;
4. verify checksums or counts;
5. train against the staged path;
6. push only artifacts and metrics back to object storage.

That pattern avoids training directly against object storage latency while preserving object storage as the durable truth. It is an inference from the respective roles of S3-compatible remotes and Lustre’s parallel I/O model. citeturn16view3turn23view6turn15view7

### Experiment tracking and sweeps

Use **MLflow Tracking** unless your team already has a stronger preference. MLflow explicitly tracks parameters, code versions, metrics, and output files, and its dataset tracking module records lineage from raw/training/evaluation data to predictions. That is enough structure for a serious FRC vision program without forcing students into platform theater. citeturn14view8turn16view1

Track at least these metrics every run:

- precision,
- recall,
- mAP50,
- mAP50-95,
- per-class AP,
- confusion matrix,
- confidence-threshold sweeps,
- export latency by backend,
- end-to-end Jetson FPS/latency,
- representative false-positive and false-negative clips.

Ultralytics’ training tips explicitly call out precision, recall, mAP, PR curves, and confusion matrices as part of training-result review. citeturn31view0

For sweeps, keep them boring. Sweep only a handful of things first: model size (`n` vs `s`), image size (often `640` vs `960` if game pieces are small), confidence/NMS thresholds when applicable, and a few augmentation toggles. Do **not** start by searching twenty knobs. The fastest path to a better FRC detector is usually better data slices and more realistic evaluation videos, not aggressive HPO. That is a recommendation, but it is consistent with Ultralytics’ own advice to establish a default baseline before changing settings. citeturn31view0

## Evaluation, export, and Jetson runtime

### Evaluation for robotics

mAP is necessary, but it is not sufficient. COCO-style metrics are standardized and useful, and Ultralytics reports them cleanly, but they do not capture the control-system costs of latency spikes, false positives near your intake, temporal flicker, or failures under auto-exposure changes. FiftyOne’s evaluation guidance is especially apt here: aggregate results matter, but they do not tell the whole story; you need to study failure modes directly. citeturn7search8turn16view13

The robotics metrics that matter most are:

- **end-to-end latency**, not just model time;
- **FPS under real camera load**;
- **false positives** in operationally dangerous zones;
- **false negatives** at the ranges where your robot needs help;
- **frame-to-frame stability**;
- **threshold sensitivity**;
- **sensitivity to exposure, blur, and resolution changes**.

PhotonVision’s camera docs are a useful reminder that higher resolution typically trades for lower update rate, and lower exposure can improve both FPS and blur behavior. That means your validation set needs to include realistic camera settings, not only nicely exposed stills. citeturn22view0turn22view2

Build a **held-out field-realistic validation video suite**. Score it clip-by-clip, not just frame-by-frame. I strongly recommend slices like:

- straight-line approach to intake,
- turning while acquiring,
- contact with another robot nearby,
- human-player station clutter,
- low light / overly bright field,
- partial occlusion by bumpers or mechanisms,
- motion-blur passes,
- empty-field negatives.

Keep the suite immutable between major iterations so you can compare model versions honestly. This recommendation follows directly from the anti-leakage guidance for duplicates and from the fact that robotics failures are often temporal and contextual, not visible in static aggregate dashboards. citeturn15view14turn15view15turn16view13

Confidence-threshold tuning is operationally important. The detector API itself is built around score thresholds and top-k controls in many edge runtimes, including Google’s and WPILib-adjacent tooling. Your controls team may prefer “fewer but cleaner detections” over maximum recall. Tune thresholds against real robot tasks, not abstract leaderboard instincts. citeturn34view0

### Export and deployment

PyTorch-to-ONNX should be your stable interchange step. PyTorch documents ONNX as an open standard that enables execution across a wide range of hardware and runtime environments, from large cloud systems to constrained edge devices. In practice, this gives you a clean seam between training and deployment. citeturn16view7

TensorRT is the deployment target when you want best Jetson performance. NVIDIA describes TensorRT as the SDK that takes models from PyTorch, TensorFlow, and ONNX and optimizes them for high-performance deployment with support for mixed precision and dynamic shapes. `trtexec` is the standard command-line tool for benchmarking and engine generation. citeturn14view4turn14view5turn16view4

Compile TensorRT engines **on the Jetson** when practical, especially for final release artifacts. NVIDIA’s support matrix says serialized engines are not portable across platforms and, without hardware compatibility mode, are not portable across different GPU architectures. NVIDIA’s quick-start guide also notes that engine building is time-consuming and is usually done offline. For a robot workflow, that means: keep ONNX as the portable artifact; build the final engine on the target Jetson or on a fully identical Jetson software/hardware image. citeturn37search4turn37search2

For precision modes:

- **FP16 first.** It reduces model size and often speeds inference substantially with very little extra complexity. In Ultralytics export, that is the `half=True` path. citeturn18view1turn19view1
- **INT8 second.** TensorRT states that calibration is only applicable to INT8 and requires representative inputs to compute tensor scales. It also notes that calibration-cache reuse is possible but that recalibration is your responsibility if the network or input data changes. Ultralytics adds an important operational warning: use the same device for INT8 export/deployment because calibration can vary across devices, and provide `data=...` so the exporter uses representative validation images rather than a default example dataset. citeturn19view5turn19view6turn18view0turn18view3

For INT8 calibration data, do not invent a separate “calibration dataset” aesthetic. Use a few hundred representative validation images that contain the actual field lighting, exposure, blur patterns, distances, and negatives you expect on the robot. TensorRT says the needed amount is application-dependent and gives about 500 images as a reference point for ImageNet classification networks; for robotics detection, the right answer is still “representative, diverse, held-out frames.” citeturn19view5

### Runtime benchmarking and camera pipeline choices

Every benchmark should record all of the following: JetPack version, TensorRT version, camera type, camera resolution, color format, power mode, clocks, model precision, batch size, and whether preprocessing/postprocessing is included. This sounds obvious, but official Jetson benchmark tables often exclude pre/post-processing, and that is exactly where robotics teams lose their real-time budget. citeturn19view0turn19view1turn19view2

On Jetson, standardize power and clocks before comparing runs. NVIDIA documents `nvpmodel -q` / `nvpmodel -m x` and `jetson_clocks --show` for setting and inspecting supported power modes. If you benchmark one model at a different power state or thermal condition than another, you are not comparing models; you are comparing chassis heat. citeturn15view1

For camera ingest, there are three sane paths:

- **OpenCV `VideoCapture`** if you want maximum simplicity and quick iteration. It is the simplest camera/file ingest API and is fine for many USB cameras. citeturn15view5
- **GStreamer + `appsink`** if you want explicit pipeline control, lower-copy integration, or Jetson-specific camera/codec paths. GStreamer describes `appsink` as the sink that lets the application get a handle to pipeline data. citeturn9search3turn9search22
- **DeepStream** if you want a production-grade NVIDIA media/inference pipeline, especially for multiple streams or batched inference. DeepStream’s `gst-nvinfer` works over TensorRT and handles input transforms into the network format. citeturn14view19

For **ZED vs USB camera**, my opinionated recommendation is:

- **Start with a plain USB camera**, ideally global shutter if you can get it. It is mechanically and operationally simpler, easier to benchmark, and easier to keep deterministic in a competition robot.
- Use **ZED** only if you know you need stereo depth or the rest of the ZED SDK stack. Stereolabs positions ZED as a stereo/depth/spatial-AI camera family and provides Jetson-specific install flows, but its own depth docs also note that depth accuracy degrades with distance and can become unstable on homogeneous or reflective surfaces. That is a real tradeoff on FRC fields, where carpets, glare, polycarbonate, and low-texture backgrounds are common. citeturn14view18turn15view4turn23view2turn15view3

One specific ZED advantage is that the system is passive rather than IR-projecting, so multiple sensors do not interfere with each other and sunlight is not an issue in the same way as active depth sensors. That matters more outdoors than in FRC, but it is still technically useful context. citeturn23view0

## Robot integration, student development, and season execution

### Robot integration

For first ship, publish detections over **NetworkTables 4**. WPILib describes NetworkTables as a publish-subscribe messaging system across the robot network, and WPILib’s own basic vision example already demonstrates publishing vision-derived target information to NetworkTables. This is the shortest path from Jetson to robot code and dashboards. citeturn36search6turn15view8turn14view16

Do **not** start by publishing raw full detection arrays if your controls code only needs one target. Publish a compact interface first:

- `hasTarget: bool`
- `timestamp_ns: int`
- `latency_ms: double`
- `confidence: double`
- `yaw_deg: double`
- `pitch_deg: double`
- `range_m_est: double`
- `class_id` or `class_name`
- optional `allDetectionsJson: string` for debugging

That minimizes network chatter and makes controls integration boring. It also helps driver/debug UI. This is an engineering recommendation aligned with NT4’s topic model. WPILib’s topic/publisher docs also note that publishers are type-specific and remain active only while the publisher object exists, so your Jetson process should create publishers once and keep them alive for the lifetime of the program. citeturn36search0turn36search4turn36search7

Coordinate frames should follow WPILib’s conventions. WPILib uses the **NWU** frame in most robotics contexts: positive **X forward**, positive **Y left**, positive **Z up**, with CCW positive. Keep your camera-frame-to-robot-frame transform explicit and written down. If your team gets this wrong, the detector will look fine on screen and still sabotage controls. citeturn21view0turn21view2

Camera calibration is not optional if you want angle and range estimates that are better than “sort of looks right.” OpenCV’s calibration docs explain that calibration requires known 3D points and corresponding 2D image points, and that reprojection error is the basic quality check. OpenCV also notes at least 10 calibration patterns as a starting point. WPILib’s AprilTag documentation is equally direct: accurate camera calibration is required for reliable geometric pose estimation. citeturn15view12turn15view13turn21view3

For approximate **distance and angle estimation** from a detected box:

- angle from image center is a direct pinhole-camera inference from `fx`, `fy`, `cx`, `cy`;
- range from box size is only trustworthy if the real object size is known and the object is not heavily rotated out of plane;
- if you have stereo depth from ZED, use it only after validating its behavior on your actual field surfaces and distances. OpenCV’s calibration and projection docs and Stereolabs’ depth docs support those distinctions. citeturn23view5turn15view12turn15view13turn15view3

The highest-value fusion pattern is to combine **AprilTags for field/robot pose** with **object detection for game-piece localization in the camera image**. WPILib’s AprilTag docs explain the chain: calibrated camera → tag-relative camera pose → field tag lookup → field-relative camera pose → robot pose if camera extrinsics are known. Once the robot pose is known, the game-piece bearing from the detector becomes much more actionable for navigation or assistive pickup logic. WPIcal also exists to create calibrated field-layout JSON for imperfect practice fields. citeturn21view3turn21view4

Failure modes should be made visible to drivers and controls. At minimum, surface:

- stale detections,
- low-confidence mode,
- camera disconnect,
- exposure saturation or darkness,
- FPS drop,
- calibration invalid / unknown,
- “detection seen but range estimate unreliable.”

A detector that occasionally lies without telling the operator is worse than a detector that occasionally says “unknown.” That is not a doc quote; it is the correct robotics design principle.

### Student training plan

A six-week student curriculum that usually works is:

**Week one:** vision system architecture. Students learn the full path from camera to label to model to Jetson to robot code. Exercise: stand up the repo, run a pretrained detector in a container, and publish a dummy topic to NT4.

**Week two:** data collection and annotation. Exercise: define the label guide, record three camera scenarios, label 200–400 images, and review them against the style guide.

**Week three:** train the baseline. Exercise: run the first single-GPU experiment, inspect PR curves/confusion matrix, and prepare a failure-gallery notebook or FiftyOne session. Ultralytics explicitly recommends reviewing precision/recall/mAP and confusion matrices for training results. citeturn31view0turn16view13

**Week four:** Jetson deployment. Exercise: export ONNX, build FP16 TensorRT, benchmark on the Jetson under fixed power mode, and compare end-to-end latency including pre/post-processing. TensorRT and Jetson docs support exactly this style of benchmarking and power normalization. citeturn14view5turn16view4turn15view1

**Week five:** robot integration. Exercise: publish `hasTarget`, `yaw_deg`, `range_m_est`, and `latency_ms` to NetworkTables; consume them in robot code; build a dashboard indicator; verify coordinate-frame sign conventions. citeturn15view8turn14view16turn21view0

**Week six:** debugging and failure analysis. Exercise: produce a top-20 false-positive gallery, a top-20 false-negative gallery, and one concrete dataset or deployment fix for each family of failure. FiftyOne’s guidance about failure-mode analysis makes this a much better teaching loop than “ran training again with different epochs.” citeturn16view13

Suggested student roles:

- **data collection lead**
- **annotation / QA lead**
- **training / experiment lead**
- **Jetson deployment lead**
- **robot integration lead**
- **evaluation / benchmarking lead**

Rotate at least one student through two roles so nobody treats “their part” as magic.

### FRC season workflow

The right seasonal workflow is the one that ships before your third event, not the one that looks most like a startup pitch deck.

**Kickoff weekend:** define the task narrowly. Decide what the model must do for the first release. Usually that is something like “detect loose game pieces from the intake camera and report bearing/confidence.” Record synthetic stand-ins immediately if the real game pieces are not yet in hand, but treat that only as bootstrap data.

**Early prototype dataset:** within the first few days, collect robot-mounted footage in the shop and label enough to prove the training/export/deploy loop. The goal is not accuracy. The goal is pipeline integrity.

**Week one baseline:** train the first model from pretrained weights with defaults. Ultralytics’ own training guidance recommends exactly that—train defaults first to establish a baseline before changing settings. If you cannot produce one end-to-end baseline by the end of week one, the problem is not missing distributed infrastructure. citeturn31view0

**Weeks two and three:** replace synthetic assumptions with real robot-camera data. Re-record from the real mount, real intake, real controls maneuvers, and real illumination. This is usually where dataset quality overtakes all architecture arguments.

**Weeks four and five:** tune confidence thresholds, camera exposure, and resolution on the actual robot. PhotonVision and WPILib documentation both reinforce how much camera settings and resolution affect performance and throughput. citeturn22view0turn15view17

**Competition phase:** collect difficult clips from practice matches and dead time, but be disciplined. Do not continuously retrain between matches unless your workflow is already reliable and fast. The competition-useful loop is: capture failures, sort them into failure families, patch the dataset, retrain once between events, benchmark, redeploy, and lock the build.

The rule that keeps ML from becoming a science project is simple: **if the model does not inform a robot action or driver decision by a fixed date, freeze scope rather than adding sophistication**.

## Starter implementation, risks, and limits

### Concrete starter repository layout

```text
frc-gamepiece-vision/
  README.md
  pyproject.toml
  requirements.txt
  Dockerfile
  .env.example

  configs/
    train/
      yolo26n_640.yaml
      yolo11n_640.yaml
    deploy/
      jetson_fp16.yaml
      jetson_int8.yaml

  data/
    manifests/
    calibration/

  datasets/
    frc_piece_v0.1.0/
      data.yaml
      images/
        train/
        val/
        test/
      labels/
        train/
        val/
        test/
      metadata.json

  tools/
    stage_dataset.py
    sync_artifacts.py
    benchmark_trt.sh
    extract_video_frames.py
    split_by_session.py
    export_onnx.py
    build_engine.py

  train/
    train.py
    evaluate.py
    export.py

  deploy/
    jetson/
      infer_camera.py
      publish_nt.py
      trt_runner.py
      camera_gst.py

  robot/
    java/
    cpp/
    python/

  k8s/
    train-job.yaml
    pvc-lustre.yaml

  artifacts/
```

Use the classic Ultralytics dataset YAML first. Their docs define a dataset root with `path`, split paths with `train`, `val`, `test`, and a `names` dictionary for classes, while object labels are stored one text file per image in normalized `xywh` format. citeturn35view0turn35view2turn35view3

### Example dataset YAML

```yaml
path: /workspace/datasets/frc_piece_v0.1.0
train: images/train
val: images/val
test: images/test

names:
  0: gamepiece
```

### Example Kubernetes Job YAML

This is intentionally plain Kubernetes, not a full Trainer CRD. The first version should be easy to reason about.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: frc-piece-train-v010
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      nodeSelector:
        accelerator: nvidia-v100
      containers:
        - name: trainer
          image: nvcr.io/nvidia/pytorch:25.12-py3
          workingDir: /workspace
          command:
            - bash
            - -lc
            - |
              set -euxo pipefail
              python tools/stage_dataset.py \
                --dataset-version frc_piece_v0.1.0 \
                --s3-endpoint ${S3_ENDPOINT} \
                --bucket ${S3_BUCKET} \
                --dest /lustre/${JOB_NAME}/dataset

              yolo detect train \
                model=yolo26n.pt \
                data=/lustre/${JOB_NAME}/dataset/data.yaml \
                imgsz=640 \
                epochs=150 \
                batch=64 \
                device=0 \
                project=/lustre/${JOB_NAME}/runs \
                name=yolo26n_640_baseline

              python tools/sync_artifacts.py \
                --src /lustre/${JOB_NAME}/runs/yolo26n_640_baseline \
                --s3-endpoint ${S3_ENDPOINT} \
                --bucket ${S3_BUCKET} \
                --dst artifacts/train/frc_piece_v0.1.0/yolo26n_640_baseline
          env:
            - name: JOB_NAME
              value: frc-piece-train-v010
            - name: S3_ENDPOINT
              valueFrom:
                secretKeyRef:
                  name: s3-creds
                  key: endpoint
            - name: S3_BUCKET
              value: frc-vision
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: s3-creds
                  key: access_key
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: s3-creds
                  key: secret_key
          resources:
            requests:
              cpu: "8"
              memory: "32Gi"
            limits:
              cpu: "16"
              memory: "64Gi"
              nvidia.com/gpu: "1"
          volumeMounts:
            - name: lustre
              mountPath: /lustre
      volumes:
        - name: lustre
          persistentVolumeClaim:
            claimName: lustre-pvc
```

This is aligned with Kubernetes Jobs as run-to-completion workloads and with Kubernetes GPU scheduling via `nvidia.com/gpu` device-plugin resources. citeturn14view6turn32view0turn32view1

### Example training, evaluation, export, and Jetson commands

Use pretrained weights first. Ultralytics explicitly recommends pretrained initialization for custom training and exposes consistent train/val/export modes. citeturn28view5turn35view3turn28view1

```bash
# Train baseline on HPC
yolo detect train \
  model=yolo26n.pt \
  data=datasets/frc_piece_v0.1.0/data.yaml \
  imgsz=640 \
  epochs=150 \
  batch=64 \
  device=0 \
  project=artifacts/train \
  name=yolo26n_640_baseline

# Validate
yolo detect val \
  model=artifacts/train/yolo26n_640_baseline/weights/best.pt \
  data=datasets/frc_piece_v0.1.0/data.yaml \
  imgsz=640 \
  device=0 \
  project=artifacts/eval \
  name=yolo26n_640_val

# Export portable ONNX
yolo export \
  model=artifacts/train/yolo26n_640_baseline/weights/best.pt \
  format=onnx \
  imgsz=640 \
  simplify=True

# Build FP16 TensorRT engine on the Jetson
yolo export \
  model=artifacts/train/yolo26n_640_baseline/weights/best.pt \
  format=engine \
  imgsz=640 \
  half=True \
  device=0 \
  workspace=4

# Build INT8 TensorRT engine on the Jetson using representative data
yolo export \
  model=artifacts/train/yolo26n_640_baseline/weights/best.pt \
  format=engine \
  imgsz=640 \
  int8=True \
  data=datasets/frc_piece_v0.1.0/data.yaml \
  batch=8 \
  device=0 \
  workspace=4
```

The ONNX step is grounded in PyTorch’s exporter and TensorRT’s ONNX workflow, while the TensorRT export flags correspond to Ultralytics’ TensorRT integration docs. citeturn16view7turn37search2turn18view1turn18view3

Benchmark the engine separately from your application:

```bash
/usr/src/tensorrt/bin/trtexec \
  --loadEngine=best.engine \
  --warmUp=200 \
  --duration=20 \
  --useCudaGraph \
  --noDataTransfers
```

`trtexec` is explicitly provided for benchmarking networks and generating serialized engines. citeturn14view5turn16view4

A simple Jetson inference smoke test:

```bash
sudo nvpmodel -m 0
sudo jetson_clocks

python deploy/jetson/infer_camera.py \
  --engine artifacts/export/best.engine \
  --source /dev/video0 \
  --width 1280 \
  --height 720 \
  --imgsz 640 \
  --conf 0.35 \
  --publish-nt
```

Locking power mode before benchmarking is grounded in NVIDIA’s Jetson guidance. citeturn15view1

### Example NetworkTables publisher pseudocode

This sketch keeps the robot contract small and persistent.

```python
from ntcore import NetworkTableInstance
import time
import json

nt = NetworkTableInstance.getDefault()
nt.setServer("10.XX.YY.2")   # roboRIO address
nt.startClient4("jetson-vision")

vision = nt.getTable("/vision/gamepiece")

pub_has = vision.getBooleanTopic("hasTarget").publish()
pub_yaw = vision.getDoubleTopic("yaw_deg").publish()
pub_pitch = vision.getDoubleTopic("pitch_deg").publish()
pub_range = vision.getDoubleTopic("range_m_est").publish()
pub_conf = vision.getDoubleTopic("confidence").publish()
pub_lat = vision.getDoubleTopic("latency_ms").publish()
pub_json = vision.getStringTopic("allDetectionsJson").publish()

while True:
    t0 = time.perf_counter_ns()

    detections = run_inference()   # your TensorRT inference
    primary = select_best_detection(detections)

    latency_ms = (time.perf_counter_ns() - t0) / 1e6

    if primary is None:
        pub_has.set(False)
        pub_lat.set(latency_ms)
        pub_json.set("[]")
        continue

    yaw_deg, pitch_deg = angles_from_box(primary.box, K)      # from camera intrinsics
    range_m = estimate_range(primary.box, K, real_width_m)    # only if you trust this geometry

    pub_has.set(True)
    pub_yaw.set(yaw_deg)
    pub_pitch.set(pitch_deg)
    pub_range.set(range_m)
    pub_conf.set(primary.confidence)
    pub_lat.set(latency_ms)
    pub_json.set(json.dumps([d.to_dict() for d in detections]))
```

NT4 publishers are type-specific and need to remain alive as long as you want them active, which is why they are created once and kept for the process lifetime. citeturn36search0turn36search4turn36search7

### Risks, mitigations, and where teams usually go wrong

If the **dataset is too small**, the fix is rarely “switch model families.” First add more diverse robot-mounted data from the real failure modes. Ultralytics’ own training guidance puts more emphasis on dataset scale, variety, and label quality than on fancy configuration changes. citeturn31view0

If the **labels are bad**, stop training immediately. Tight, consistent boxes and complete labeling matter. Ultralytics explicitly warns that partial labeling and inaccurate boxes degrade training, and CVAT gives you the QA machinery to detect that before you waste GPU time. citeturn31view0turn14view13turn14view14

If the model **overfits one lighting regime**, the response is not “augment harder forever.” First collect more real conditions. PhotonVision and WPILib both show how strongly exposure, brightness, and resolution affect practical vision performance. citeturn22view0turn15view16turn15view17

If **Jetson performance is too slow**, the cheapest sequence of fixes is usually: lower input resolution; reduce camera resolution while preserving detector crop fidelity; move from PyTorch/ONNX Runtime to TensorRT FP16; remove excess copies in the camera pipeline; and only then consider INT8 or a smaller model family. The published Orin Nano timing differences between PyTorch and TensorRT support this order of operations. citeturn19view1turn19view2turn14view5

If **camera exposure is wrong**, no model change will save you. PhotonVision explicitly recommends keeping exposure as low as possible while maintaining trackability and calls out motion blur as a primary robotics failure mode. citeturn22view0turn22view3

If the model is **not useful to controls**, the failure is usually interface design, not ML. Publish the few variables that downstream code can actually act on, define stale/invalid semantics, and align coordinate frames to WPILib’s conventions. A detector that only produces pretty rectangles is not integrated. citeturn21view0turn15view8turn14view16

If students start treating training as magic, force them to answer four questions for every run:
what changed, why it changed, what metric moved, and what failure cases remain. MLflow tracking and a fixed validation-video suite make this teachable. citeturn14view8turn16view13

### Open questions and limitations

This guide is intentionally architecture- and workflow-focused rather than tied to a specific FRC season’s game pieces. The exact class design, camera position, and distance-estimation geometry should be finalized only after the season game object, robot mechanism geometry, and real camera placement are known.

Published benchmark numbers for Jetson backends are useful, but vendor benchmark tables often exclude pre/post-processing and may use specific model variants, software versions, and device modes. You should treat them as directional, not as guaranteed robot-match performance. citeturn19view0turn19view1turn19view2

The most defensible first deployment decision, given the current tooling and hardware, is still: **small YOLO-family detector, disciplined dataset versioning, single-GPU training, FP16 TensorRT on-Jetson, and NetworkTables for first robot integration**. Everything more complicated should justify itself against that baseline.