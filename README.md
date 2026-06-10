# skill-peak-close-audit

Sensor-verified QA of PEAK tickets in Claude Chat. Give it a list of ticket IDs (action, alert, or mixed) and it decides which **actions can be confidently closed**.

It returns a markdown table — one row per ticket with a verdict (✅ CLOSE / 🚨 KEEP / ⚠️ INVESTIGATE / ⏳ HOLD) and an evidence one-liner — then offers to batch-close the ✅ ones on your confirmation.

## Pre-requisites

This skill requires the **PEAK MCP connector** to be installed and authenticated in Claude.

- MCP URL: `https://api.cimenviro.com/mcp`
- Auth: OAuth 2.0

In Claude, go to **Settings > Connectors > Add custom connector**, paste the URL above, and complete the OAuth sign-in with your PEAK account.

## Install

1. Click **Code > Download ZIP** from the Github repo
2. In Claude, go to **Customize > Skills > Create Skill > Upload a skill**
3. Upload the ZIP file as a skill

## Usage

Run `/peak-qa-actions` and supply a list of ticket IDs:

- `/peak-qa-actions audit these: 48213, 48590, 49001`
- `/peak-qa-actions` then paste a list of IDs when prompted
