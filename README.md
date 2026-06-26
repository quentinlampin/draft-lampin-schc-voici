# VOICI — Link Multiplexer

This repository contains the draft "VOICI" by Quentin Lampin (Orange).

## Abstract

A minimal, general-purpose link multiplexer for constrained networks. Provides session multiplexing, content dispatch, and optional integrity protection with a 1-byte minimum overhead (for inline Session IDs 0–6). Works with SCHC and other compression mechanisms.

## Building the Draft

This draft uses [kramdown-rfc](https://github.com/cabo/kramdown-rfc) and [xml2rfc](https://github.com/ietf-tools/xml2rfc) for generating the various output formats.

### Prerequisites

```bash
gem install kramdown-rfc

uv tool install xml2rfc
```

### Building

```bash
make           # Generate TXT, HTML, XML
make complete  # All formats including PDF
make validate  # Validate XML
make clean     # Remove generated files
```

## Files

- `draft-lampin-voici-00.md` - Main draft source in Markdown format
- `Makefile` - Build automation

## License

This document is subject to the rights, licenses and restrictions contained in BCP 78.

## Author

Quentin Lampin
Orange
quentin.lampin@orange.com
