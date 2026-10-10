# ZenOS-AI Architecture

This directory is the Book of Friday, the architecture record for ZenOS-AI. Volume 2 describes the system as it ships, starting with 2026.10.0 'Tron'.

* [Preface](00_preface.md): why the book was rewritten, and the thesis.
* [Contents](00_toc.md): every chapter, by part.

The book describes behavior you can find in the code. Anything designed but not built is marked **Not yet built**, and all of those are collected in [Appendix E](27_appendices.md#e-not-yet-built). If the code and the book disagree, the code is right.

For operating ZenOS rather than understanding its design, start with the working docs below.

## Beyond the book

The book explains the design. These are the working docs:

* [Documentation Hub](../readme.md): everything, organized by area
* [Getting Started](../getting_started/readme.md): install, first run, and operator manuals
* [Script readmes](../scripts/readme.md): one reference per tool, with every mode
* [Component docs](../components/): Room Manager, Media Manager, ZenLux, AlertManager, and the rest
* [Kung Fu](../kung_fu/readme.md): building your own components
* [Cabinets](../cabinets/readme.md), [Library](../library/readme.md), [Templates](../custom_templates/readme.md), [Sensors](../sensors/readme.md)
* [Release notes](../releases/tron.md)

Volume 1, the design history, is the tree at commit `57a935f`.
