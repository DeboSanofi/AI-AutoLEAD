# AI AutoLEAD – ChemDiary Field Mapping Matrix (Draft)

Draft of 24 Nov 2026.

This matrix links every field GBG can send to the ChemDiary template field it fills, so the Tech Lead can size the API and the business can confirm what is mandatory. The block and column names come from snapshots of the AutoLEAD template; mandatory flags, allowed values and GBG sources are still to be completed.

## 1. Purpose and how to complete it

- **Source of truth:** ChemDiary data model and template (for field names, mandatory flags, dropdown values); GBG project data structure (for what GBG already holds).
- **Status values:** To do, Draft, Confirmed by business, Confirmed by Biosero.
- **Rule:** A field with no GBG source goes on the gap list in section 4. It is never silently dropped.
- **Scope:** ChemDiary integration only.

## 2. Template blocks

The AutoLEAD template has eight blocks under "Synthetic Chemistry & Samples". The business confirmed that GBG fills these sections; how they map to API endpoints is still to be agreed.

| # | Block | Type and content | Phase 1 endpoint? |
| --- | --- | --- | --- |
| 1 | Reaction Scheme | Structure sketcher plus a Reactants table (24 columns) and a Products table (17 columns) | Yes |
| 2 | Procedure | Rich-text worksheet, empty in the template | Yes |
| 3 | Placeholder worksheet procedure | Rich-text worksheet, empty in the template | Yes, to confirm |
| 4 | Analytic | Text with the HPLC method, a note that the chromatogram is attached, and a variable "[add found/not found]" | Yes |
| 5 | Samples Table | Table, 17 columns | Yes |
| 6 | Container table | Table, 12 columns | Yes |
| 7 | Refresh analytical results | Information block with a button; analytical requests are created by adding tasks to samples | No, project Phase 2 |
| 8 | Conclusion | Rich-text worksheet, empty in the template | Yes, to confirm |

There is no separate work-up worksheet in this template. Confirm where work-up text goes (Procedure, or the placeholder worksheet). The Analytic block says the chromatogram is attached, so the API may need to upload a file.

## 3. Mapping matrix

Column names are as shown in the template. Every row starts at status To do. "Calculated?" means the value may be derived by ChemDiary from the structure, so GBG would not send it; confirm for each.

| Block | ChemDiary field | Mandatory? | GBG source | Notes |
| --- | --- | --- | --- | --- |
| Reactants | Rxn ID | To confirm | To confirm | Key shared with Products and Samples |
| Reactants | Reactant | To confirm | To confirm | |
| Reactants | Role | To confirm | To confirm | Likely a dropdown |
| Reactants | Limit? | To confirm | To confirm | |
| Reactants | Barcode | To confirm | To confirm | |
| Reactants | Location | To confirm | To confirm | |
| Reactants | Reference | To confirm | To confirm | |
| Reactants | Material ID | To confirm | GBG knows the building block ID | |
| Reactants | MF, FM, EM, MW | To confirm | To confirm | Calculated? |
| Reactants | Eq | To confirm | To confirm | |
| Reactants | Sample Mass, Moles, Molarity, Vol, d | To confirm | To confirm | Amounts used in the run |
| Reactants | CAS Number, MFCD, IUPAC Name | To confirm | Maybe another source | Building block properties may not be in GBG |
| Reactants | Structural Comment, % Wt, Comment | To confirm | To confirm | |
| Products | Rxn ID | To confirm | To confirm | |
| Products | Product ID, Product | To confirm | To confirm | |
| Products | MF, FM, EM | To confirm | To confirm | Calculated? |
| Products | Actual Mass, Actual Mol | To confirm | To confirm | |
| Products | Theo Mass, Theo Mol, Yield | To confirm | To confirm | Calculated? |
| Products | Purity | To confirm | Analytics result | Returned to GBG in project Phase 2 |
| Products | Structural Comment, IUPAC Name, CAS | To confirm | To confirm | |
| Products | Biomolecule 1 ID, Biomolecule 2 ID | To confirm | To confirm | |
| Procedure | Procedure text | To confirm | To confirm | Free rich text; or fixed template text with variables? |
| Placeholder worksheet procedure | Worksheet text | To confirm | To confirm | Is it needed? |
| Analytic | Analysis text (HPLC method) | To confirm | To confirm | Fixed text in the template |
| Analytic | "[add found/not found]" | To confirm | To confirm | Variable to fill |
| Analytic | Chromatogram attachment | To confirm | To confirm | File upload via API? |
| Samples Table | ID, Rxn ID | To confirm | To confirm | Auto-populated from structures? |
| Samples Table | Chemical Name, Structural comment | To confirm | To confirm | |
| Samples Table | Corporate ID, Corporate ID status | To confirm | To confirm | Linked to compound registration |
| Samples Table | Amount, Purity, Batch Comment | To confirm | To confirm | |
| Samples Table | MF, FM, Status, Material ID | To confirm | To confirm | |
| Samples Table | Site, Project Code, Origin Code, Department | To confirm | To confirm | Possibly constants per lab or project |
| Container table | Sample ID (link) | To confirm | To confirm | Links to Samples Table |
| Container table | Container Type, Container Barcode, Container ID | To confirm | To confirm | |
| Container table | Concentration, Amount, Solvent/Buffer | To confirm | To confirm | |
| Container table | Project Code, IDG 1, IDG2, IDG3, IDG4 | To confirm | To confirm | Meaning of IDG fields to confirm |
| Conclusion | Conclusion text | To confirm | To confirm | May stay human-entered |

Split rows with grouped fields into one row per field once the data model is available.

## 4. Open questions and inputs needed

| # | Question or input | From | Needed for |
| --- | --- | --- | --- |
| 1 | Data model: field names or IDs, mandatory flags, dropdown values, validation rules | Business, ChemDiary vendor | Completing section 3 |
| 2 | GBG project data structure: which fields it holds | Biosero, Jan H. | GBG source column |
| 3 | Which fields are calculated by ChemDiary from the structure (MF, FM, EM, MW, Theo Mass, Theo Mol, Yield)? | ChemDiary vendor | Payload size |
| 4 | Where does work-up text go, and is the placeholder worksheet used? | Jan H. | Endpoint list |
| 5 | Do API writes trigger the template automations (e.g. structure fills Samples Table and Container table)? | ChemDiary vendor | Design |
| 6 | Where do missing building block properties (CAS, MFCD, IUPAC Name) come from? | Jan H., users | Gap list |
| 7 | Source and meaning of Corporate ID, Corporate ID status, Site, Project Code, Origin Code, Department, IDG 1 to 4 | Jan H., users | Samples and Container fields |
| 8 | Can the API attach the chromatogram file to the Analytic block? | ChemDiary vendor | Analytic endpoint |
| 9 | Is Procedure free text, or fixed text with variables to fill? | Second Jan | Procedure endpoint |
| 10 | One completed experiment per block with real values | Second Jan | Test cases |
