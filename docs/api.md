# API Reference

The Seed exposes a REST API on `http://169.254.42.1/`. All endpoints return JSON unless noted.

## Status

- `GET /api/v1/status` - Device state, firmware version, uptime, vector count

## Cogs

- `GET /api/v1/apps/installed` - installed cogs
- `GET /api/v1/apps/available` - cogs in the store
- `POST /api/v1/apps/install` - install a cog
- `DELETE /api/v1/apps/:id` - uninstall

## Mesh

- `GET /api/v1/mesh/status` - peer list, connectivity
- `POST /api/v1/mesh/join` - join an existing mesh

## Upgrade

- `GET /api/v1/upgrade/check` - check for new firmware
- `POST /api/v1/upgrade/apply` - trigger OTA

Full OpenAPI spec: available at `http://169.254.42.1/openapi.json` on a running seed.
