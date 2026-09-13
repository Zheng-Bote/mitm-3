# Changelog

All notable changes to the **MitM-3 Data Aggregator** project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [2.0.0] - 2026-09-13

### Added

- **Core-Layer Splitting**: Separated the legacy scheduler into distinct Rust 2024 microservices (`mitm_http-server`, `mitm_scheduler-server`, `mitm_iam-server`).
- **Identity & Access Management (IAM)**: New `mitm_iam-server` (Rust) providing OIDC/OAuth2, MFA, and strict ABAC/RBAC authorization for PII data.
- **Delivery-Layer Migration**: Re-architected the delivery layer into Rust 2024 components specifically tailored for targets (`mitm_apigee` and `mitm_cority`).
- **Agentic SDD**: Introduced `.sdd` architectural contracts and GitHub Spec Kit (`specs/`) for isolated, flow-forward feature development.
- **Medical Devices Interface**: Added explicit architecture scope and interfaces for securely collecting data from Medical Devices.
- **Admin Pane Extensibility**: Added the **MTA (Medical Technical Assistant)** Desktop Frontend (Qt6/C++23) and Web Frontend (Angular 22) alongside the original Desktop Frontend.
- **Rust Workspace**: Created a central `Cargo.toml` workspace managing all new Rust components.
- **Architecture Documentation**: Added `mitm-3.sdd`, updated `architecture.md`, and `concept_mitm-3_aggregator.md` to reflect the new hybrid Rust/Go ecosystem.

### Changed

- **Tech Stack Paradigm**: Evolved from a pure Go backend (`mitm-2`) to a hybrid topology utilizing Rust 2024 for Core, Delivery, and Maintenance layers, while retaining Go for Collectors.
- **Transformation Layer**: Moved to branch `mitm-3_v2.xx` and ported core mapping/validation engines to Rust.
- **API Versioning**: Upgraded to a strict domain-driven versioning pattern (e.g., `/api/admin/v1`, `/api/transformation/v1`).
- **Documentation Language**: Enforced English as the mandatory language for all specifications, issues, and code documentation.
