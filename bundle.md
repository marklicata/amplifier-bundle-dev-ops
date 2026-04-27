---
bundle:
  name: dev-ops
  version: 1.0.0
  description: Complete DevOps methodology — 7-stage workflow, automated recipes, team patterns, Azure deployment integration

includes:
  - bundle: git+https://github.com/microsoft/amplifier-foundation@main
  - bundle: git+https://github.com/microsoft/amplifier-bundle-recipes@main
  - bundle: git+https://github.com/marklicata/amplifier-bundle-azure-zap  # Azure deployment orchestration
  - bundle: dev-ops:behaviors/dev-ops
---

# DevOps Methodology Bundle

Complete DevOps methodology bundle for systematic development from code to production, with Azure deployment integration.

@dev-ops:context/devops-awareness.md

---

@foundation:context/shared/common-system-base.md
