# Dataset Documentation — IMPPAT HTI AAC ECFP4

## 1. Overview

This directory (`hti_pipeline/datasets/imppat_hti_aac_ecfp4_v1/`) contains the primary machine learning dataset package for predicting compound–target interactions (CTI) and tracing multi-hop herb–target candidate pathways in Ayurvedic medicine.

The biological hierarchy underlying the pipeline is:
$$\text{Ayurvedic Medicinal Herb} \longrightarrow \text{Phytochemical Compound} \longrightarrow \text{Human Target Protein}$$

### Scope & Primary Datasets
This documentation focuses on the active tabular modeling and graph-traversal files:
1. **Chemical Representation (`compound_features_ecfp4.csv`)**: 2,048-bit Extended-Connectivity Fingerprints (ECFP4 / Morgan $r=2$) covering all 2,319 canonical phytochemical compounds.
2. **Biological Target Representation (`target_features_aac.csv`)**: 20-dimensional normalized Amino Acid Composition (AAC) feature vectors and official Swiss-Prot recommended protein names for all 1,341 verified human protein targets.
3. **Curated Positive Interactions (`compound_target_labeled.csv`)**: 14,166 deduplicated positive compound–protein associations extracted from IMPPAT 2.0 (cross-referenced with ChEMBL, BindingDB, and NPASS) with verified human UniProt targets (`label=1`, `label_status='IMPPAT_ASSOCIATION'`).
4. **Unobserved Interaction Universe (`compound_target_unlabeled.csv`)**: 3,095,613 candidate pairs representing the full Cartesian product ($2,319 \times 1,341$) excluding known positives, structured for Positive-Unlabeled (PU) learning or ranking evaluations (`label_status='UNLABELED'`).
5. **Multi-Hop Traversal Paths (`herb_compound_target.csv`)**: 126,830 qualified paths linking 60 canonical Ayurvedic herbs to 1,341 human protein targets via intermediate phytochemical compounds and specific plant parts.

*(Note: `compound_target.csv` is the precursor table containing identical 14,166 pairs to `compound_target_labeled.csv` without the `label_status` column; auxiliary audit files `identifier_mapping_audit.csv` and `unresolved_targets.csv` are omitted here per dataset configuration).*

---

## 2. Dataset File Inventory

| Filename | Data Rows | Columns | Purpose | Generation / Source Method |
|:---|---:|---:|:---|:---|
| [`compound_features_ecfp4.csv`](file:///c:/Aryan/BTP/ayurveda-kg-pipeline2/hti_pipeline/datasets/imppat_hti_aac_ecfp4_v1/compound_features_ecfp4.csv) | 2,319 | 2,049 | Compound ECFP4 chemical fingerprint feature matrix | Morgan circular fingerprints (radius 2, 2,048 bits) generated via RDKit from canonical compound SMILES |
| [`target_features_aac.csv`](file:///c:/Aryan/BTP/ayurveda-kg-pipeline2/hti_pipeline/datasets/imppat_hti_aac_ecfp4_v1/target_features_aac.csv) | 1,341 | 23 | Target protein feature matrix (AAC) and UniProt names | 20 amino acid frequencies computed from UniProt FASTA sequences; official recommended protein names fetched via UniProt REST API |
| [`compound_target_labeled.csv`](file:///c:/Aryan/BTP/ayurveda-kg-pipeline2/hti_pipeline/datasets/imppat_hti_aac_ecfp4_v1/compound_target_labeled.csv) | 14,166 | 15 | Labeled positive compound–target dataset for ML baselines | Deduplicated IMPPAT phytochemical-target associations mapped to reviewed human Swiss-Prot targets (`label=1`) |
| [`compound_target_unlabeled.csv`](file:///c:/Aryan/BTP/ayurveda-kg-pipeline2/hti_pipeline/datasets/imppat_hti_aac_ecfp4_v1/compound_target_unlabeled.csv) | 3,095,613 | 3 | Unobserved candidate pairs forming the unlabeled evaluation pool | Cartesian product of all 2,319 compounds and 1,341 targets minus the 14,166 labeled positive pairs (`label_status=UNLABELED`) |
| [`herb_compound_target.csv`](file:///c:/Aryan/BTP/ayurveda-kg-pipeline2/hti_pipeline/datasets/imppat_hti_aac_ecfp4_v1/herb_compound_target.csv) | 126,830 | 22 | Multi-hop paths from Ayurvedic herbs through phytochemicals to human targets | Inner join between canonical herb–compound edges (`herb_compound_edges.tsv`) and `compound_target.csv` |

---

## 3. Individual CSV Documentation

### 3.1 `compound_features_ecfp4.csv`

- **Relative Path**: `datasets/imppat_hti_aac_ecfp4_v1/compound_features_ecfp4.csv`
- **Data Rows**: 2,319
- **Columns**: 2,049
- **Row Semantics**: Each row represents one canonical phytochemical compound from the complete project chemical library, represented as a 2,048-bit folded binary Extended-Connectivity Fingerprint.

#### Column Dictionary
| Column Name | Inferred / Storage Type | Description | Provenance / Verification Details |
|:---|:---|:---|:---|
| `compound_id` | string (`object`) | Unique canonical compound identifier in the form `IMPPAT3_PHYIDxxxxxx`. Unique primary key (2,319 distinct values, 0 nulls). | Canonical compound namespace from `canonical/compounds.tsv`. Verified non-null and unique. |
| `ecfp4_0000` through `ecfp4_2047` (2,048 columns) | integer (`int64`) | Binary fingerprint bit indicator (0 = feature absent, 1 = feature present). | Derived using RDKit's Morgan fingerprint algorithm with radius $r=2$ (equivalent to diameter 4 / ECFP4), folded to a fixed bit length of 2,048 bits. Sourced from `training/features/compound_morgan_ecfp4_r2_2048.npy`. 0 nulls across all 2,048 columns. |

#### Generation Methodology
1. Canonical compound SMILES were extracted and standardized from IMPPAT 2.0 extractions.
2. RDKit generated circular Morgan fingerprints with radius 2 (`AllChem.GetMorganFingerprintAsBitVect(mol, radius=2, nBits=2048)`).
3. The matrix provides complete 100% feature coverage for all 2,319 compounds in the candidate universe, including all 726 compounds present in the positive labeled interactions.

#### Limitations
- **Hash Collisions**: Compressing circular atomic neighborhoods into a 2,048-bit bitset inevitably introduces structural hash collisions.
- **Topological Focus**: ECFP4 captures 2D connectivity; stereochemical differences and 3D conformations are not distinguished.

---

### 3.2 `target_features_aac.csv`

- **Relative Path**: `datasets/imppat_hti_aac_ecfp4_v1/target_features_aac.csv`
- **Data Rows**: 1,341
- **Columns**: 23
- **Row Semantics**: Each row represents one verified human protein target, providing its UniProt accession, official protein name, canonical namespaced identifier, and 20 normalized amino acid composition features.

#### Column Dictionary
| Column Name | Inferred / Storage Type | Description | Provenance / Verification Details |
|:---|:---|:---|:---|
| `uniprot_accession` | string (`object`) | Official UniProtKB accession (e.g., `P15121`, `A5LHX3`). Unique primary key (1,341 unique values, 0 nulls). | Resolved from IMPPAT Entrez gene IDs via `entrez_uniprot_crosswalk.tsv` filtered for reviewed Swiss-Prot entries (`taxid == 9606`). |
| `protein_name` | string (`object`) | Official recommended protein name from UniProtKB Swiss-Prot (e.g., `Acrosin`, `Proteasome subunit beta type-11`, `5-hydroxytryptamine receptor 3E`). | Fetched directly from the UniProt REST API (`https://rest.uniprot.org/uniprotkb/`) by `add_protein_name_to_aac.py`. 0 nulls, 0 empty strings (100% resolved). |
| `protein_id` | string (`object`) | Canonical protein identifier formatted as `UNIPROT:{uniprot_accession}` (e.g., `UNIPROT:P15121`). Unique key (1,341 unique values, 0 nulls). | Namespaced identifier used across graph and tabular pipeline modules. |
| `aac_A` through `aac_Y` (20 columns) | float (`float64`) | Normalized frequency of each standard amino acid in the protein sequence: Alanine (A), Cysteine (C), Aspartic acid (D), Glutamic acid (E), Phenylalanine (F), Glycine (G), Histidine (H), Isoleucine (I), Lysine (K), Leucine (L), Methionine (M), Asparagine (N), Proline (P), Glutamine (Q), Arginine (R), Serine (S), Threonine (T), Valine (V), Tryptophan (W), Tyrosine (Y). | Computed in `build_imppat_hti_aac_ecfp4_v1.py` (`compute_aac`). Values range from $0.0$ to $1.0$. For every row, $\sum_{i=1}^{20} \text{aac}_i = 1.0$ within floating-point precision ($1.0 \pm 10^{-6}$). 0 nulls. |

#### Generation Methodology
1. **Sequence Fetching**: Full canonical FASTA sequences were retrieved via HTTPS from the UniProt REST API (`https://rest.uniprot.org/uniprotkb/{accession}.fasta`) for all 1,341 unique resolved human accessions.
2. **Composition Calculation**:
   $$\text{aac}_{a} = \frac{\text{count}(a)}{\sum_{k \in \mathcal{A}} \text{count}(k)}$$
   where $\mathcal{A}$ is the standard 20 amino acid alphabet. Non-standard IUPAC characters (e.g., $U, O, X, B, Z$) are excluded from the denominator.
3. **Protein Name Enrichment**: `add_protein_name_to_aac.py` queried UniProtKB for `proteinDescription.recommendedName.fullName.value` (with fallback to submitted/alternative names) and positioned `protein_name` immediately after `uniprot_accession`.

#### Limitations
- Discards amino acid sequence order, tertiary 3D folding, active-site conformations, and post-translational modifications.

---

### 3.3 `compound_target_labeled.csv`

- **Relative Path**: `datasets/imppat_hti_aac_ecfp4_v1/compound_target_labeled.csv`
- **Data Rows**: 14,166
- **Columns**: 15
- **Row Semantics**: Each row represents a deduplicated, featurized positive compound–protein interaction pair supported by curated bioactivity databases and mapped to a unique human Swiss-Prot target.

#### Column Dictionary
| Column Name | Inferred / Storage Type | Description | Provenance / Verification Details |
|:---|:---|:---|:---|
| `compound_id` | string (`object`) | Canonical compound identifier (`IMPPAT3_PHYIDxxxxxx`). | Primary compound key (726 distinct compounds). Verified in `compound_features_ecfp4.csv`. 0 nulls. |
| `protein_id` | string (`object`) | Canonical protein identifier (`UNIPROT:{accession}`). | Primary target key (1,341 distinct targets). Verified in `target_features_aac.csv`. 0 nulls. |
| `label` | integer (`int64`) | Binary interaction label. Exactly `1` across all 14,166 rows. | Ground-truth positive association label. |
| `label_status` | string (`object`) | Semantic label designation (`IMPPAT_ASSOCIATION`). | Assigned by `build_labeled_unlabeled.py` to differentiate curated evidence from unlabeled pairs. 0 nulls. |
| `imppat_phytochemical_id` | string (`object`) | Raw IMPPAT phytochemical identifier. | Directly preserved from `phytochemical_target_information.tsv`. 0 nulls. |
| `uniprot_accession` | string (`object`) | Human UniProtKB accession of the target. | Resolved via `entrez_uniprot_crosswalk.tsv`. 0 nulls. |
| `ensembl_gene_id` | string (`object`) | Ensembl gene ID (e.g., `ENSG00000085662`). | Carried through from IMPPAT source records. 0 nulls. |
| `entrez_gene_id` | integer (`int64`) | NCBI Entrez Gene ID (e.g., `231`). | Original numeric gene ID from IMPPAT records. 0 nulls. |
| `hgnc_symbol` | string (`object`) | Official HGNC gene symbol (e.g., `AKR1B1`). | Gene symbol reported in IMPPAT source records. 0 nulls. |
| `source` | string (`object`) | Curated bioactivity databases recording the interaction, pipe-delimited (`BindingDB`, `ChEMBL`, `NPASS`, or combinations like `ChEMBL \| NPASS`). | Aggregated across contributing raw IMPPAT records for this pair. 0 nulls. |
| `evidence_source` | string (`object`) | Duplicate of `source` retained for downstream schema compatibility. | Preserved from `compound_target.csv`. 0 nulls. |
| `n_supporting_records` | integer (`int64`) | Number of raw IMPPAT association rows collapsed into this unique pair (range: 1 to 2). | Count of raw records in `phytochemical_target_information.tsv`. 0 nulls. |
| `has_ecfp4` | boolean (`bool`) | Compound feature availability flag. Exactly `True` for all rows. | Verified present in `compound_features_ecfp4.csv`. 0 nulls. |
| `has_aac` | boolean (`bool`) | Target feature availability flag. Exactly `True` for all rows. | Verified present in `target_features_aac.csv`. 0 nulls. |
| `fully_featurized` | boolean (`bool`) | Complete featurization status (`has_ecfp4 & has_aac`). Exactly `True` for all rows. | 100% of labeled pairs are fully featurized. 0 nulls. |

#### Generation Methodology
1. 14,385 raw IMPPAT association rows were mapped against reviewed human Swiss-Prot accessions, yielding 14,167 successfully resolved rows (218 ambiguous/unmapped rows were quarantined).
2. Deduplication on `(compound_id, protein_id)` merged one duplicate row, resulting in 14,166 unique pairs with aggregated source strings.
3. Formatted by `build_labeled_unlabeled.py` with `label=1` and `label_status='IMPPAT_ASSOCIATION'`.

#### Limitations
- **Positive-Only Selection**: Contains only recorded positive associations.
- **Reporting Bias**: Reflects historical publication and screening interest in well-characterized drug targets.

---

### 3.4 `compound_target_unlabeled.csv`

- **Relative Path**: `datasets/imppat_hti_aac_ecfp4_v1/compound_target_unlabeled.csv`
- **Data Rows**: 3,095,613
- **Columns**: 3
- **Row Semantics**: Each row represents an unobserved candidate compound–protein pair within the complete candidate universe ($2,319 \times 1,341 = 3,109,779$), strictly excluding the 14,166 labeled pairs.

#### Column Dictionary
| Column Name | Inferred / Storage Type | Description | Provenance / Verification Details |
|:---|:---|:---|:---|
| `compound_id` | string (`object`) | Canonical compound identifier across all 2,319 project compounds. | Sourced from `compound_features_ecfp4.csv`. 0 nulls. |
| `protein_id` | string (`object`) | Canonical protein identifier across all 1,341 project targets. | Sourced from `target_features_aac.csv`. 0 nulls. |
| `label_status` | string (`object`) | Semantic label indicator. Exactly `UNLABELED` across all 3,095,613 rows. | Assigned by `build_labeled_unlabeled.py`. 0 nulls. |

#### Generation Methodology
1. Cartesian product of 2,319 canonical compounds and 1,341 verified targets:
   $$\text{Universe Size} = 2,319 \times 1,341 = 3,109,779 \text{ pairs}$$
2. Vectorized anti-join subtracted all 14,166 pairs from `compound_target_labeled.csv`:
   $$\text{Unlabeled Pairs} = 3,109,779 - 14,166 = 3,095,613 \text{ pairs}$$
3. Generated in chunked blocks of 200 compounds using NumPy grid tiling to optimize memory usage.

#### Data Quality & Limitations
- **CRITICAL SEMANTICS — NOT CONFIRMED NEGATIVES**: This file intentionally **omits a `label` column**. These pairs represent unmeasured or unrecorded interactions. Treating them as true negatives ($y=0$) in standard supervised classifiers introduces false-negative label noise. Use Positive-Unlabeled (PU) learning or ranking objectives.
- **Disjointness**: Verified exact 0-pair intersection with `compound_target_labeled.csv` ($\text{labeled} \cap \text{unlabeled} = \emptyset$).

---

### 3.5 `herb_compound_target.csv`

- **Relative Path**: `datasets/imppat_hti_aac_ecfp4_v1/herb_compound_target.csv`
- **Data Rows**: 126,830
- **Columns**: 22
- **Row Semantics**: Each row represents a specific multi-hop network path:
  $$\text{Ayurvedic Medicinal Herb} \xrightarrow[\text{plant\_part}]{\text{reference}} \text{Phytochemical Compound} \longrightarrow \text{Human Target Protein}$$
  connecting an herb to a human target via an intermediate phytochemical identified in a specific plant part with literature citation.

#### Column Dictionary
| Column Name | Inferred / Storage Type | Description | Provenance / Verification Details |
|:---|:---|:---|:---|
| `herb_id` | string (`object`) | Canonical herb identifier in format `IMPPAT_PLANT:IMPPAT3_PLTIDxxxxxx` (60 distinct herbs). | Sourced from `canonical/herbs.tsv`. 0 nulls. |
| `herb_botanical_name` | string (`object`) | Latin botanical binomial (e.g., `Acmella paniculata`, `Acorus calamus`). | Enriched from `canonical/herbs.tsv` (`botanical_name`). 0 nulls. |
| `herb_ayurvedic_name` | string (`object`) | Traditional Ayurvedic or common Indian plant name. | Enriched from `canonical/herbs.tsv` (`Indian_Medicinal_plant`). 0 nulls. |
| `compound_id` | string (`object`) | Canonical compound identifier (`IMPPAT3_PHYIDxxxxxx`) linking herb to target (726 distinct compounds). | Sourced from `canonical/herb_compound_edges.tsv`. 0 nulls. |
| `plant_part` | string (`object`, nullable) | Anatomical plant part yielding the compound (e.g., `leaf`, `flower`, `root`, `aerial part`, `fruit`, `stem`). | Sourced from `canonical/herb_compound_edges.tsv`. Contains 39,402 null values where IMPPAT records omitted anatomical localization. |
| `reference_identifier` | string (`object`) | Bibliographic citation or database record ID documenting the phytochemical extraction (e.g., `CID_5281`). | Preserved from `canonical/herb_compound_edges.tsv`. 0 nulls. |
| `protein_id` | string (`object`) | Canonical protein target identifier (`UNIPROT:{accession}`) (1,341 distinct targets). | Sourced from `compound_target.csv`. 0 nulls. |
| `uniprot_accession` | string (`object`) | Primary UniProt accession of the human target protein. | Preserved from `compound_target.csv`. 0 nulls. |
| `ensembl_gene_id` | string (`object`) | Ensembl gene ID of the target protein. | Preserved from `compound_target.csv`. 0 nulls. |
| `entrez_gene_id` | integer (`int64`) | NCBI Entrez Gene ID of the target protein. | Preserved from `compound_target.csv`. 0 nulls. |
| `hgnc_symbol` | string (`object`) | Official HGNC gene symbol of the target protein. | Preserved from `compound_target.csv`. 0 nulls. |
| `compound_target_label` | integer (`int64`) | Interaction label inherited from the compound–target step (`1`). | Positive association indicator. 0 nulls. |
| `evidence_source` | string (`object`) | Curated bioactivity database source for the compound–target step (`ChEMBL`, `NPASS`, `BindingDB`). | Preserved from `compound_target.csv`. 0 nulls. |
| `herb_compound_evidence_source` | string (`object`) | Source database for the herb–phytochemical relationship. Constant `IMPPAT` across all rows. | Preserved from `canonical/herb_compound_edges.tsv`. 0 nulls. |
| `compound_target_evidence_source` | string (`object`) | Duplicate of `evidence_source` documenting the compound–target step. | Retained for bipartite provenance. 0 nulls. |
| `evidence_type` | string (`object`) | Biological evidence category for the herb–compound association. Constant `plant_phytochemical_association`. | Preserved from `canonical/herb_compound_edges.tsv`. 0 nulls. |
| `n_supporting_records` | integer (`int64`) | Number of supporting records for the compound–target association. | Preserved from `compound_target.csv`. 0 nulls. |
| `source` | string (`object`) | Database source for the compound–target interaction step. | Preserved from `compound_target.csv`. 0 nulls. |
| `has_ecfp4` | boolean (`bool`) | ECFP4 fingerprint availability flag. Exactly `True` for all rows. | Preserved from `compound_target.csv`. 0 nulls. |
| `has_aac` | boolean (`bool`) | AAC protein feature availability flag. Exactly `True` for all rows. | Preserved from `compound_target.csv`. 0 nulls. |
| `fully_featurized` | boolean (`bool`) | Complete featurization indicator. Exactly `True` for all rows. | Preserved from `compound_target.csv`. 0 nulls. |
| `association_label_note` | string (`object`) | Explicit semantic caveat. Exactly `derived_herb_compound_target_path_not_direct_herb_target_evidence` across all rows. | Generated by `build_herb_compound_target.py` (line 83). 0 nulls. |

#### Entity and Relationship Cardinality
- **Total Path Rows**: 126,830
- **Unique Herbs**: 60 (out of 62 canonical herbs; 2 herbs lack compounds with mapped human targets)
- **Unique Intermediate Compounds**: 726
- **Unique Human Targets**: 1,341
- **Unique `(herb_id, protein_id)` Pairs**: 24,835
- **Unique `(herb_id, compound_id, protein_id)` Triples**: 70,716
- **Unique `(herb_id, compound_id, plant_part, protein_id)` Quads**: 126,830 (0 duplicates)

#### Limitations
- **Derived Path Hypothesis**: Composing plant phytochemistry with in vitro bioactivity creates hypothesis paths, not proof of direct in vivo efficacy. In vivo target engagement depends on phytochemical abundance, plant part used, pharmacokinetics, and bioavailability.
- **Missing Plant Parts**: 39,402 rows (31.1%) lack plant part annotations due to omissions in primary literature sources.

---

## 4. Relationships Between Files

The primary dataset files connect through standardized, shared canonical keys:

```
+---------------------------------------------------------------------------------------------------+
|                                      RELATIONAL ARCHITECTURE                                      |
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|     +----------------------------------+                   +----------------------------------+   |
|     |   compound_features_ecfp4.csv    |                   |     target_features_aac.csv      |   |
|     |   - 2,319 canonical compounds    |                   |     - 1,341 human target proteins|   |
|     |   - 2,048 ECFP4 bits             |                   |     - 20 AAC features + name     |   |
|     +----------------------------------+                   +----------------------------------+   |
|                      | [compound_id]                                         | [protein_id]       |
|                      |                                                       |                    |
|                      +------------------------+     +------------------------+                    |
|                      |                        |     |                        |                    |
|                      v                        v     v                        v                    |
|   +------------------------------------+   +------------------------------------+                 |
|   |    compound_target_labeled.csv     |   |   compound_target_unlabeled.csv    |                 |
|   |    - 14,166 unique pairs           |   |   - 3,095,613 candidate pairs      |                 |
|   |    - label=1, IMPPAT_ASSOCIATION   |   |   - label_status=UNLABELED         |                 |
|   +------------------------------------+   +------------------------------------+                 |
|                      | [compound_id]                                                              |
|                      v                                                                            |
|   +------------------------------------+                                                          |
|   |      herb_compound_target.csv      |                                                          |
|   |      - 126,830 paths               |                                                          |
|   |      - 60 herbs -> 726 cpds        |                                                          |
|   |        -> 1,341 targets            |                                                          |
|   +------------------------------------+                                                          |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
```

### Key Join Invariants
1. **Compound Mapping**:
   - `compound_id` (`IMPPAT3_PHYIDxxxxxx`) uniquely joins `compound_features_ecfp4.csv` to `compound_target_labeled.csv`, `compound_target_unlabeled.csv`, and `herb_compound_target.csv`.
   - 100% of compounds in `compound_target_labeled.csv` (726) and `compound_target_unlabeled.csv` (2,319) exist in `compound_features_ecfp4.csv`.
2. **Protein Mapping**:
   - `protein_id` (`UNIPROT:{accession}`) or `uniprot_accession` uniquely joins `target_features_aac.csv` to `compound_target_labeled.csv`, `compound_target_unlabeled.csv`, and `herb_compound_target.csv`.
   - 100% of proteins in the labeled and unlabeled datasets exist in `target_features_aac.csv` (1,341 unique targets).
3. **Herb Mapping**:
   - `herb_id` (`IMPPAT_PLANT:IMPPAT3_PLTIDxxxxxx`) links `herb_compound_target.csv` to canonical herb registries (`canonical/herbs.tsv`).

---

## 5. Data Generation Pipeline

The verified pipeline workflow proceeded through seven sequential stages:

```
[1. Ingest IMPPAT Target Associations] (14,385 raw records)
                         |
                         v
[2. Entrez -> Swiss-Prot Resolution] (Taxonomy filter: 9606 Homo sapiens)
                         |
                         v
[3. Deduplicate Pairs] -> 14,166 unique pairs (1 raw duplicate collapsed)
                         |
       +-----------------+-----------------+
       |                                   |
       v                                   v
[4. Fetch UniProt FASTA]          [5. Generate ECFP4 Fingerprints]
   (1,341 sequences)                 (RDKit Morgan r=2, 2,048 bits)
       |                                   |
       v                                   v
[Calculate 20 AAC Frequencies]    [Full Universe: 2,319 compounds]
       |                          [compound_features_ecfp4.csv]
       v
[Fetch Official Protein Names]
       |
       v
[target_features_aac.csv]
   (1,341 rows x 23 cols)
       |
       +-----------------+-----------------+
       |                                   |
       v                                   v
[6. Build Labeled Table]          [7. Build Unlabeled Universe & HCT Paths]
  (14,166 rows x 15 cols)            - Anti-join against Cartesian product
  [compound_target_labeled.csv]        -> compound_target_unlabeled.csv (3,095,613)
                                     - Inner join with herb_compound_edges.tsv
                                       -> herb_compound_target.csv (126,830)
```

1. **Identifier Resolution**: Raw IMPPAT records were evaluated against `entrez_uniprot_crosswalk.tsv` under strict human taxonomy validation (`taxid == 9606`). Records resolving to a single reviewed Swiss-Prot entry were retained (14,167 rows); ambiguous multi-reviewed entries (198) and entries without Swiss-Prot candidates (20) were quarantined.
2. **Deduplication**: 14,167 resolved records were grouped by `(compound_id, protein_id)`. One duplicate was merged, yielding 14,166 unique pairs with aggregated source strings (`BindingDB`, `ChEMBL`, `NPASS`).
3. **Target Featurization**: Full sequences were downloaded from UniProtKB via HTTPS; 20 normalized standard residue frequencies were computed ($\sum=1.0$). Official recommended protein names were fetched via the UniProt REST API.
4. **Compound Featurization**: 2,048-bit Morgan circular fingerprints ($r=2$) were computed across all 2,319 canonical compounds.
5. **Path Construction**: Canonical herb–compound edges were inner-joined with compound–target pairs on `compound_id`, enriching paths with plant part localizations and Ayurvedic taxonomy.
6. **Labeled & Unlabeled Synthesis**: `compound_target_labeled.csv` was generated as the primary positive table. The candidate universe ($2,319 \times 1,341 = 3,109,779$) was generated in chunks, and an anti-join subtracted all 14,166 labeled pairs to produce `compound_target_unlabeled.csv` ($3,095,613$ rows).

---

## 6. Validation Summary

| Validation Check | Metric / Condition | Result | Status |
|:---|:---|:---|:---|
| **Disjointness** | $\text{Labeled} \cap \text{Unlabeled}$ | Exactly 0 overlapping pairs | **PASS** |
| **Universe Identity** | $\text{Labeled} (14,166) + \text{Unlabeled} (3,095,613)$ | Exactly 3,109,779 ($2,319 \times 1,341$) | **PASS** |
| **Labeled Key Uniqueness** | Duplicates in `(compound_id, protein_id)` | Exactly 0 duplicate pairs | **PASS** |
| **Unlabeled Key Uniqueness** | Duplicates in `(compound_id, protein_id)` | Exactly 0 duplicate pairs | **PASS** |
| **Compound ID Validity** | All compounds exist in `compound_features_ecfp4.csv` | 2,319 / 2,319 (100% valid) | **PASS** |
| **Protein ID Validity** | All proteins exist in `target_features_aac.csv` | 1,341 / 1,341 (100% valid) | **PASS** |
| **ECFP4 Dimensionality** | Column count in `compound_features_ecfp4.csv` | 1 ID + 2,048 bits = 2,049 columns | **PASS** |
| **AAC Normalization** | $\sum_{i=1}^{20} \text{aac}_i$ for every protein row | $1.0 \pm 10^{-6}$ for all 1,341 rows | **PASS** |
| **Protein Name Coverage** | Populated `protein_name` in `target_features_aac.csv` | 1,341 / 1,341 (100% resolved, 0 empty) | **PASS** |
| **Positive Label Consistency** | `label` values in `compound_target_labeled.csv` | Exactly 14,166 rows with `label == 1` | **PASS** |
| **Semantic Labels** | `label_status` assignment | Labeled = `IMPPAT_ASSOCIATION`; Unlabeled = `UNLABELED` | **PASS** |
| **HCT Path Consistency** | Herb, compound, and target key validity | 100% valid canonical IDs; 0 missing joins | **PASS** |

---

## 7. Limitations and Interpretation

1. **Curated Association vs. Binding Affinity**: Recorded positive interactions reflect literature-curated associations across varying assay types ($K_i$, $IC_{50}$, percent inhibition). They denote recorded bioactivity, not uniform binding potencies.
2. **Unlabeled Candidates $\neq$ Inactive Non-binders**: Unlabeled pairs are simply unmeasured in curated databases. Treating them as hard negatives ($y=0$) induces false-negative noise. Practitioners should apply Positive-Unlabeled (PU) learning or ranking models.
3. **Multi-Hop Hypotheses vs. In Vivo Action**: Herb–compound–target triples are compositional graph paths ($\text{Herb} \rightarrow \text{Compound} \rightarrow \text{Target}$). They serve as candidate hypotheses; clinical target modulation depends on compound concentration, bioavailability, and extraction method.
4. **Feature Model Boundaries**:
   - **ECFP4**: 2D circular topology; blind to 3D conformations and stereochemical configurations.
   - **AAC**: Global amino acid frequencies; omits linear motif ordering and 3D binding pocket geometry.
