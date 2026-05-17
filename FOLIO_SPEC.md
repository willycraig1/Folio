# Folio — Product Spec

A lightweight, single-file browser-based document editor. Create multi-page documents with rich text, images, and basic shapes. Export to PDF. No install, no account, no server. Files never leave your machine.

---

## Vision

Word is bloated and has poor graphic control. Pages is locked to Apple. Google Docs requires a server. The gap: a clean, fast document editor that treats layout and images as first-class citizens alongside text — without crossing into illustration territory (no bezier pen tool, no boolean path ops, no artboards).

Think of it as Word if Word were designed by someone who also used Figma. The primary output format is PDF.

---

## What It Is Not

- Not a full word processor: no mail merge, no tracked changes, no macros, no table of contents generation
- Not a vector illustration tool: no pen tool, no bezier curves, no boolean path operations
- Not a collaborative tool: single-user, local only
- Not a spreadsheet or presentation tool
- Not a Figma clone: no components, no symbols, no prototyping

---

## Core MVP Features

### Pages
- Fixed-size pages: Letter (8.5×11"), A4, Legal — selectable per document
- Add, remove, and reorder pages (drag thumbnails in a sidebar)
- Page background color (default white)
- Configurable margins as a guide overlay (not enforced)

### Text Boxes
- Place a text box anywhere on the page by clicking and dragging
- Resize and reposition freely (drag corners/edges)
- Rich text within a box:
  - Font family (curated set of ~8 web-safe + CDN fonts: serif, sans-serif, monospace options)
  - Font size, bold, italic, underline, strikethrough
  - Text color
  - Alignment: left, center, right, justify
  - Line height control
- Text boxes do not reflow between boxes (no linked text frames in MVP)

### Images
- Drag image files (JPG, PNG) onto the page to place them
- Resize maintaining aspect ratio (shift-drag to free resize)
- Reposition freely
- Opacity control
- No in-app cropping in MVP (crop before importing)

### Shapes
- Rectangle, ellipse, line — placed by click-drag on the page
- Fill color (including transparent/none)
- Stroke color and weight
- Corner radius on rectangles
- No freeform paths in MVP

### Selection and Arrangement
- Click to select any element
- Shift-click or drag-select for multi-select
- Move selected elements with arrow keys (1px; shift+arrow = 10px)
- Z-order controls: bring forward, send backward, bring to front, send to back
- Align tools: align left/center/right/top/middle/bottom edges of selection
- Distribute evenly (horizontal and vertical)
- Group/ungroup elements

### Export
- Export to PDF via pdf-lib
- Sequential filenames: folio-1.pdf, folio-2.pdf, etc.
- All pages exported in order with correct dimensions

### Undo / Redo
- Full command history (Cmd/Ctrl+Z / Cmd/Ctrl+Shift+Z)
- Minimum 50-step history
- Every user action (move, resize, text edit, style change, add/delete) must be undoable
- **This is non-negotiable and must be designed in from day one**

---

## Architecture

### Working Principle
Maintain a clean **JSON document model** as the single source of truth. The UI renders from the model; export reads from the model. Never read back from the DOM to determine document state.

### Document Model

```javascript
{
  version: 1,
  pageSize: 'letter',           // 'letter' | 'a4' | 'legal'
  pages: [
    {
      id: 0,
      background: '#ffffff',
      elements: [
        {
          id: 0,
          type: 'text',         // 'text' | 'image' | 'rect' | 'ellipse' | 'line'
          x: 72,                // points from left (1pt = 1/72 inch)
          y: 72,                // points from top
          width: 468,
          height: 200,
          rotation: 0,          // degrees
          opacity: 1,
          zIndex: 0,
          // type-specific fields below
          runs: [               // rich text as an array of styled runs
            { text: 'Hello ', bold: true, italic: false, underline: false,
              fontSize: 24, fontFamily: 'sans-serif', color: '#000000' }
          ],
          textAlign: 'left',
          lineHeight: 1.4,
        },
        {
          id: 1,
          type: 'image',
          x: 72, y: 300, width: 200, height: 150,
          rotation: 0, opacity: 1, zIndex: 1,
          dataURL: 'data:image/jpeg;base64,...'
        },
        {
          id: 2,
          type: 'rect',
          x: 300, y: 300, width: 100, height: 60,
          rotation: 0, opacity: 1, zIndex: 2,
          fill: '#e2e8f0',
          stroke: '#94a3b8',
          strokeWidth: 1,
          cornerRadius: 4,
        }
      ]
    }
  ]
}
```

### Coordinate System
Use **points** (1/72 inch) throughout — the native PDF unit. Letter = 612×792pt. This avoids any conversion math at export time. The canvas renders at a scale factor (e.g. 1pt = 1.33px at 96dpi) but the model always stores points.

### Rendering Layer
Each page renders as a `<div class="page">` with explicit pixel dimensions (model points × scale). Elements are `position: absolute` children:
- Text boxes: `<div contenteditable>` with inline styles
- Images: `<img>` 
- Shapes: inline `<svg>` elements

The editing surface is HTML. Do not attempt to use a single `<canvas>` for everything — text editing on canvas requires implementing a full text editor from scratch.

### Selection Layer
A transparent overlay `<div class="selection-overlay">` sits above the page and handles:
- Click/drag-select hit testing
- Rendering selection handles (resize corners, rotation handle)
- Arrow key movement

### Export to PDF
Walk the document model (not the DOM) and write PDF using pdf-lib:
- Pages: `pdfDoc.addPage([width, height])`
- Text: embed font, use `page.drawText()` per run, computing x/y positions
- Images: `pdfDoc.embedJpg/embedPng`, `page.drawImage()`
- Shapes: `page.drawRectangle()`, `page.drawEllipse()`, `page.drawLine()`
- Rotation: pdf-lib supports rotation on all draw operations

### Undo Stack
Use a **command pattern**. Every action is an object with `execute()` and `undo()` methods that operate on the document model. After execute or undo, re-render affected elements.

```javascript
// Example command
{ 
  execute() { element.x = newX; element.y = newY; },
  undo()    { element.x = oldX; element.y = oldY; }
}
```

Store commands in an array with a cursor. Undo moves cursor back; redo moves it forward.

---

## Tech Stack

All dependencies via CDN — no build step, no node_modules.

| Library | Purpose |
|---|---|
| [pdf-lib](https://pdf-lib.js.org/) v1.17.1 | PDF export |
| [opentype.js](https://opentype.js.org/) | Font metrics for accurate text layout (line breaking, character widths) |

Potential additions (evaluate when needed):
- A small color picker library (or build one — they're not large)
- Font files loaded from Google Fonts CDN for the curated font set

---

## UI Layout

```
┌─────────────────────────────────────────────────────┐
│  [Logo]  [toolbar: text/shape/image tools]  [Export] │  ← Header
├──────┬──────────────────────────────────┬────────────┤
│      │                                  │            │
│ Page │         Page Canvas              │ Properties │
│ Side │         (scrollable)             │   Panel    │
│ bar  │                                  │            │
│      │                                  │            │
└──────┴──────────────────────────────────┴────────────┘
```

- **Left sidebar**: page thumbnails, drag to reorder, click to navigate
- **Center canvas**: the document, zoomed to fit by default
- **Right panel**: context-sensitive properties for selected element (position, size, fill, font, etc.)
- **Top toolbar**: tool selector (select, text, rectangle, ellipse, line, image), zoom control, undo/redo

---

## Interaction Model

### Tools
- **Select tool** (V / Escape): click to select, drag to move, handles to resize
- **Text tool** (T): click to create text box, immediately enter edit mode
- **Rectangle** (R), **Ellipse** (E), **Line** (L): click-drag to draw
- **Image**: triggered by dragging a file onto the canvas, or a button in toolbar

### Double-click
- Double-click a text box to enter text editing mode
- Double-click a shape to edit its properties inline

### Keyboard
| Key | Action |
|---|---|
| V / Escape | Switch to select tool |
| T | Text tool |
| R | Rectangle tool |
| E | Ellipse tool |
| Cmd+Z | Undo |
| Cmd+Shift+Z | Redo |
| Cmd+A | Select all on current page |
| Cmd+D | Duplicate selection |
| Delete / Backspace | Delete selected element(s) |
| Arrow keys | Nudge 1pt |
| Shift+Arrow | Nudge 10pt |
| Cmd+Shift+E | Export PDF |
| Cmd+] / Cmd+[ | Bring forward / send backward |

---

## What to Tackle First (Suggested Build Order)

1. **Page canvas** — render a blank page at correct dimensions with zoom
2. **Page sidebar** — add/remove/reorder pages, click to navigate
3. **Rectangle tool** — simplest element, establishes the selection/move/resize pattern for all elements
4. **Selection handles** — resize corners, move, delete; this interaction model is shared by all elements
5. **Undo/redo** — wire in the command pattern before adding more element types
6. **Text box** — hardest element; get basic placement working before rich text
7. **Rich text** — bold, italic, size, color within a text box
8. **Images** — drag to place, resize
9. **Ellipse and line** — straightforward once rect is done
10. **Properties panel** — surface controls for whatever is selected
11. **Export to PDF** — walk the model, write with pdf-lib
12. **Polish** — alignment guides, snapping, zoom controls, font selection

---

## Lessons from PDF Jockey

- **Single-file HTML works beautifully** for tools of this scope. Keep it self-contained.
- **CDN dependencies are fine.** No build step is a feature, not a compromise.
- **Local-only is a privacy selling point.** Lead with it.
- **JSON model as source of truth** — PDF Jockey suffered slightly from reading DOM state; a proper model avoids this entirely.
- **Undo was the one thing PDF Jockey skipped** because operations were low-stakes (you can re-drop a file). A creation tool cannot skip it.
- **Points as the coordinate unit** from day one avoids painful conversion bugs later.
