# claude-skills

Claude Code skill templates for IoT, agriculture, DevOps, and more.

## Usage

Copy desired skill files to your project's `.claude/skills/` directory:

```bash
# Copy a single skill
cp skills/w5500-evb-pico-guide.md /path/to/your/project/.claude/skills/

# Copy all skills
cp skills/*.md /path/to/your/project/.claude/skills/
```

## Skills

### IoT / Embedded

| Skill | Description |
|-------|-------------|
| agri-iot-board-design-template | Agricultural IoT board design templates |
| enclosure-generator | Enclosure design generator (OpenSCAD) |
| esp32-cam-timelapse-builder | ESP32-CAM timelapse system builder |
| i2c-sensor-auto-detector | I2C sensor auto-detection tool |
| iot-auto-test-generator | IoT automated test generator (includes MQTT testing) |
| iot-design-doc-generator | IoT design document generator (includes system spec) |
| iot-timer-db-generator | IoT timer database schema generator |
| pico-mqtt-health-checker | Pico MQTT health check tool |
| pico-mqtt-repl-tester | Pico MQTT REPL tester |
| pinout-diagram-generator | MCU pinout diagram generator |
| sensor-driver-generator | I2C/SPI sensor driver generator |
| w5500-evb-pico-guide | W5500-EVB-Pico guide (includes connection test) |

### Home Automation / Agriculture

| Skill | Description |
|-------|-------------|
| env-derived-values-calculator | Environmental derived values calculator (VPD, DLI, etc.) |
| ha-os-network-discovery | Home Assistant OS network discovery |
| homeassistant-agri-starter | HA agricultural starter (includes integration designer) |
| nodered-error-alert-flow-generator | Node-RED error alert flow generator |
| nodered-setup-guide | Node-RED setup guide |
| nodered-timer-flow-generator | Node-RED timer flow generator (includes timeslot) |

### DevOps / Infrastructure

| Skill | Description |
|-------|-------------|
| docker-compose-generator | Docker Compose file generator |
| docker-compose-test | Docker Compose test runner |
| docker-pytest-runner | Docker-based pytest runner |
| git-confidential-docs-isolation | Git confidential docs isolation |
| wireguard-peer-manager | WireGuard peer management (server + client) |

### Web / Backend

| Skill | Description |
|-------|-------------|
| crud-business-logic-generator | CRUD business logic generator |
| csv-safe-wrapper-generator | CSV safe wrapper generator |
| dataclass-model-generator | Python dataclass model generator |
| frontend-backend-schema-migration | Frontend-backend schema migration |
| playwright-e2e-scaffold | Playwright E2E test scaffold |

### Research / Documentation

| Skill | Description |
|-------|-------------|
| oss-competitive-analysis | OSS competitive analysis (includes tech comparison) |
| oss-research-reporter | OSS research reporter |
| raspberrypi-os-installer-guide | Raspberry Pi OS installer guide |
| sequential-technical-guide-writer | Sequential technical guide writer |

## License

MIT
