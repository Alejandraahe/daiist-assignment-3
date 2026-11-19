# Assignment 3 Report

- **Name**: Alejandra Hernandez
- **Student ID**: 20122
- **Group**: BBADBA 5A 

## Use case

A pharmacy assistant in a community pharmacy in Spain. Patients taking blood thinnersask them questions at the counter, such as “I forgot a dose, what do I do?” or “Can I take ibuprofen with it?” With the answer and with the citation to the official EMA leaflet, they answer the patient or decide to send them to the pharmacist or doctor. Th cost of a wrong answer vs "I don’t know, is that a wrong answer can cause major bleeding or a stroke. “I don’t know” only costs a few minutes while the assistant asks the pharmacist. So the system must refuse rather than guess.

## Corpus

| Document | URL | Date | Licence | Words (raw / after cleaning) |
|---|---|---|---|---|
| Eliquis (apixaban), EMA product information | https://www.ema.europa.eu/en/documents/product-information/eliquis-epar-product-information_en.pdf | 25 March 2026 | EMA content may be reproduced provided the source is acknowledged | 68,262 / 37,185 |
| Xarelto (rivaroxaban), EMA product information | https://www.ema.europa.eu/en/documents/product-information/xarelto-epar-product-information_en.pdf | 27 July 2026 | same | 104,262 / 57,412 |
| Pradaxa (dabigatran), EMA product information | https://www.ema.europa.eu/en/documents/product-information/pradaxa-epar-product-information_en.pdf | 4 September 2026 | same | 63,320 / 35,242 |
| **Total** | | | | **235,844 / 129,839** |

Dates are the creation dates of the PDF files published by EMA (the documents themselves only link to the EMA website for the revision date). No personal data: these are official regulatory documents about medicines, not about patients.

## Ingestion

Extraction
`index.py` downloads the three PDFs from EMA into `corpus/` (git-ignored) and reads them page by page with `pypdf`, so every piece of text keeps its page number.

Cleaning
Words split by a hyphen at a line end are joined, repeated spaces and empty lines are collapsed, and lines that only hold a page number are removed.

Structure
The code walks the text line by line and tracks three things: the *part* (SmPC = technical sheet, Annex II, Labelling, or patient Leaflet), the *product* (e.g. "Eliquis 2.5 mg film-coated tablets") and the *section* (e.g. "4.2 Posology"). A numbered line only counts as a heading if its number goes forward, so numbers inside tables are not mistaken for headings.

Extraction problem I fixed: repeated text
Each EMA document repeats almost the same SmPC and leaflet for every strength and form (2.5 mg, 5 mg, children's granules...). Without a fix, the top 5 search results were often the same paragraph five times. The code removes any paragraph (8+ words) that already appeared earlier in the same document: 95,919 words (about 40%) were removed. I also skip the Labelling part (carton and blister text, 10,086 words), which never answers a question.

Known imperfection
Numbered sub-documents inside the leaflet (the patient alert card, and the children's Instructions for Use with steps "1.", "2."...) are sometimes read as leaflet sections 1-6, so a few chunks get a wrong section label.

Chunk metadata
Every chunk carries: `source` (eliquis / xarelto / pradaxa), `page`, `location` (part + section, e.g. `SmPC 4.2`), a `header` with the product name, and an `id` that is a hash of its text (so unchanged text is never embedded twice).

Chunks.
Two designs, both about 300 words per chunk: *fixed* = 520 chunks; *structured* = 604 chunks. Embedded with `gemini-embedding-2` (768 dimensions) and stored in ChromaDB in `index/chroma` (22.3 MB). Vectors are saved after every batch of 50, and `main.py index` embeds only chunks without a saved vector. The work was spread over two days because of the free quota (about 1,000 embeddings a day).

## Test set

I wrote 32 questions from the three documents before comparing any retrieval design. For each question I read the relevant section (with a small helper, `peek.py`, that prints any section of a medicine), chose one concrete fact (a dose, a prohibition, an "if... then" rule), and phrased the question the way a patient or customer would ask at the counter, not with the words of the PDF.

| Kind | Count |
|---|---|
| single (one passage) | 20 |
| multi (two passages, usually comparing two medicines) | 6 |
| unanswerable (on topic but not in the documents) | 6 |
| **Total** | **32** |

Questions cover dosing (4.2), contraindications (4.3), warnings (4.4), interactions (4.5), pregnancy (4.6), side effects (4.8), overdose (4.9), storage, and the patient leaflet. For the unanswerable questions I searched all pages of the three PDFs for the key words (edoxaban, price, alcohol, ginkgo, generic, tattoo) and confirmed none appear. One of them (edoxaban / Lixiana) is used later for the before/after demo.

Two examples:
- **q02 (single):** "My father had an eye operation last week, can he start taking Eliquis?" → No: recent eye, brain or spinal surgery is a contraindication (eliquis, SmPC 4.3).
- **q22 (multi):** "After an overdose, can dialysis remove Xarelto or Pradaxa?" → Rivaroxaban is not expected to be dialysable; dabigatran can be dialysed (xarelto + pradaxa, SmPC 4.9).

## Retrieval

*Hit rate at 1/3/5 per design, words added to the prompt, the design you chose.*

## Tools and subagent

*Your tools' descriptions, one search result, the subagent's instructions, one transcript from your editor.*

## Adding a document

*What happens when a document is already indexed. The unanswerable question, the document you added, the answer before and after.*

## Harness and guardrails

*Your AGENTS.md rules, the approval step, and your extra guardrail with one case where it fired.*

## Evaluation

*Code checks, the judge's rubric, your agreement with the judge, and results: correct, unsupported, correct refusals, wrong refusals.*

## Failures

*Failed questions by cause, with counts, and what to fix first.*

## Decision

*Would you let users rely on it, for what, with which safeguard? Cost and latency per question.*

## AI use

I used Claude (Anthropic) as a step-by-step tutor throughout. Claude wrote the code (index.py, server.py, agent.py, evaluate.py, peek.py) and explained it; I ran and tested it, and fixed the errors with its help. For the test set, Claude showed me the sections and proposed options for each question; I chose which fact to test, which answer was correct, and checked the unanswerable ones were not in the documents. The use case, the chunking choices (removing duplicates, skipping labelling), the AGENTS.md rules and the judge rubric were my decisions from options Claude explained. Claude drafted parts of this report from my work; I reviewed and edited them.
