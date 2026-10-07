# ue-rust: walk a shipped UE5 level in Bevy (milestone 1)

Date: 2026-10-07. Status: approved in brainstorming, written up for review.

## What this is

A Rust framework for porting shipped Unreal Engine games to Rust + Bevy.
You point it at a game's packaged files, and Bevy can read and use what is
inside: meshes, textures, materials, levels, and later animation, UI, AI and
Blueprint logic.

A shipped game has no C++ source. Its C++ logic is compiled into the `.exe`
and cannot be turned back into readable code. So the framework brings over
assets and data, decompiles Blueprint bytecode later on, and gives porters
Unreal-like building blocks to make the hand-written rewrite easier.

## Goal of milestone 1

Point the viewer at a shipped UE5 game's files, pick a map, and walk it in
Bevy. Meshes sit in the right place with recognisable textures, lights are
on, and collision works.

**Done when:** one map from our own test project, one Lyra map and one map
from a real shipped game each load in the viewer. In each, meshes, positions,
textures, lights and collision are right, and you can fly and walk around.

## Decisions

| Decision | Choice | Why |
|---|---|---|
| Source files | Shipped games: `.pak` and IoStore (`.utoc`/`.ucas`) | Owner's target. No editor files needed. |
| Engine version | UE5 first; UE4 later | Owner's choice. |
| Language | Rust only. Rust crates are fine; we add no C, C++ or C# libraries | Owner's rule. One exception is opt-in: see Oodle. |
| Other tools | Study CUE4Parse (Apache-2.0), FModel (GPL-3.0, read only), UEViewer, stove (no licence, read only) | Learn from them; port only from permissive licences, with credit. |
| Loading | Live: Bevy reads the game files at runtime | Owner chose this over a separate convert step. |
| Look | "Recognisable": Unreal materials map onto Bevy's `StandardMaterial` | Shipped games keep no material graphs, only compiled shaders. |
| Oodle | One decompress trait. Default is a pure-Rust decoder; loading Epic's own library at runtime is an opt-in feature | Pure Rust by default; the official library stays available for those who want it. |
| Physics | Rapier (`bevy_rapier3d` 0.36, for Bevy 0.19) | Pure Rust, has a character controller. |
| Engine | Bevy 0.19.1, Rust 1.98, edition 2024 | Current stable. |
| Debug UI | `bevy_ui` | Matches the owner's other Bevy projects. |
| Licence | Apache-2.0, with a `NOTICE` crediting CUE4Parse | Lets us port CUE4Parse logic legally. |

## Not in milestone 1

Skeletal meshes, animation, sound, UI, AI, Blueprint logic, full Nanite
decoding, World Partition streaming, texture streaming, painted landscape
layers, exact material looks, UE4 games, console and mobile builds, and a disk
cache for converted assets. These are on the roadmap below.

## Legal ground rules

- The framework never ships or commits any game's content. The user supplies
  game files, decryption keys and `.usmap` files.
- Epic sample content (such as Lyra) is licensed for use inside Unreal only.
  It is never committed; tests against it run on the owner's machine only.
- Code is ported only from permissively licensed projects, with credit in
  `NOTICE`. GPL and unlicensed projects are read for understanding only.

## How it fits together

One Bevy plugin, `UnrealPlugin`. Add it, give it a game profile, load a map:

```rust
App::new()
    .add_plugins((DefaultPlugins, UnrealPlugin::from_profile("games/lyra.toml")))
    .add_systems(Startup, |mut commands: Commands, assets: Res<AssetServer>| {
        commands.spawn(UnrealWorldRoot(assets.load("ue://Game/Maps/Arena.umap")));
    })
    .run();
```

Data flow: game profile, then containers mounted, then `ue://` path loaded,
then package read into generic objects, then translators build Bevy assets,
then the level spawner builds entities.

### 1. Game profile

A `game.toml` per game: game folder, engine version, AES keys, `.usmap`
path and Oodle choice. Loading fails early with a clear message if a file is
missing or a key is wrong.

### 2. File source (`ue://`)

A Bevy asset source that mounts every `.pak` (via `repak`) and every
`.utoc`/`.ucas` (via `retoc`) in the game folder. To Bevy the whole game is
one folder: `ue://Game/Maps/Arena.umap` is Unreal's `/Game/Maps/Arena`.
Later containers override earlier ones by Unreal's patch-order rule.

### 3. Package reader

One parser for the classic package layout. IoStore ("Zen") packages are
converted to the classic layout by `retoc` first.

Every object becomes the same generic shape:

```rust
pub struct UnrealObject {
    pub class: Name,
    pub name: Name,
    pub properties: Vec<(Name, Value)>,
    pub native: Bytes,
}
```

- `Value` covers every Unreal property type: bool, integers, floats, name,
  string, text, enum, object reference, soft reference, struct, array, map,
  set.
- `native` holds the class-specific binary after the properties (mesh
  vertices, texture pixels). Translators decode it, not the reader.
- References to other objects are `ObjectRef { package, object }`.
  Translators turn them into Bevy `Handle`s, so Bevy loads dependencies.
- Unversioned properties (no names stored) are decoded with the `.usmap`.
  Versioned properties need no `.usmap`.
- Bulk data (`.ubulk`, IoStore chunks, inline) is read only when a
  translator asks for it.

### 4. Translators

One Bevy asset loader per Unreal class, chosen through a registry. A class
with no translator still loads as a generic `UnrealObject` asset.

**Coordinates.** Unreal is Z-up, left-handed, centimetres. Bevy is Y-up,
right-handed, metres. One module, `coords`, converts positions, rotations and
scales at the translator boundary: Bevy `x` = Unreal `y`, Bevy `y` = Unreal
`z`, Bevy `z` = minus Unreal `x`, all times 0.01. Nothing past a translator
sees Unreal space.

**Texture2D to `Image`.** BC1/3/4/5/7 go to the GPU as they are. Other
formats are decoded in Rust (`texture2ddecoder`). The sRGB flag comes from
the texture. All mips load at once.

**StaticMesh to `Mesh`.** LOD0: positions, normals, tangents, UVs, vertex
colours, indices. One sub-mesh per material section. Nanite meshes use the
fallback mesh Unreal stores beside them.

**Material and MaterialInstance to `StandardMaterial`.** Walk the parent
chain, collecting texture, scalar and colour parameters. Match names:
BaseColor/Albedo/Diffuse to base colour, Normal to normal map (flip green:
Unreal uses DirectX convention), ORM/Roughness/Metallic to the
metallic-roughness and occlusion slots (Unreal's ORM packing matches Bevy's),
Emissive to emissive. Masked becomes alpha mask, translucent becomes alpha
blend, two-sided turns off culling. No match gives plain grey and a log line
naming the parameters, so matching can improve.

### 5. Level spawner

A `.umap` holds a `World`, its persistent level, and that level's actors.
Components are separate objects that point to their attach parent; that link
becomes a Bevy parent/child link.

- **StaticMeshComponent:** entity with mesh, material and transform. Covers
  Blueprint actors too, because cooked levels store their components as
  plain objects.
- **InstancedStaticMeshComponent / HISM:** one shared mesh, many transforms.
- **Lights:** point, spot, directional and sky light become their Bevy
  equivalents, with unit conversion. Rect lights become spot lights for now.
- **PlayerStart:** where the camera starts.
- **Anything else:** entity with transform and a marker holding its Unreal
  class, ready for future translators.
- **World Partition:** every cell package loads at once. No streaming yet.
- **Landscape:** heights are rebuilt into a mesh from the heightmap textures;
  one flat material for now.
- **Collision:** simple shapes on each mesh (box, sphere, capsule, convex)
  become Rapier colliders. Meshes with none, or set to "complex as simple",
  get a triangle-mesh collider from the visible mesh.

### 6. Viewer app

Map picker listing every `.umap`; fly mode; walk mode (capsule with gravity,
Rapier character controller); debug toggles to draw collision, list objects
that failed to load, and inspect any entity's original Unreal object tree.

## When things go wrong

- An unknown value type is kept as raw bytes with a warning; the rest of the
  object still loads.
- An object that fails to translate becomes a pink placeholder box at its
  position, and goes on a failure list the viewer shows.
- One broken file never stops the rest of the map from loading.
- Profile mistakes (missing folder, bad AES key, missing `.usmap` when the
  game needs one) fail at start-up with a message naming the fix.

## Testing

- **Own tiny UE5 test project** (cubes, textured planes, lights, a small
  landscape, one World Partition map), made by us and safe to commit. It is
  packaged several ways: `.pak` and IoStore, Oodle and Zlib, versioned and
  unversioned.
- **Answer keys:** FModel, run once as an outside tool, dumps expected values
  (vertex counts, bounds, property values) to JSON beside each fixture.
- **Reader tests** parse fixtures and compare with the answer keys.
- **Headless Bevy tests** load a map without a window or GPU and check
  counts and positions of meshes, lights and colliders.
- **Lyra and the real game** run as tests on the owner's machine only,
  skipped unless `UE_RUST_LYRA` / `UE_RUST_GAME` point at a game profile.
- **Visual check** (manual in milestone 1): viewer screenshot beside an
  Unreal editor screenshot.

## Layout

```
Cargo.toml                 workspace
crates/bevy_unreal/        library: profile, source, package, translate/{texture,mesh,material,world,landscape,collision}, coords
crates/unreal_viewer/      app: map picker, fly/walk, debug panels
fixtures/                  own test project, packaged variants, answer keys
docs/                      specs and plans
```

## Roadmap after milestone 1

Each later milestone gets its own design before work starts.

- **M2 Characters:** skeletal meshes, skeletons, animation sequences, our own
  Rust decoder for ACL animation compression.
- **M3 Detail:** full Nanite, World Partition streaming, texture streaming,
  painted landscape layers, richer materials, UE4 games.
- **M4 Data and sound:** DataTables, curve tables, sound waves.
- **M5 UI and AI:** UMG widgets to Bevy UI; Behavior Tree and Blackboard
  runtime; navmesh.
- **M6 Logic:** Blueprint bytecode decompiler; Unreal-style gameplay API
  (Actor, Pawn, Controller, GameMode).

## Risks

- **`oozextract` licence:** it is published as MIT but is a port of `ooz`,
  which is GPL-3.0. Check where it came from before making it the default.
  If in doubt, use `ymelois/oodle-rs` (Apache-2.0) or make the pure-Rust
  decoder opt-in too.
- **`repak` and `retoc` handle Oodle their own way** (`repak` downloads Epic's library). Plugging in our decompress trait may need a small patch or fork.
- **`repak` and `retoc` are git-only dependencies.** crates.io rejects those,
  so publishing our crates later means vendoring or upstream releases.
- **Zen packages before UE 5.3** may need version overrides in `retoc`.
- **Nanite fallback meshes** can be very low detail in some games. Full
  Nanite decoding is in M3.
- **Loading every World Partition cell** may be slow or run out of memory on
  big maps. Streaming is in M3.
- **Per-game quirks** (custom encryption, changed serialisation) exist in
  many games. The real test game may need a small quirk hook in the profile.
