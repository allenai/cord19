# CORD-19

CORD-19 is a corpus of academic papers about COVID-19 and related coronavirus research.  It's curated and maintained by the Semantic Scholar team at the Allen Institute for AI to support text mining and NLP research.

*While CORD-19 was initially released on 2020-03-13, the current schema is defined base on an update on 2020-05-26.  Older versions of CORD-19 will not necessarily adhere to exactly the schema defined in this README.  Please reach out for help on this if working with old CORD-19 versions.*

### Overview

CORD-19 is released **daily**.  Each version of the corpus is tagged with a datestamp (e.g. `2020-05-26`).  Releases look like:

```
|-- 2020-05-26/
    |-- changelog
    |-- cord_19_embeddings.tar.gz
    |-- document_parses.tar.gz
    |-- metadata.csv
|-- 2020-05-27/
|-- ...
```

The files in each version are:

- `changelog`:  A text file summarizing changes between this and the previous version.
- `cord_19_embeddings.tar.gz`:  A collection of precomputed [SPECTER](https://arxiv.org/abs/2004.07180) document embeddings for each CORD-19 paper
- `document_parses.tar.gz`: A collection of JSON files that contain full text parses of a subset of CORD-19 papers
- `metadata.csv`: Metadata for all CORD-19 papers.

When `cord_19_embeddings.tar.gz` is uncompressed, it is a CSV file with two fields:  `(cord_uid, document vector weights)`.

When `document_parses.tar.gz` is uncompressed, it is a directory:

```
|-- document_parses/
    |-- pdf_json/
        |-- 80013c44d7d2d3949096511ad6fa424a2c740813.json
        |-- bfe20b3580e7c539c16ce4b1e424caf917d3be39.json
        |-- ...
    |-- pmc_json/
        |-- PMC7096781.xml.json
        |-- PMC7118448.xml.json
        |-- ...
```

### Example usage

We recommend everyone primarily use `metadata.csv` & augment data when needed with full text in `document_parses/`.  For example, let's say we wanted to collect a bunch of Titles, Abstracts, and Introductions of papers.  In Python, such a script might look like:

```
import csv
import os
import json
from collections import defaultdict

cord_uid_to_text = defaultdict(list)

# open the file
with open('metadata.csv') as f_in:
    reader = csv.DictReader(f_in)
    for row in reader:
    
        # access some metadata
        cord_uid = row['cord_uid']
        title = row['title']
        abstract = row['abstract']
        authors = row['authors'].split('; ')

        # access the full text (if available) for Intro
        introduction = []
        if row['pdf_json_files']:
            for json_path in row['pdf_json_files'].split('; '):
                with open(json_path) as f_json:
                    full_text_dict = json.load(f_json)
                    
                    # grab introduction section from *some* version of the full text
                    for paragraph_dict in full_text_dict['body_text']:
                        paragraph_text = paragraph_dict['text']
                        section_name = paragraph_dict['section']
                        if 'intro' in section_name.lower():
                            introduction.append(paragraph_text)

                    # stop searching other copies of full text if already got introduction
                    if introduction:
                        break

        # save for later usage
        cord_uid_to_text[cord_uid].append({
            'title': title,
            'abstract': abstract,
            'introduction': introduction
        })
```

### `metadata.csv` overview

We recommend everyone work with `metadata.csv` as the starting point.  This file is comma-separated with the following columns:

- `cord_uid`:  A `str`-valued field that assigns a unique identifier to each CORD-19 paper.  This is not necessariy unique per row, which is explained in the FAQs.
- `sha`:  A `List[str]`-valued field that is the SHA1 of all PDFs associated with the CORD-19 paper.  Most papers will have either zero or one value here (since we either have a PDF or we don't), but some papers will have multiple.  For example, the main paper might have supplemental information saved in a separate PDF.  Or we might have two separate PDF copies of the same paper.  If multiple PDFs exist, their SHA1 will be semicolon-separated (e.g. `'4eb6e165ee705e2ae2a24ed2d4e67da42831ff4a; d4f0247db5e916c20eae3f6d772e8572eb828236'`)
- `source_x`:  A `List[str]`-valued field that is the names of sources from which we received this paper.  Also semicolon-separated.  For example, `'ArXiv; Elsevier; PMC; WHO'`.  There should always be at least one source listed.
- `title`:  A `str`-valued field for the paper title
- `doi`: A `str`-valued field for the paper DOI
- `pmcid`: A `str`-valued field for the paper's ID on PubMed Central.  Should begin with `PMC` followed by an integer.
- `pubmed_id`: An `int`-valued field for the paper's ID on PubMed.  
- `license`: A `str`-valued field with the most permissive license we've found associated with this paper.  Possible values include:  `'cc0', 'hybrid-oa', 'els-covid', 'no-cc', 'cc-by-nc-sa', 'cc-by', 'gold-oa', 'biorxiv', 'green-oa', 'bronze-oa', 'cc-by-nc', 'medrxiv', 'cc-by-nd', 'arxiv', 'unk', 'cc-by-sa', 'cc-by-nc-nd'`
- `abstract`: A `str`-valued field for the paper's abstract
- `publish_time`:  A `str`-valued field for the published date of the paper.  This is in `yyyy-mm-dd` format.  Not always accurate as some publishers will denote unknown dates with future dates like `yyyy-12-31`
- `authors`:  A `List[str]`-valued field for the authors of the paper.  Each author name is in `Last, First Middle` format and semicolon-separated.
- `journal`:  A `str`-valued field for the paper journal.  Strings are not normalized (e.g. `BMJ` and `British Medical Journal` can both exist). Empty string if unknown.
- `mag_id`:  Deprecated, but originally an `int`-valued field for the paper as represented in the Microsoft Academic Graph.
- `who_covidence_id`:  A `str`-valued field for the ID assigned by the WHO for this paper.  Format looks like `#72306`. 
- `arxiv_id`:  A `str`-valued field for the arXiv ID of this paper.
- `pdf_json_files`:  A `List[str]`-valued field containing paths from the root of the current data dump version to the parses of the paper PDFs into JSON format.  Multiple paths are semicolon-separated.  Example: `document_parses/pdf_json/4eb6e165ee705e2ae2a24ed2d4e67da42831ff4a.json; document_parses/pdf_json/d4f0247db5e916c20eae3f6d772e8572eb828236.json`
- `pmc_json_files`:  A `List[str]`-valued field.  Same as above, but corresponding to the full text XML files downloaded from PMC, parsed into the same JSON format as above.
- `url`: A `List[str]`-valued field containing all URLs associated with this paper.  Semicolon-separated.
- `s2_id`:  A `str`-valued field containing the Semantic Scholar ID for this paper.  Can be used with the Semantic Scholar API (e.g. `s2_id=9445722` corresponds to `http://api.semanticscholar.org/corpusid:9445722`)


