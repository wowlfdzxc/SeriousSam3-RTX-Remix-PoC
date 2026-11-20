Serious Sam 3: BFE + NVIDIA RTX Remix (Proof of Concept)
Status: Research / Partial Reverse-Engineering (PoC)
Engine: Serious Engine 3.5 (Direct3D 9Ex / IDirect3DDevice9Ex)
Objective: Compatibility Mod for Path Tracing via NVIDIA RTX Remix Bridge
Author: wowlfdzxc
Project Origins: The initial algorithmic approach and Shader Model 2.0 bypass concept were developed with assistance from Google AI Overview, serving as the foundational starting point for further reverse-engineering.

📌 Overview
This repository contains the results of partial reverse-engineering, technical documentation, and the source code of a custom proxy wrapper (d3d9.dll) designed to research the viability of running NVIDIA RTX Remix path tracing in Serious Sam 3: BFE.

Because Serious Engine 3.5 completely strips out the legacy Fixed Function Pipeline (FFP), this project implements a compatibility mod following the remix-comp-base principle—intercepting DirectX 9 shader and pipeline calls on the fly and translating them into a format understandable by the RTX Remix runtime.

⚠️ Important Disclaimer:
I am not a professional programmer. This code is the result of research, written with heavy assistance from Artificial Intelligence (AI) tools. The code may contain errors, lack optimization (e.g., declaration caching), and proper error handling. The goal of this repository is not to provide a finished mod, but to share a starting point and reverse-engineering findings. I invite experienced developers to continue this work by porting the logic into remix-comp-base.

🏗️ 1. Serious Engine 3.5 Render Architecture
Monolithic Renderer: The engine lacks separate graphics DLLs (such as GfxD3D9.dll). The entire rendering pipeline is statically built directly into Sam3.exe, while shaders (.hs / .hsl) are stored in open formats within .gro archives. Additionally, high-level shader source code can be found embedded directly inside Sam3.exe (readable via text editors).

Direct3D 9Ex & DirectInput: The engine leverages the extended IDirect3DDevice9Ex interface (requiring interception of CreateDeviceEx and PresentEx) and handles user input via dinput8.dll.

Hard Shader Model 2.0 Threshold: The engine mandates at least SM 2.0 (gfx_iPixelProgramVersion = 20). Forcing fallback profiles like 0.0 or 1.4 results in an immediate crash with the error: Missing support for vertex/pixel shaders 2.0!.

🔍 2. Confirmed Memory Data (ss3_matrices.log Dumps)
Using a custom frame-dumping mechanism implemented in Present (triggered via the F10 key), the following registers and memory structures were successfully extracted and verified:

2.1. Vertex Shader Constants (SetVertexShaderConstantF)
c00–c03 (World-View-Projection Matrix / WVP): Stored in column-major format. Requires transposition to row-major for proper FFP/Remix handling.

Note on Y-axis (c01 = -1.8107): The negative Y scale in the projection matrix is either a Render-to-Texture (Offscreen HDR/Bloom) vertical flip, or a result of Croteam's internal Right-Handed (RH) coordinate conventions being converted into D3D9's Left-Handed (LH) clip space. Keep this in mind when decomposing matrices (you may need to flip Y back or test LookAtRH vs LookAtLH).

c06 (Forward / View Vector): Normalized camera direction vector (length ≈ 1.0).

c38 (Camera World Position & Render State Switch):

c38.xyz — Exact world coordinates of the player's eyes (e.g., height 1.4850 corresponds to Sam's eye level).

c38.w — Primary Render Switch: 1.0 during 3D world rendering, and 0.0 when rendering flat 2D user interfaces (HUD, fonts, menus). Used as the main filter for Remix.

c41–c95 (Bone Matrices): Arranged in 3x4 blocks driving skeletal animations for weapon models and enemies.

2.2. Croteam Vertex Topology Quirks
Position Masking: Inside CreateVertexDeclaration, vertex position coordinates (Stream 0) are not assigned with Usage = 0 (POSITION). Instead, they are disguised as texture coordinates (Usage = 5 (TEXCOORD0)). Without explicit runtime remapping, RTX Remix completely drops or misinterprets the geometry, logging unsupported primitive topology.

CullMode: The exact winding order (CW/CCW) was not verified in logs. However, Serious Engine historically uses D3DCULL_CCW (Counter-Clockwise). This must be confirmed during integration, as incorrect culling will cause walls to be invisible to rays.

🛠️ 3. Implemented Solutions & PoC Features
Shader Profile Spoofing (GetDeviceCaps):

Forcing reporting of SM 2.0 capabilities (VS 2.0, PS 2.0, MaxVertexShaderConst = 256, NumTemps = 12, PS20Caps.NumInstructionSlots = 96).

Crucial Parameter: PixelShader1xMaxValue = 65535.0f — forces the engine to automatically disable heavy internal HDR, FXAA, and multi-pass post-processing, yielding clean base geometry. Heavy shaders are stripped, leaving simple materials (walls, wires) intact and complex ones (asphalt, weapons) black or white.

Hardware Vertex Processing: Intercepted CreateDevice to force HARDWARE_VERTEXPROCESSING. This eliminated 95% of screen artifacts (flickering, scrambled geometry) caused by software vertex processing in SM 2.0 mode.

Smart Z-Buffer Restoration (Esc-Menu Fix):

Under SM 2.0 profiles, opening the pause menu causes the engine to clear/release the depth stencil surface, rendering the 3D world transparent.

Solution: Caching non-null g_SavedZBuffer pointers inside SetDepthStencilSurface (secured via manual AddRef() / Release() and cleaned up in Reset()). The depth buffer is restored strictly inside SetRenderState when the engine itself activates D3DRS_ZENABLE for 3D scenes—leaving 2D text, menus, and HUD elements completely unaffected.

Object Flickering Elimination (D3DQUERYTYPE_OCCLUSION):

In SM 2.0 mode, the engine drops Depth Prepass cycles while continuing to issue hardware occlusion queries, causing concrete barricades to flicker.

Solution: Returning D3DERR_NOTAVAILABLE in CreateQuery for occlusion queries.

Trade-off: Eliminates flickering (vital for a stable RTX Remix BVH tree) at the cost of a performance drop (from ~280 to ~192 FPS).

Frame Dumper (Present + F10):

An active runtime logging tool outputting verified matrix states and vertex declarations to ss3_matrices.log.

🧪 4. Negative Results (Dead-Ends)
❌ Blocking 16F/32F Formats in CheckDeviceFormat: Attempting to block half/float render target formats to disable internal HDR breaks screen clearing (Clear) and results in black screens or a "Hall of Mirrors" echo effect. CheckDeviceFormat must remain unmolested.

❌ Manual Matrix Injection via Free Constants (c200/c204): Writing custom camera matrices into high constant registers via SetVertexShaderConstantF is entirely ignored by the RTX Remix runtime.

❌ Forcing D3DRS_ZENABLE = TRUE inside BeginScene: Breaks 2D HUD and menu text rendering, causing them to clip behind 3D elements.

❌ ATI X550 XT (RV370) Hardware Pipeline Issues: While the wrapper functions cleanly on modern hardware, legacy 2005 ATI architectures suffer from fixed-function vertex processing driver bugs requiring separate software vertex pipeline adjustments.

🚀 5. RTX Remix Integration Roadmap (Next Steps)
To achieve actual path tracing, the following steps are required (best done within remix-comp-base):

Vertex Declaration Patching (CreateVertexDeclaration):

Safe and robust iteration over the D3DVERTEXELEMENT9 array stopping explicitly at the stream terminator (Stream == 0xFF).

Remapping Stream 0 usage to Usage = 0 (POSITION) while preserving normal and UV streams.

Precise computation of vertex Stride sizes for SetStreamSource.

Caching modified declarations via std::unordered_map to prevent per-frame memory allocation overhead.

Matrix Separation & Valid Camera Detection:

Decomposing the combined WVP matrix (c00–c03) into separate D3DTS_VIEW and D3DTS_PROJECTION matrices, incorporating camera position data from c38.

Note: For static geometry (walls, terrain), World is likely Identity (vertices are in world space). For dynamic objects (weapons, enemies), the World matrix is baked into c00–c03. To extract it, one must multiply the inverse of the global ViewProjection by the object's WVP (World = (VP_global)^-1 * WVP_object). Otherwise, the weapon will appear glued to the camera and cast incorrect shadows.

Fix the not detecting a valid camera warning: Remix requires separate View and Projection, not a combined WVP.

Render State Management:

Managing active shaders inside DrawIndexedPrimitive for 3D meshes (c38.w == 1.0f). Save GetVertexShader/GetPixelShader, set them to nullptr for the draw call, then restore immediately.

Filtering passes: Only send geometry to Remix when the main Render Target is active and COLORWRITEENABLE != 0 (ignore Z-Prepass, Shadow Maps, and HUD).

Architecture Note: For D3D9 titles (including Serious Engine), the RTX Remix Bridge hooks standard D3D9 draw calls and builds scenes automatically. Low-level explicit Remix API calls (remixapi_CreateMesh) are not required for baseline geometry raytracing—they are only necessary for custom injected meshes or specific particle effects.

📦 6. Building
Compile the project as a 32-bit (x86) shared library named d3d9.dll (using Visual Studio or MinGW g++).

Place d3d9.dll into the Serious Sam 3\Bin\ directory directly alongside Sam3.exe (configure the Remix runtime wrapper loader d3d9_remix.dll and .trex asset folder accordingly).
