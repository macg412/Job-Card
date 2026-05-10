# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A single-file HTML application (`JOB CARD V1.6.html`) for Downer NZ Ltd field staff to create, save, and submit electrical job cards as PDFs via email. No build system, no server — open the file directly in a browser.

## Running the App

Open `JOB CARD V1.6.html` directly in a browser. There are no dependencies to install, no build step, and no dev server. The only external dependency is jsPDF loaded from a CDN:
```
https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js
```
The app requires a browser with IndexedDB support (all modern browsers).

## Architecture

Everything — HTML, CSS, and JavaScript — lives in the single file, structured in clearly-marked sections:

**Two screens** (only one visible at a time, toggled via `showScreen()`):
- `#screenLibrary` — list of saved drafts
- `#screenForm` — the job card form

**Persistence layer (IndexedDB)**:
- Database: `JobCardDraftsDB`, store: `drafts`, keyed by `id` (e.g. `draft_1716000000000`)
- Draft objects: `{ id, updatedAt, data }` where `data` is `collectFormData()` output
- Auto-save triggers 1.5 s after any field change via `scheduleSave()`

**PDF generation** (`generateAndSend()`):
- Uses jsPDF in A4 format
- Embeds photos (resized to max 1800 px, JPEG 0.85) and the signature canvas as PNG
- After saving the PDF, opens a `mailto:` link pre-filled with job details; the user must manually attach the PDF
- Email routing by job type:
  - Fault → `EnergyValleyFaults@downergroup.com`
  - Defect → `EnergyValleyDefects@downergroup.com`
  - Project → `EnergyValleyProjects@downer.co.nz`

**Photo handling**:
- Three groups (`before`, `after`, `add`), 4 slots each, stored as base64 data URLs in the `photos` object
- Images are downscaled with `createImageBitmap` (falls back to `<img>` element) before storing
- Photos are included in IndexedDB drafts as base64 strings — large photos will inflate storage significantly

**Signature**:
- Canvas-based, supports mouse and touch (`touch-action: none` on canvas)
- `sigHasContent` flag tracked separately from canvas state; restored from `dataUrl` on draft load
- `resizeSig()` must be called after the canvas is visible to get correct dimensions

**Mandatory fields** (validated in `validateForm()`):
- Job Type, Date, 500 WO Number (exactly 8 digits), Job Location, Name, Job Details, Staff on Site (0–100), Signature
- PO For becomes mandatory when the PO Required toggle is on

## Key Patterns

- `gv(id)` — get trimmed string value from an input
- `tb(id)` — get boolean checked state from a checkbox
- `sv(id, v)` / `sc(id, v)` — set value / checked state
- `escH(s)` — HTML-escape for injecting text into `innerHTML`
- `fmtD(d)` / `fmtF(d)` — format ISO date as `DD/MM/YYYY` / `DD.MM.YY` (used in PDF and filename)
