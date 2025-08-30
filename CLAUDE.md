# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview
This is a GitHub Pages repository (geminiwen.github.io) that includes a custom Claude Code slash command for personal KPI tracking and growth monitoring.

## Custom Claude Code Commands

### /weekly check-in (aliases: /weekly, /checkin, /kpi)
- **Purpose**: Automated weekly KPI collection and personal dashboard generation
- **Location**: `.claude/commands/weekly-checkin.md`
- **Data Storage**: `metrics`
- **Dashboard Output**: `personal-dashboard.html`

**Usage**: `/weekly-checkin`

**Features**:
- Collects Business KPIs (revenue, customers, projects, meetings)
- Tracks Career KPIs (skills, learning, networking, code commits) 
- Monitors Personal KPIs (health, exercise, sleep, meditation)
- Records mood, achievements, and challenges
- Generates visual dashboard with trend charts
- Calculates weekly scores and progress indicators

## Architecture
- **Command System**: Custom Claude Code slash commands in `.claude/commands/`
- **Dashboard**: Self-contained HTML with embedded CSS/JS
- **Visualization**: Chart.js for trend analysis and scoring

## Development Notes
- The weekly check-in command automatically creates data directory structure
- Dashboard is regenerated each time command is run
- Dashboard includes 8-week trend analysis and scoring system
- use `trend-dashboard-generator` agent to generate dashboard if needed

## Repository Structure
```
├── .claude/
│   ├── commands/
│   │   └── weekly-checkin.md     # Main slash command
├── personal-dashboard.html       # Generated dashboard
└── CLAUDE.md                     # This file
```


