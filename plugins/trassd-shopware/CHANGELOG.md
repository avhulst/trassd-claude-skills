# Changelog

All notable changes to `trassd-shopware` are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- App-system skills: `shopware-app-fundamentals` (manifest, registration,
  webhooks, signatures), `shopware-app-scripts` (Twig server-side scripting),
  `shopware-app-admin-extensions` (Meteor Admin SDK).
- Theme skill: `shopware-themes` (theme.json, configuration & inheritance, SCSS
  styling, assets/icons).
- Storefront JS skill: `shopware-storefront-javascript` (the JavaScript plugin
  system / interactive storefront components).
- ADR skill `shopware-adrs` (binding Architecture Decision Records distilled
  into actionable rules) + `shopware-adr-auditor` agent (ADR-conformance audit).

## [0.1.0]

### Added

- Initial release.
- Skills: `shopware-plugin-fundamentals`, `shopware-dependency-injection`,
  `shopware-dal-entities`, `shopware-migrations`, `shopware-events`,
  `shopware-store-api-routes`, `shopware-storefront`, `shopware-administration`,
  `shopware-tasks-and-messaging`.
- Agents: `shopware-plugin-reviewer`, `shopware-store-quality-gate`.
