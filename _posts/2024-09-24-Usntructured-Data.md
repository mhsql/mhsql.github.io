---
layout: post
title: "Is Unstructured Data Just Corrupt Data?"
---

## Overview
With the rise of Big Data, and data provided via files saved in folders, or what we now call <b>Data Lake</b>, we got the terms
- Structured Data
- Semi-Structured Data
- Unstructured Data

Let's take a look at what they meant at the time and how they relate to the world of AI

### Structured Data
This should be straight forward, most of the files are structured at two levels

1. They have a file type extension like .csv or .xlsx, which defines how the <b>binary</b> file data can be read, this is a formal structure
1. The content has a fixed structure, similar to a relational database table, this is a data structure

You can think of this like a single relation database table, with a well defined schema and with blank (or <b>NULL</b>) values allowed

### Semi-Structured Data
This is a logical step on from Structured Data, as it has optional things
<br>It still has a file format and formal structure, just the data structure is more dynamic

i.e. instead of all the files being the same csv file with a 10 column header line and 10 columns per row, they are different Sheets in an Excel Workbook, they could
- have different column headers in each file/sheet
<br>With each file/sheet having a different <em>schema</em>
- a little more dynamic, JSON is used instead of csv, and each row has different columns or even array/object values instead of scalar values
<br>Each column has a <em>key/value pair</em>, the column self defining the itself

I have worked with complex csv file definitions, one had section headers, one for each data group, e.g. one group per item category
<br>Each header being the summary section for the group, e.g. a definition of the category and how many rows to expect
<br>Meaning the file was a combination of 2+ different data structures, so semi-structured

You can think of Semi-Structured Data as groups of relational database tables
<br>They have some common columns and could be combined with a UNION operator, in a view, to allow them to be queried as a single logical table/added to the same file
<br>This is a similar logical structure to a document database, where different documents with independent definitions are stored together for a single subject

Unlike the analogy of the single file with UNIONed tables, which has <em>extra</em> columns with <b>NULL</b> values
<br>Each semi-structured file would be a single table with it's own set of columns, like a single document in a document database

### Unstructured
This one is where I have an issue, as to me it really means a corrupt file
<br>Given, to access the binary file data, you need some form of formal structure, if there is no formal structure, then the file is unusable or corrupt

In the Big Data World, these files are Image, Video, Audio, Database Data Files (the actual file) and other standard *data* files

However, when you think about it, these have a data format too
<br>In the case of a Video file, the picture & sound data need to be sync'd when you're watching the video, and even <em>chapter</em> markers to 'time jump' past the scary scene you don't want to watch

And in the modern AI world of large parameter input models, that data structure is needed to be able to input a 'frame' image and sound to train the model, the audio with a crying face used to identify a happy (cheering sound) or sad (booing sound) emotion

So, I'd still classify these files as Semi-Structured, as a data structure is known and allows the <em>processor</em> to understand the content
<br>If there was no data structure, how can you see what the data represents?
<br>Which is where the question comes in, 'Is Unstructured Data Just Corrupt Data?'

### All Data is Semi-Structured, Even Relational Data
Ok, include Relational Data may sound like a stretch, so lets look at the modern data world
<br>Parque, JSON, XML and yes, I'm skipping a lot of other formats, have the two levels of structure, with a formal definition to get from on disk binary data to a usable data structure
- XML - this is seen as close to a traditional relation table, as it has a formal schema and changes can cause problems
<br>I once worked with an API that provided an undocumented XML schema response, if the backend data had been processed via a support request, and it needed special treatment to handle the <em>extra</em> data that didn't get processed
- JSON - like XML, though more flexible, it has a more dynamic definition of what is contains
<br>JSON is a set of Key\Value pairs, where any value can be a set of Key\Value pairs or a <em>real</em> value, then the document Keys and hierarchy makes it a self describing schema with three data types
    - number - integer or decimal
    - string - anything that's not a number, e.g. dates are strings
    - object - a set or array of Keys, or a list of values
- Parque - a candidate for the modern default CSV format, the data engineer's version of JSON
<br>Parque like JSON is a self describing document, though it can support more data types and compression

Now I did say '<em>Even Relational Data</em>', so what has been added to Relational Databases?
<br>JSON support and <em>polybase</em> or external files

With JSON support, in some cases a trick, a single column can contain dynamic sub-table data
<br>I use this for logging with-in a database, where different stored procedures can share a common log table &amp; providing customer data via  JSON into the same column

While I don't use <em>polybase</em> or External Tables, they now support data import from cloud storage without the need for ETL tools
<br>And even archiving data to a data lake via Parque (or other) files

One final note for '<em>Even Relational Data</em>', even a basic table that support's NULLable columns can be though of as semi-structured, due to different groups of rows having different combinations of column values
<br>e.g. a Product Table can have a size, weight, length &amp; width <em>sizing</em> columns. Depending on the product the <em>sizing</em> column(s) that are relevant can be different, meaning the table is a combination of different sub-product classification with different logical structures

## Summary
At the start of the Big Data era, it felt like a combination of changes suggesting relational data was on the way out and dynamic data was the future
<br>Even daring to suggest that data without any structure was part of this new world
<br>When they meant that data didn't need a rigid and well defined structure, things could be more dynamic

Even the Big was misleading
- a 1 Billion Row table for a physical store's stock wasn't Big Data
- a set of ten 1 Mb files, each with 1,000 rows of data could be Big Data, if it was saved in the <em>data lake</em>

What <b>unstructured</b> really defined, was an option for iterations of data schema definition, which could all be supported at the same time
<br>Allowing a new format to be sent before code could make use of the latest schema to access it
<br>And anything that wasn't alpha numeric data values, like photos and music

If we look at the modern AI landscape, where a million parameter input to an AI model can be used to return
- a list of recommended products a customer is likely to purchase
- sample code to create that new website feature
- video promo of a movie idea

The data is semi-structured, and good data structures are the key to good AI Models
<br>For training, data needs to be understood and <em>stripped</em> into suitable tokens, for analyses
<br>For questions, the <em>prompt</em> needs to be converted into parameters/tokens/vectors that the model can understand

In return the AI provides data responses humans can understand, even if they contain hallucinations or misleading information, the can read a response that may even be translated, English to Spanish
<br>Even Java to Python with suitable training data