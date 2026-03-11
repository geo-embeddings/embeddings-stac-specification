# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Embedding extension schema definitions for collection, item properties, asset fields, and extension-specific objects (`patch_layout`, `quantization`, `uncertainty`).
- Embedding-focused Item and Collection examples covering required `emb:model` and `emb:source-data` relations.

### Changed
- Replaced template extension scaffolding with Embedding extension identity (`emb` prefix, embeddings schema identifier, and package metadata).
- Updated README to align with embedding requirements and link canonical STAC and STAC MLM specifications/schemas.
- Updated validator schema mapping to use the Embedding extension schema URL.

### Deprecated

### Removed
- Template-specific field definitions and documentation (`template:*`).

### Fixed
- Example JSON formatting for validator lint compliance.

[Unreleased]: <https://github.com/stac-extensions/template/compare/v0.0.1...HEAD>
