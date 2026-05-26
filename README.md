# SCHC Transport Encapsulation (SCHC-TE)

This repository contains the draft "SCHC Transport Encapsulation" by Quentin Lampin (Orange).

## Abstract

This document specifies a minimal Transport Encapsulation (SCHC-TE) for the Static Context Header Compression (SCHC) framework. SCHC-TE provides Session multiplexing and Context version awareness for deployments where extrinsic Discriminators are insufficient.

## Building the Draft

This draft uses [kramdown-rfc](https://github.com/cabo/kramdown-rfc) and [xml2rfc](https://github.com/ietf-tools/xml2rfc) for generating the various output formats.

### Prerequisites

Make sure you have the following tools installed:

```bash
# Install kramdown-rfc (Ruby gem)
gem install kramdown-rfc

# Install xml2rfc (Python package)
# Using uv (recommended)
uv tool install xml2rfc

# Or using pip
pip install xml2rfc
```

### Building

Use the provided Makefile to build the draft:

```bash
# Generate TXT, HTML, and XML files
make

# Generate all formats including PDF
make complete

# Generate specific format
make txt
make html
make xml
make pdf

# Check if tools are installed
make check-tools

# Validate the XML
make validate

# Clean generated files
make clean

# Show help
make help
```

### Manual Generation

You can also generate the files manually:

```bash
# Generate TXT file
kdrfc draft-lampin-schc-transport-encapsulation-00.md

# Generate HTML file
kdrfc --html draft-lampin-schc-transport-encapsulation-00.md

# Generate XML file
kdrfc --xml draft-lampin-schc-transport-encapsulation-00.md

# Generate PDF file
kdrfc --pdf draft-lampin-schc-transport-encapsulation-00.md
```

## Files

- `draft-lampin-schc-transport-encapsulation-00.md` - Main draft source in Markdown format
- `Makefile` - Build automation
- `README.md` - This file
- `.gitignore` - Git ignore rules

## Contributing

This is an IETF draft. Please follow the standard IETF process for contributions.

## License

This document is subject to the rights, licenses and restrictions contained in BCP 78, and except as set forth therein, the authors retain all their rights.

## Author

Quentin Lampin
Orange
quentin.lampin@orange.com
