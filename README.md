# CORD-19

CORD-19 is a corpus of academic papers about COVID-19 and related coronavirus research.  It's curated and maintained by the Semantic Scholar team at the Allen Institute for AI to support text mining and NLP research.  Please read our paper for an in-depth description of how it was created:  https://www.aclweb.org/anthology/2020.nlpcovid19-acl.1/

### Updates

* 2020-07-09 - CORD-19 presented at the NLP-COVID workshop.

### Important notes

*We have performed some data cleaning that is sufficient to fuel most text mining & NLP research efforts.  But we do not intend to provide sufficient cleaning for this data to be usable for directly consuming (reading) papers about COVID-19 or coronaviruses.  There will always be some amount of error, which will make CORD-19 more/less usable for certain applications than others.  We leave it up to the user to make this determination, though please feel free to consult us for recommendations.*

*While CORD-19 was initially released on 2020-03-13, the current schema is defined base on an update on 2020-05-26.  Older versions of CORD-19 will not necessarily adhere to exactly the schema defined in this README.  Please reach out for help on this if working with old CORD-19 versions.*

### Download

Download CORD-19 at https://www.semanticscholar.org/cord19/download.

### Overview

CORD-19 is released **weekly**.  Each version of the corpus is tagged with a datestamp (e.g. `2020-05-26`).  Releases look like:

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

When `cord_19_embeddings.tar.gz` is uncompressed, it is a 769-column CSV file, where the first column is the `cord_uid` and the remaining columns correspond to a 768-dimensional document embedding.  For example:

```
ug7v899j,-2.939983606338501,-6.312200546264648,-1.0459030866622925,5.164162635803223,-0.32564637064933777,-2.507413387298584,1.735608696937561,1.9363566637039185,0.622501015663147,1.5613162517547607,...
```



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


### Questions about CORD-19

#### Why can the same `cord_uid` appear in multiple rows?
This is a very tricky issue, and we have not decided on the best way forward. To explain, let’s take example `cord_uid=hox2xwjg`. Examining their respective rows in the metadata file, we see that they are the same paper, but sent from different sources (Elsevier, PMC). The Elsevier row has DOI and PDF, but the PMC row doesn’t. Furthermore, the PMC ID, publication date, and URL for each of these rows is different.

Technically all of this data is representative of paper `hox2xwjg` so we don’t want to remove any of it. But combining them into one cluster would require a schema change to the data, which would break a lot of people’s code. Hopefully this is not too big an issue because there are only a small percentage of papers affected, but know that this issue exists and we’re debating what’s the best way forward.

#### Why do the PMC JSONs not contain any abstracts, yet the PDF JSONs contain abstracts?

Abstracts in the metadata.csv file are “gold” provided directly from publishers or digital archives. Because PMC is very consistent at providing us “gold” abstracts, we do not bother with parsing the PMC XMLs for abstract text (it’s already in the metadata.csv). As such, the PMC JSONs do not contain abstracts. This is not the case for PDF JSONs. We often obtain PDFs through crawling, and in this manner, we would not have “gold” abstracts provided to us. As such, we still opt to parse the PDF for abstract text, which is why that field exists.

#### Why do the title/authors in the JSON look different from what’s in the metadata file?

The most likely reason is PDF parsing errors. Occasionally, publishers will have different metadata from what is actually displayed on the PDF itself (e.g. slight differences in author names). We encourage users to use fields in the metadata file by default and only fall back on the JSON when it is missing.

#### Why is the JSON missing certain metadata, like publication dates?

The JSONs are only meant for representing the full text of the PDF in a structured, machine-readable format. Many metadata fields like dates and venues don’t commonly appear on the PDF. Please defer to the metadata file for all such fields, since these come from the publishers directly.


#### How do you handle paper objects like tables, figures, equations?
Many papers in CORD-19 include HTML table parses. These table parses are available in the document parse files under `ref_entries` of type table. Note: not *all* tables will have HTML parses. These parses leverage IBM Watson Discovery capabilities (more details can be found in our paper).

Figure images are currently not available. We’re currently looking into how to best support these. As for equations, we do not do anything special here – the symbols are treated as text and should be included in the text blobs.

#### What should we do if both PDF and PMC JSONs exist?  Or if there are multiple PDF JSONs?
We view these as different attempts/views to represent the same paper/document.  Some are going to be higher quality than others.  Treat these are separate representations of the same document – you can choose to use one, both, neither (i.e. just use the metadata fields).  On average, we believe the PMC JSONs are cleaner than the PDF JSONs but that’s not necessarily true. 

#### Why can the same `sha` appear for different `cord_uid`?
Let’s take a look at examples `cord_uid=d9v5xtx7` and `cord_uid=8avkjc84`. They both share PDF `sha=5d0d0bd116976e1412c10a84902894999df4a342`. These are two papers we sourced from Elsevier. If you follow the URLs, you’ll notice that they actually retrieve the same PDF despite different having different DOIs. This is an upstream error from the publisher, which we can’t necessarily do anything about. Hopefully the number of these cases is small.

# Contact

### Mailing list

Subscribe to notifications about CORD-19 at:  https://share.hsforms.com/1cM7MMF68RqCdbBKTcyN7VQ3ioxm

### Email

Please email `lucyw@allenai.org` and `kylel@allenai.org` for any questions or concerns.

### Citing CORD-19

Our paper was accepted to the NLP-COVID workshop at ACL 2020.  See the reviews on OpenReview: https://openreview.net/forum?id=0gLzHrE_t3z.  The paper is available in the ACL Anthology (BibTeX below):  https://www.aclweb.org/anthology/2020.nlpcovid19-acl.1

```
@inproceedings{wang-etal-2020-cord,
    title = "{CORD-19}: The {COVID-19} Open Research Dataset",
    author = "Wang, Lucy Lu  and Lo, Kyle  and Chandrasekhar, Yoganand  and Reas, Russell  and Yang, Jiangjiang  and Burdick, Doug  and Eide, Darrin  and Funk, Kathryn  and Katsis, Yannis  and Kinney, Rodney Michael  and Li, Yunyao  and Liu, Ziyang  and Merrill, William  and Mooney, Paul  and Murdick, Dewey A.  and Rishi, Devvret  and Sheehan, Jerry  and Shen, Zhihong  and Stilson, Brandon  and Wade, Alex D.  and Wang, Kuansan  and Wang, Nancy Xin Ru  and Wilhelm, Christopher  and Xie, Boya  and Raymond, Douglas M.  and Weld, Daniel S.  and Etzioni, Oren  and Kohlmeier, Sebastian",
    booktitle = "Proceedings of the 1st Workshop on {NLP} for {COVID-19} at {ACL} 2020",
    month = jul,
    year = "2020",
    address = "Online",
    publisher = "Association for Computational Linguistics",
    url = "https://www.aclweb.org/anthology/2020.nlpcovid19-acl.1"
}
```


### Projects using CORD-19

This is a Google Sheet tracking [systems and demos](https://docs.google.com/spreadsheets/d/1WKlRwaRBpbBE1umT2DsQ_NL4QqK5D2lO_ab7siGMnHA/edit?usp=sharing) that use CORD-19.  *Projects are listed in random order.*  Our focus here is to collect community efforts that might not be discoverable because systems and demos don't always translate to papers (which we can find via citations of CORD-19). 

Missing yours or incomplete data?  Let us know using this [Google Form](https://forms.gle/s48a9RFoyBxxV9J7A) or [email](#email) us!
