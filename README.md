# DICOM Point Cloud Importer

A Blender addon for working with medical scans: it reads DICOM datasets and .fre files and turns them into point
clouds, volumes and meshes, as well as separating them into anatomical pieces. 

No additional intallations needed. NumPy is included with Blender, and a copy of PyDicom is bundled with the plugin. PyDicom is bundled under the MIT license; see vendor/pydicom/LICENSE.

Requires Blender 5.2 or newer.

---

## Contents

1. [The Basics](#the-basics)
2. [What a scan actually is](#what-a-scan-actually-is)
3. [Panel 1 — DICOM Point Cloud](#panel-1--dicom-point-cloud)
4. [Panel 2 — Segmentation Tools](#panel-2--segmentation-tools)
5. [Panel 3 — Reconstruction Settings](#panel-3--reconstruction-settings)
6. [Panel 4 — Volume Viewer](#panel-4--volume-viewer)
7. [Quick Guides](#guides)
8. [What common warnings mean](#what-common-warnings-mean)
9. [File formats](#file-formats)
10. [Limits worth knowing](#limits-worth-knowing)
11. [Layout and tests](#layout-and-tests)

---

## The Basics

There are three basic "shapes" scan data can be processed in using this plugin, with each one being useful for a different task:

| Shape | Definition | Used For |

| **Point cloud** | one point per voxel above a threshold | picking within a threshold, seeing what is there, segmenting |
| **Volume** | the measured field itself, rendered | reading the scan, finding where something is, cutting through it |
| **Mesh** | a surface at one chosen value set | printing, sculpting, export, anything that needs real geometry |

The usual workflow is: import as points to find a threshold, or as a volume to view as a whole; segment into pieces or values; then mesh the piece you want.
The scan type affects what data is easiest to extract. CT scans are ideal for processing bones, while MRI scans are best for soft tissues. MRI bone extraction and CT tissue extraction can still be used, but to a lesser degree of accuracy.

CT scans are processed using Hounsfield units, which are fine-tuned for segmenting scan data. 
Air is −1000, water is 0, muscle about +50, bone above +200, and those hold true for all CT scans. 

In MRI scans, brightness depends on the sequence, the coil, and the machine, so more experimentation with settings may be needed to get the best quality data.

---

## What a scan actually is

**Slices and spacing.** A CT or MRI is a stack of 2D images. Each has a pixel
size in millimetres and a position in space. The spacing between slices is
usually *not* the same as the pixel size. For example, having 0.7 mm pixels with 1–5 mm slices is
common. That anisotropy is why **Equalize Voxel Aspect** exists.

**Hounsfield units (CT).** A calibrated density scale:

| Tissue | HU |

| Air | −1000 |
| Lung | −700 to −600 |
| Fat | −100 |
| Water | 0 |
| Muscle | +40 to +60 |
| Cancellous bone | +300 to +400 |
| Cortical bone | +700 and up |

Because these are fixed, a CT can be segmented by *thresholding against
absolute numbers*, which is what CT Scan mode does.

**Windowing.** A scan holds far more range than a screen can show. A window is a low and a high value; everything below is black, above is white, and the range between is spread across the greys. Radiology uses standard windows (bone, soft tissue, lung) and those are the ones in the Volume Viewer.

**Dixon MRI.** A technique that acquires fat and water separately, giving four images of the same anatomy: **fat**, **water**, **in-phase** and **out-of-phase**. 
The relations are arithmetic:
in  = water + fat
out = water − fat

That matters in two ways. The **fat fraction** or "fatfrac" `fat / (fat + water)` tells
muscle from fat, which is how muscles are identified. And the out-of-phase image has a dark line wherever fat and water meet in one voxel, which distinguishes the fascia between muscles. It is the boundary the segmenting tool cuts on.

The forward direction holds in real data (`water+fat` predicts in-phase at r=0.996), so a station missing its out-of-phase series can have it computed. The reverse does not: these are magnitude images and `|water − fat|` has
thrown the sign away, so fat and water cannot be recovered from in and out.

---

## Panel 1 — DICOM Point Cloud

Reading a study in, and turning one series into points.

### Finding the data

| Control | What it does |
| **Source Path** | A folder of .dcm files, a `.zip` of scans, or a folder of `.fre` images. |
| **Source Is Zip** | Tick for an archive. |
| **Scan Series** | Reads the headers and lists every series it finds. This must be done before attempting any file processing. |

A study is usually many series — different contrasts, different stations,
and localizers. After scanning, the **Series** list shows each with a thumbnail of
its middle slice, making determining what is needed at any given time significantly easier.

### Choosing what becomes a point

| Control | What it does |
| **Threshold** | Voxels dimmer than this are discarded. |
| **Max Threshold** | Voxels brighter than this are discarded too (0 = no limit). |
| **Stride** | Take every Nth voxel. A 2 means an eighth as many points. |
| **Max Points** | A ceiling; the stride widens on its own to meet it. |
| **Safe Mode** / **Safe Cap** | A hard limit on point count to reduce processing time and prevent crashes. |
| **Point Radius** | How big each point draws. Does not change the data. |

Threshold is in normalized units (0–1 across the series' range), not HU. Use **Max Threshold** to isolate a band, like soft tissue without bone, for instance.

### Geometry and correction

**Use Scanner Position** places each cloud where the scanner recorded it. Keep this on unless you are only working with a single series. With it enabled, the stations of a whole-body study assemble correctly and the contrasts of a Dixon acquisition land exactly on each other. Without it, every cloud is centred on the origin.

**Correct Coil Shading** flattens the brightness gradient an MR receive coil produces. Tissue near the coil often reads brighter than the same tissue further away, so enable this if scans are unevenly lit. **Shading Scale (mm)** is how broad of a gradient is present to treat as shading rather than anatomy.

### Importing

**Import Point Cloud** builds one cloud from the selected series.

**Import By Type** builds one cloud per station, all of the chosen contrast. This is the quick way to get a whole-body Dixon water series in as separate pieces with a single click.

**Type** picks which contrast to import: water, fat, in-phase, out-of-phase, or everything.

---

## Panel 2 — Segmentation Tools

All the tools that divide a scan into separate pieces.

### Import Segmented Features

The main segmentation tool: it finds individual structures and imports each as its own
point cloud. It works two different ways depending on the modality.

**On Dixon MRI** it uses the dark fascial line in the out-of-phase image as
the boundary between muscles, and the fat fraction to tell muscle from fat.

| Control | What it does |
| **Boundary Strength** | How dark a line has to be to count as a boundary. A higher number cuts more aggressively. |
| **Fat Plane Cut** | How fatty a voxel must be to count as a fat plane separating muscles. Lowering the value picks up thinner layers, while raising it will result in larger clumps. |
| **Slice Linking** | How much overlap two regions need across slices to be called the same structure. Raise this if neighboring features are being counted as one, and lower it to split more aggressively between layers. |
| **Max Fat Fraction** | Above this, a voxel is considered fat rather than muscle. Raise it to keep areas of fattier tissue. |
| **Min Muscle (cm3)** | Discard pieces smaller than this in cubic centimeters. |
| **Thinnest Layer (slices)** | A structure thinner than this is treated as a leak, not anatomy. Raise this if "sheet" artifacts are appearing in segments. |

The **Thinnest Layer** setting is worth understanding. A boundary that is grazed in one slice lets two structures merge through a one-slice bridge, which shows up as a flat "shelf" sticking out of a result. Raising this detaches layers thinner than the limit and reattaches them to whichever neighbour they actually touch. Keep this setting at 0 if it is not needed.

**Work From Single Source** runs on selected series when a study has no companion contrasts. There is no fascial line in a single-source MRI scan, so boundaries come from where the image changes fastest. 

**Finding bone in an MRI.** Tick **Find Bone (MRI)**. This needs a Dixon station with both a fat and a water series. **Bone From** chooses between two readings of "bone", and they have weaknesses in opposite directions.

**Dark Interior** (the default) is a definition rather than an inference: every voxel with no signal in *either* image that lies inside the body. Cortical bone returns nothing at all, and it works wherever there is a bone, even where the other reading finds nothing. The cost is that tendon, ligament, vessel and bowel gas are signal-void too and come along with it. Each piece should arrive separately so the unwanted ones can be deleted one at a time.

What makes it possible is that subcutaneous fat is bright in the fat image, so it forms a closed bright ring between the dark inside and the dark air outside. This allows the inside to be determined from the body silhouette.

| Control | What it does |
| **Dark Level** | How dark a voxel must be, in both images, to count as having no signal. Defaults to 0.18. Raise it to take in more bone, lower it if muscle boundaries appear or two bones come out as one piece. Past about 0.30 everything fuses. |
| **Min Bone (cm3)** | Discard pieces smaller than this. |

**Marrow And Cortex**, the other reading, works outward from fatty marrow instead. It is cleaner, but it can only find bones that have fatty marrow and keep a recognisable shape, which in practice means the shafts of long bones like the femur. This method is ideal for arm and leg scans only.

Three things have to be true of a region before it is called bone, and each
one is there because measurement showed the others were not enough.

- **It is fat-bright.** Marrow is fat. On its own this says very little, because subcutaneous and visceral fat read exactly the same.
- **Its rim has no signal.** Cortex is dark in the water image and dark in the fat image both, and nothing else that bounds fat is. The rim is judged only where it lies *inside* the body, because air is signal-void too and would otherwise let any subcutaneous sheet pass.
- **It is longer than it is wide, or else plainly walled in.** On a coarse or noisy series the fat threshold shatters into dozens of small blobs that happen to sit beside something dark.Raising **Cortex Rim** tightens this route too; it can never be looser than the value you set.

| Control | What it does |
| **Marrow Brightness** | How bright the fat image must be for a voxel to count as marrow. Lower it if bones come out hollow or broken. |
| **Cortex Rim** | How much of a candidate's rim must have no signal for it to be bone. Lower it if bones are being missed, raise it if other fat is picked up. |
| **Min Bone (cm3)** | Discard marrow smaller than this. A femur section runs to tens of cm3; the noise is fractions of one. |
| **Cortex Depth** | How far to follow the signal void outward from the marrow to take in the cortex. |

**Cortex Depth, and why the bone looks smaller than it is.** Only the marrow can be seen in an MRI scan. Cortical bone returns no signal at all, so it is never fat-bright and can only be claimed outward from the marrow it surrounds, which is what this control does. It defaults to 8. Measured on a femur, 3 voxels roughly doubles the claimed volume, 6 gives about 2.8x and 10 about 3.5x. Raise it to take more of the bone; if muscle boundaries start being claimed, lower it again.

In both modes, the bones come out as their own clouds with `Bone` as a prefix after the segmentation tool is used.

**For a CT series** tick **CT Scan**. It reads Hounsfield units directly and splits into
bone, muscle and fat by calibrated thresholds rather than percentiles. Bone is
then separated object by object using the dense cortical shell as seeds, which
is what divides touching tarsals and phalanges.

| Control | What it does |
| **Cortex Level (HU)** | How dense a voxel must be to be cortical shell. Lower this if bones are coming out merged. |
| **Min Bone Seed (cm3)** | The smallest seed that can start a separate bone. Lower this to pick up on smaller bones. |
| **Split Small Bones** | How aggressively to divide bones that are touching. A lower value splits more aggressively. |

### Separate By Intensity

Divides one already-imported cloud into brightness bands. **Ranges** is how many. A blunt tool, but quick and useful when a scan has no clean boundary to cut on. Works for both MRI and CT series.

### Split Selected Cloud

Takes one cloud that came out fused and breaks it into parts, using only that cloud's own data. Results are fast, but not always accurate.

### Create Filtered Point Cloud

Removes a percentage of the darkest points from an existing cloud. Good for trimming
noise or peeling off a bright rind.

| Control | What it does |
| **Remove Percent** | How dark the points removed should be based on the percentage of the cloud they take up. |
| **Invert (Remove Lightest)** | Drop the brightest points instead of the darkest. |
| **Output Suffix** | Appended to the new cloud's name. |

---

## Panel 3 — Reconstruction Settings

Resolution and detail, and everything that rebuilds geometry. These resolution settings apply to every point cloud tool that resamples, including importing, segmenting, upsampling and meshing.

### Resolution

| Control | What it does |
| **Resolution Multiplier** | How much finer than the original scan's density to resample up to. |
| **Voxel Budget (M)** | A memory ceiling, in millions of voxels. Only raise this if the desired resolution requires it. |
| **Max Vertices (M)** | Refuse to build a mesh larger than this. |
| **Equalize Voxel Aspect** | Resample to cubic voxels, ideal for MRI scans. |
| **Suppress Ringing** | Clamp interpolation overshoot, which otherwise makes halos at sharp edges. This helps reduce stair-stepping artifacts in low resolution scans. |
| **Filter** | The interpolation kernel. Cubic works in most situations, but change to Linear or Lanczos if the result seems overly boxy or glitched. |

**Enhance Resolution** turns these settings on for any action that involves resampling. It interpolates the measured intensity field before anything is thresholded to give more accurate results. On Dixon data this matters more for segmenting than for importing, because the fascia between muscles is thinner than a slice, so whether it survives resampling decides what can be separated at all.

### Increase Point Cloud Resolution

Adds points between the existing ones in the selected point cloud(s). **Upsample Mode** chooses how.

### Keep Only Outside Points

Drops points surrounded on all sides, leaving a shell. **Shell Thickness** is how deep a rind to keep. The result is much faster to mesh.

### Surface Mesh Detail

| Control | What it does |
| **Surface Smoothing** | Blur applied before the surface is extracted. Above 1.0 rounds detail away. |
| **Fill Holes** | Bridges gaps up to roughly twice this many voxels wide. High values may weld separate structures together. |
| **Cover Every Point** / **Cover Radius** | Guarantee every point ends up inside the surface |
| **Min Blob Voxels** | Discard specks smaller than this. Lower this if detail seems to be getting lost. |
| **Min Island Volume** | Discard disconnected pieces smaller than this. Lower this if detail seems to be getting lost. |
| **Mesh Smoothing** | Smoothing level applied to the finished mesh. More detail-preserving than **Surface Smoothing**. |
| **Keep Internal Cavities** / **Min Cavity Volume** | Whether hollow interiors survive, and how small they must be to get ignored. |

**Create Surface Mesh From Selection** builds a watertight surface for each selected cloud by dual contouring, with consistent normals. **Hull Prefix** names the results and **Hide Source Clouds** hides the clouds they came from, so the new meshes are not buried inside their own points.

### Quick Remesh

**Add Voxel Remesh** attaches a correctly scaled Remesh modifier. One unit is one millimetre here, so an object is hundreds of units across and Blender's own 0.1 default voxel size would ask for a grid thousands of cells wide and run out of memory. This scales it to the object. **Remesh Detail** sets how fine that grid is, as a fraction of the object rather than in absolute units. *Never attempt to use Blender's default remesh modifier on a mesh made from a point cloud*. It will almost always crash. Remeshing will cost some detail, regardless of scale.

---

## Panel 4 — Volume Viewer

The scan itself, rendered rather than sampled into points. Blender cannot read DICOM directly, so a scan shown this way is first written out as an OpenVDB file in a cache folder. Your DICOM folders are never copied or written to, and everything outside the chosen window is dropped. Since VDB stores only what differs from its background, usually makes the result smaller than the scan it came from, making viewing volumes possible in real-time.

### Window

Which range of the scan to show.

- **Auto**, **Middle Of The Data**, **Brightest Only** are percentile-based and will work on any scan.
- **Bone**, **Soft Tissue**, **Lung**, and **Everything** are Hounsfield windows that only mean something on CT and will not affect an MRI dataset.

**Half Precision** halves the file size to further speed up processing time.
**Replace Previous** reimports the volume already in the scene for that series rather than stacking another one in the same place.

### Cut

A slice plane through the volume.

| Control | What it does |
| **Cut Through** | Turn the cut feature on. |
| **Cut Axis** | Determines which way to cut. **Auto** faces the viewport. |
| **Cut Depth** | Where along the axis of the chosen direction to make the cut. |
| **Flip Cut** | Keep the opposite half of the volume. |

**Auto** has no meaning in a render, since a render has no single view to face. An Auto cut shows in the viewport only and any render comes out whole. Pick X, Y or Z to render a cut.

Which view you get follows the viewport shading used:

Solid Shading only shows the cut selected with the depth slider. Good for analysing single images in a series.
Material Preview or Rendered view shows the volume itself, through the transfer function, along with any cuts applied to it.

### Shape

| Control | What it does |
| **Surface Level** | Where in the threshold or window the visible surface sits. |
| **Edge Hardness** | How abruptly it goes from clear to solid. At zero it resembles a soft fog. Turned up, it's closer to a surface |
| **Density** | How solid the volume looks when rendered. |

The shape tools apply as the sliders are dragged. Changes in windows need to be reimported to take effect.

### Quality

**Full Volume Resolution** is on by default and matters a great deal. Blender typically draws volumes at an eighth of screen resolution by default, but full resolution is best for volume clarity. Apply this to a selected volume if it seems too hazy or cloud-like.

### Mesh View

**Volume To Mesh** builds real geometry at the selected Surface Level, as a live modifier at the grid's own resolution. The cut and the level keep driving it, so the surface follows the same sliders the volume does. The mesh cut is done with Geometry Nodes rather than a boolean. The surface is left open at the cut rather than capped to drastically improve performance.

**Color From The Scan** colors the surface by what the scan reads just beneath it, so dense bone and soft tissue are told apart instead of everything being one white. **Color Depth (mm)** is how far beneath to read.

It samples beneath rather than at the surface because the surface is the level set where the value equals the level. It samples at three depths and takes the strongest.

The color is stored as a greyscale vertex color (`Col`) and the raw value as
a float attribute (`DICOM Tissue`), so you can drive your own material from
it. To see it: **Material Preview**, or Solid mode with Color set to
**Attribute**.

---

## Guides

### Bone from a CT

1. Scan Series, pick the CT series.
2. Segmentation Tools → tick **CT Scan** → **Import Segmented Features**.
3. You get bone, muscle and fat as separate clouds, with bones already divided
   where the cortical shell separates them.
4. Any bone that came out fused: select it, set **Pieces**, **Split Selected
   Cloud**.
5. Select the bones you want → Reconstruction Settings → **Create Surface Mesh
   From Selection**.

### Bone from an MRI

1. Scan Series, pick the water series of the station you want.
2. Segmentation Tools → tick **Find Bone (MRI)**, leave **CT Scan** off.
3. **Import Segmented Features**. Bones arrive as clouds named `Bone`
   alongside the muscles.
4. Leave the dropdown on **Dark Interior**. Each piece arrives as its own
   cloud, largest first, so delete the ones that are tendon or gas rather
   than bone.
5. If two bones have come out as one piece, lower **Dark Level**; if a bone
   is thinner than it should be, raise it. Past about 0.30 everything fuses.
6. For a clean long bone with no tendons attached, switch to **Marrow
   And Cortex**. Lower **Marrow Brightness** if bones come out hollow.
   Lower **Cortex Rim** if nothing is found, and raise **Cortex Depth** if
   the bone looks too thin. Result quality may vary.

### Individual muscles from a Dixon MRI

1. Scan Series. The study needs fat, water and out-of-phase at one station —
   fat and water alone also works, out-of-phase is computed if needed.
2. Segmentation Tools, leave **CT Scan** off.
3. Start with **Max Fat Fraction** around 0.35 and **Min Muscle** around
   5 cm³, then **Import Segmented Features**.
4. If results are merging, raise **Boundary Strength**. If they are
   fragmenting, lower it. If flat shelves appear, raise **Thinnest Layer**.

### Reading a scan

1. Volume Viewer → **Import As Volume**.
2. Leave the viewport in Solid shading and drag **Cut Depth**. This is the closest thing in this plugin to a radiology viewer.
3. Switch to Rendered shading to see the volume, and use **Surface Level** and **Edge Hardness** to pick out what you want.

### A mesh to print

1. Get the structure as a cloud, by whatever route.
2. **Keep Only Surface Points** is recomended as it is much faster but results in the same surface.
3. **Create Surface Mesh From Selection**. It is watertight with consistent normals already.
4. Only remesh if you need even topology; it will cost detail.
5. Alternately: Convert a volume into a mesh, remesh the selection, then remove unwanted loose parts.

---

## What common warnings mean

A series that cannot support a 3D reconstruction produces a point cloud that looks perfectly fine and then fails to mesh for reasons that look like settings. When the files themselves are the problem, the addon says so in a popup before you spend time tuning.

| Warning | What it means |
| **localizer / scout** | A planning image, not a volume. Selected series is three orthogonal planes and cannot support a reconstruction however it is tuned |
| **slices are missing** | The positions span more slices than the file count. Everything past a gap sits too close to the start. |
| **slices share a position** | Duplicates, stacked as if separate. |
| **all slices report the same position** | Not a stack at all |
| **Nx anisotropic** | Slices far apart relative to the pixel size; through-plane detail cannot be recovered |
| **each slice averages over N units** | Thick slices; structures thinner than that are blended with their surroundings before you ever see them |
| **a Hounsfield window on a non-CT** | Bone/Soft/Lung do not apply; the middle of the data was used instead |

---

## File formats

**DICOM**, as a folder or a `.zip`. Slices are ordered along the axis the
stack actually advances along, not by file name and not by the z coordinate —
a sagittal series advances along x, and sorting it by z leaves it in whatever
order the files were read in. Spacing is the median gap, not the mean, so one
missing slice does not stretch the whole volume.

Files that are not DICOM are skipped by a format check before any parsing.
That check exists because reading headerless DICOM needs `force=True`, which
tells the parser to trust whatever sits where the length fields belong — a
2.5 KB Python file in a scan folder was measured asking for 1.8 GB, so a
folder of a few thousand files in another format did not fail, it took the
machine down.

**GE `.fre`**, as the Visible Human Project ships them. Point a folder of them
at Scan Series and each run appears as its own series. This is worth doing
rather than converting to DICOM first, because this dataset loses its geometry
in three ways that a conversion has to get right:

- The field of view changes partway down the scan. On the female CT, series
  370 is reconstructed at 250 mm over the head, then 370, then 440, then
  480 mm over the body. One pixel size applied throughout stretches the head
  by 48% and squashes the body by 23%.
- One folder holds more than one series — 370 with 985 slices and 371 with
  749, covering overlapping anatomy. Stacked together, 749 pairs of slices sit
  in the same place.
- File names run straight through both series, so sorting by name walks down
  one and then jumps 940 mm back up to start the other.

So position, pixel size and series are read per slice from each header, and
the slices are resampled onto one grid — the pixel size most of them already
use, so the majority is untouched. The import reports how many it rescaled.

---

## Limits worth knowing

- **Bone on an MRI is inferred, not measured.** Cortical bone returns no
  signal at all, so there is no equivalent of "above +300 HU". Dark Interior
  takes everything signal-void inside the body, which is bone *and* tendon,
  ligament, vessel and bowel gas - it over-claims by design, and sorting the
  pieces is left to you. Marrow And Cortex is cleaner but only finds bones
  with fatty marrow. Neither is a measurement of bone.
- **Anisotropy is not recoverable.** If the slices are 5 mm apart, nothing
  here invents detail between them. Enhance Resolution interpolates, which is
  honest about being interpolation.
- **Meshing is not segmentation.** Create Surface Mesh builds the hull of the
  cloud it is given, as it is. Deciding *what* to mesh happens before, in the
  Segmentation Tools.
- **The volume cut is open, not capped.** Capping costs about ten times as
  much per update.
- **An Auto cut does not appear in renders.** Pick an explicit axis.
- **Window changes need a reconversion**; Shape and Cut do not.
- **Half-precision volumes** are for looking at, not for reading values back
  out for measurement.

---

## Layout and tests

| File | What is in it |
| --- | --- |
| `io_dicom.py` | Reading studies, series discovery, stack order and geometry checks |
| `fre.py` | GE genesis `.fre` images: headers, per-slice geometry, stacking |
| `pointcloud.py` | Volume to points, and the lattice bookkeeping that lets a cloud be mapped back to voxels |
| `volume.py` | DICOM to OpenVDB: windowing, the grid transform, the cache key |
| `muscle.py` | Segmentation: Dixon channels, HU classes, splitting touching objects |
| `surface.py`, `isosurface.py`, `resample.py`, `voxelgrid.py` | Resampling and surface reconstruction |
| `preprocessing.py` | Coil shading correction |
| `operators.py`, `panel.py` | Blender operators and UI |
| `segmentation.py` | The older intensity region-growing split |
| `nifti.py` | Minimal NIfTI read/write. Tested, but not currently reachable from the UI |

Ten suites run outside Blender against a stub `bpy` and synthetic volumes with
known ground truth:

```bash
for t in tests/test_*.py; do python "$t"; done
```

Two more need a real Blender, because OpenVDB and the UI only exist inside it:

```bash
blender -b --factory-startup --python tests/test_volume_live.py
blender -b --factory-startup --python tests/test_panels_live.py
```

## Install

Zip the `dicom_pointcloud_importer` folder, or copy it into your Blender
addons path, then `Edit > Preferences > Add-ons > Install...` and enable
**DICOM Point Cloud Importer**. No pip step.
