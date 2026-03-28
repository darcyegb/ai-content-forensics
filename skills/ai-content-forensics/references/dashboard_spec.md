# Interactive Dashboard Specification

When `output_mode` is `dashboard`, generate a self-contained HTML file at `dashboard/index.html`.

## Requirements
- **Completely self-contained**: All CSS and JavaScript inline. No external dependencies. Must work when opened as a local file.
- **Data embedded**: Embed the video index and packaging features JSON directly in a `<script>` tag.
- **Responsive**: Works on desktop and tablet.

## Features

### Sortable Video Table
- Columns: title, host, publish date, duration, total views, format family, key title features
- Click column headers to sort
- Click a row to see full details (description, transcript opening, thumbnail URL)

### Filter Controls
- Filter by host (for multi-host channels)
- Filter by format family
- Filter by duration bracket
- Filter by title feature (has_ellipsis, has_negative, has_colon, etc.)
- Filter by performance quartile

### Summary Statistics Panel
- Updates dynamically as filters are applied
- Shows: count, avg views, median views, top title feature effects
- Allows the user to see "what's the avg views for Hank's videos with ellipsis titles over 10 minutes?"

### Scatter Plot
- X axis: selectable (duration, word count, publish date)
- Y axis: total views
- Color: selectable (format family, host, quartile)
- Hover for video title

### Feature Effect Bar Chart
- Shows all boolean title features ranked by effect ratio
- Updates when filters change (so you can see "which features matter for Hank but not John?")

## Implementation Notes
- Use vanilla JavaScript — no React, no build tools
- Use HTML `<canvas>` or inline SVG for charts
- Keep the file under 500KB if possible (embed data, not images)
- The dashboard is an exploration tool, not a presentation — prioritize interactivity over polish
