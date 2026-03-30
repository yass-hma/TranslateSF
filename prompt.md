Build a single-file HTML app (no framework, no backend, vanilla JS + SheetJS via CDN) 
called "STF Translator" for Salesforce Translation Workbench.

## Goal
The user uploads a Salesforce .stf file (up to 60,000 lines) and an Excel file 
(.xlsx, 2 columns: English | Translation). The app matches them, lets the user 
select which Salesforce objects to include, then generates a new .stf file 
containing only the selected translations.

---

## Step 1 — Upload UI
- Two drag & drop zones side by side: one for .stf, one for .xlsx
- Show filename + green checkmark once each file is loaded
- "Analyser" button appears only when both files are uploaded

---

## Step 2 — Parsing (performance is critical, STF can be 60,000 lines)

### Excel parsing:
- Use SheetJS (loaded via CDN)
- Read only columns A and B (row 1 = header, skip it)
- Build a Map<english_value, french_value> (key = trimmed English value)
- Cells with empty French value → store as null in the Map

### STF parsing:
- The STF is a tab-separated text file
- Each line has this structure:
  [type] [TAB] [object] [TAB] [field/key] [TAB] [english_value] [TAB] [french_value or empty]
- Parse line by line using a streaming approach (split by \n, process in chunks 
  of 5000 lines using setTimeout to avoid blocking the UI)
- For each line: check if the english_value exists in the Excel Map
- If yes → keep the line, attach the french_value from the Map (or null if missing)
- If no → discard the line
- Group kept lines by object name → result is an index:
  { "Case": [ {type, field, en, fr, missing: bool}, ... ], "Account": [...], ... }
- Show a progress bar during parsing

---

## Step 3 — Object selection tree

Display the index as a tree, one row per Salesforce object:

  ☑ Case        187 entrées   ⚠️ 12 manquants
  ☑ Account      55 entrées   ✅ complet
  ☑ Lead         38 entrées   ⚠️ 3 manquants

Rules:
- All objects checked by default
- Single checkbox per object (not per field) — user selects at object level
- ⚠️ badge shows count of missing French translations for that object
- ✅ badge if all translations are present
- "Tout cocher" / "Tout décocher" buttons at the top
- Below the tree: show an inline report panel (always visible):
  "32 traductions manquantes au total"
  List each missing entry as: [Object] — [field/key] — EN: [value]

---

## Step 4 — Generate STF

"Générer STF" button (disabled if no object selected).

On click:
- Filter the index to only checked objects
- For each entry: rebuild the STF line using original structure with french_value injected
- Entries where french_value is null → don't include the line in output file
- Build the full output as a single string
- Trigger browser download as output.stf (text/plain, UTF-8)

---

## UI & Design
- Dark theme, clean and professional
- Font: DM Mono for data/values, Syne for titles
- Color palette: dark navy background (#0d0f14), accent blue (#4f8ef7), 
  warning amber (#f59e0b), success green (#10b981)
- 4 clear visual steps with a top progress indicator: 
  Upload → Analyse → Sélection → Génération
- Responsive (works on 1280px+ screens)
- All labels and UI text in French

---

## Technical constraints
- Single .html file, no build step, no backend
- SheetJS loaded from CDN: https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js
- No other external dependencies
- Must handle 60,000 line STF without freezing the browser 
  (use chunked parsing with setTimeout)
- STF parsing must be done only once at upload time
- All data kept in memory as lightweight JS objects after parsing

---

## Error handling
- Wrong file type uploaded → show inline error message
- STF has no matching lines with Excel → show warning "Aucune correspondance trouvée"
- Excel has no data → show error