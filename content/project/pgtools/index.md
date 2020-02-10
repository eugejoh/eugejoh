---
title: 'Writing tables from R to PostgreSQL'
summary: '`pgtools` is an R package with tools to write data frames as PostgreSQL tables'
tags:
- R
- Databases
date: "2020-02-09T00:00:00Z"

# Optional external URL for project (replaces project detail page).
external_link: ""

image:
  caption: Photo from Microsoft Azure
  focal_point: Smart

links:
- icon: github
  icon_pack: fab
  name: GitHub
  url: https://github.com/eugejoh/pgtools
url_code: ""
url_pdf: ""
url_slides: ""
url_video: ""

# Slides (optional).
#   Associate this project with Markdown slides.
#   Simply enter your slide deck's filename without extension.
#   E.g. `slides = "example-slides"` references `content/slides/example-slides.md`.
#   Otherwise, set `slides = ""`.
slides: ""
---

# Tables from R to PostgreSQL
`pgtools` Is an R package that contains functions to provide a consistent workflow for writing a `data.frame` in R as a PostgreSQL database table. The tools are built around the `DBI` and `RPostgres` packages.

**What this package provides?**

 - Arguments to specify table and variable comments in an efficient manner  
 - Conversion of R column types to PostgreSQL field types
 - Schema specification for writing  
 - Simple genration of a SQL `CREATE TABLE ...` statements  
 - Convenient connection to PostgreSQL with credentials based on `/.Renviron` 
 - Vectorized to accept a `list` of `data.frames` to write to PostgreSQL  

This was created for primary use among the data management team at the [Centre for Global Health Research](http://www.cghr.org/) to standardize and optimize the data storage workflow.

[Link](https://github.com/eugejoh/pgtools) to the GitHub repository.