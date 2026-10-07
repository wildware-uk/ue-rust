# ue-rust

Load a shipped Unreal Engine 5 game's files straight into [Bevy](https://bevyengine.org), in pure Rust.

The aim is to help port Unreal games to Rust + Bevy. Point it at a game's `.pak` or `.utoc`/`.ucas` files, and Bevy can use what is inside: meshes, textures, materials and levels first, then animation, UI, AI and Blueprint logic.

**Status:** design stage. Milestone 1 is "pick a map from a shipped UE5 game and walk it in Bevy". See the [milestone 1 design](docs/superpowers/specs/2026-10-07-level-walk-design.md) and the [issues](https://github.com/wildware-uk/ue-rust/issues).

## Ground rules

- This repo never contains game content. You supply the game files, decryption keys and `.usmap` name maps.
- Rust only. Logic ported from other projects comes only from permissive licences and is credited in [NOTICE](NOTICE).

## Licence

Apache-2.0. See [LICENSE](LICENSE).
