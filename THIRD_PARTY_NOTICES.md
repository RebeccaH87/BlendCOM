# Third-party notices

BlendCOM is licensed under GPL-2.0-or-later (see [LICENSE](LICENSE)). It
redistributes and depends on the work listed here. The notices in part 1 are
required to travel with the addon: they cover code that is physically inside
the `.zip` you install.

---

## 1. Redistributed inside the addon

### pydicom 3.0.2 — MIT License

Copyright (c) 2008-2020 Darcy Mason and pydicom contributors

- Homepage: https://github.com/pydicom/pydicom
- Shipped at: `blendcom/vendor/pydicom/`
- Full licence text: `blendcom/vendor/pydicom-3.0.2.dist-info/licenses/LICENSE`

Vendored so that reading DICOM needs no `pip install` into Blender's Python.

> Permission is hereby granted, free of charge, to any person obtaining a copy
> of this software and associated documentation files (the "Software"), to deal
> in the Software without restriction, including without limitation the rights
> to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
> copies of the Software, and to permit persons to whom the Software is
> furnished to do so, subject to the following conditions:
>
> The above copyright notice and this permission notice shall be included in
> all copies or substantial portions of the Software.
>
> THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
> IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
> FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
> AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
> LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
> OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
> THE SOFTWARE.

### GDCM (Grassroots DICOM) — BSD 3-Clause

Copyright (c) 2006-2016 Mathieu Malaterre
Copyright (c) 1993-2005 CREATIS
(CREATIS = Centre de Recherche et d'Applications en Traitement de l'Image)
All rights reserved.

pydicom's private tag dictionary was generated from GDCM's, so **that portion
is BSD 3-Clause, not MIT**. The terms are reproduced in pydicom's own licence
file at the path above. Note the third clause in particular:

> Redistribution and use in source and binary forms, with or without
> modification, are permitted provided that the following conditions are met:
>
> * Redistributions of source code must retain the above copyright notice,
>   this list of conditions and the following disclaimer.
>
> * Redistributions in binary form must reproduce the above copyright notice,
>   this list of conditions and the following disclaimer in the documentation
>   and/or other materials provided with the distribution.
>
> * Neither name of Mathieu Malaterre, or CREATIS, nor the names of any
>   contributors (CNRS, INSERM, UCB, Universite Lyon I), may be used to
>   endorse or promote products derived from this software without specific
>   prior written permission.
>
> THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS ``AS IS''
> AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
> IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
> ARE DISCLAIMED. IN NO EVENT SHALL THE AUTHORS OR CONTRIBUTORS BE LIABLE FOR
> ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
> DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
> SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
> CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
> OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
> OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

**This means BlendCOM must not be promoted using the names Mathieu Malaterre,
CREATIS, CNRS, INSERM, UCB or Universite Lyon I.**

### DICOM well-known colour palettes

`blendcom/vendor/pydicom/data/palettes/*.dcm`, derived from the DICOM standard,
PS3.6 Annex B (NEMA). Data from a published standard rather than software.

---

## 2. Required at run time, not redistributed

These ship with Blender; BlendCOM does not include copies of them.

| Project | Licence | Used for |
| --- | --- | --- |
| Blender and its `bpy` API | GPL-2.0-or-later | The host application and all UI, operators and preview handling |
| NumPy | BSD 3-Clause | Every array operation in the addon |
| OpenVDB | Apache-2.0 | The grids behind the Volume Viewer |

Because the addon imports `bpy`, the Blender Foundation's position is that it
must be distributed on GPL-compatible terms. MIT, BSD 3-Clause and Apache-2.0
are all compatible with GPL-2.0-or-later, so nothing above conflicts with the
licence BlendCOM is released under.

---

## 3. Standards and published methods

Specifications and algorithms the addon implements. No code was taken from
these; they are credited because the implementations follow them.

| Source | Where |
| --- | --- |
| DICOM standard, NEMA PS3 | `io_dicom.py` — headers, series discovery, stack geometry |
| NIfTI-1 specification | `nifti.py`, written against the spec rather than depending on nibabel |
| GE "genesis" `.fre` image format | `fre.py` — `IMGF` magic, 3416-byte header, per-slice geometry |
| Visible Human Project, U.S. National Library of Medicine | The `.fre` data the reader was built and tested against. The NLM sets its own terms for that dataset; they are not covered by this project's licence |
| Unix `compress` / LZW | `.Z` decompression |
| Gibson, S.F.F. (1998), "Constrained Elastic Surface Nets" | `isosurface.py` |

---

## 4. Scope of this notice

Beyond the vendored pydicom in part 1, the addon's own modules do not bundle
third-party code. The volumetric primitives in `voxelgrid.py` — connected
component labelling, binary morphology, watershed, flood fill — are standard,
unpatented techniques implemented directly against numpy, because Blender ships
numpy but not scipy.

If you add a dependency, vendor a library, or paste in code from elsewhere, add
it to part 1 or part 2 here. A notice that is not updated is worse than none,
because it reads as a claim that nothing else is present.
