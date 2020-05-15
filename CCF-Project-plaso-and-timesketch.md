# CCF Project

## Using modern open-source tools Plaso and Timesketch for forensic investigations of Linux based systems.

Artem Abramov

This project focuses on using two open-source projects focused on forensics for analysing Linux systems:

- Plaso (https://github.com/log2timeline/plaso) - tool to collect and categorise
    information from digital evidence, such as disk images or event logs (rewrite of
    log2timeline).
- Timesketch (https://github.com/google/timesketch) - graphical tool for collaborative
    forensic timeline analysis (can be used as an alternative to Autopsy).




### 1. How suited are Plaso and Timesketch for analysing Linux based servers.

#### a. What options do they provide for analyzing data found on various disks.

Plazo tags the files it finds.

Timesketch also provides a way to organise the whole investigation. It presents the notion of a story.

There are graphs.



#### b. What filters/plugins are there to help with analysis.

There are a number of plugins that should provide a default analysis. E.g. browser history, executable files, PDFs, zip, 

#### c. What metadata is collected and parsed.



#### d. How can Timesketch API be used to export filtered data into another tool for further analysis (or scripted analysis).





### 2. How can this analysis be improved in terms of accuracy, usefulness and speed.

#### a. What plugins are missing.

Analyzers for linux browser history. Plazo does not collect info well.



#### b. What views/graphs are missing.

Heatmap, histogram



#### c. What outstanding bugs / limitations exist.

https://github.com/log2timeline/plaso/issues/2501

https://github.com/google/timesketch/pull/1195



ADD PLUGING TO SEARCH FOR POSSIBLY ENCRYPTED DATA!!! (random data) see https://github.com/antagon/TCHunt-ng



REMOVE LIMITATION ON NUMBER OF EVENTS (i.e. 10000 events then you need to change filters, why can not I just keep going from start to end??) see https://github.com/google/timesketch/issues/921





### 3. What can be done to fix the limitation in the time scope of this CCF project.

#### a. Writing a new plugin/filter.

Write analyser for linux browsers.



#### b. Writing an external python script that would use the Timesketch API to provide desired functionality.



#### c. Actually contributing bug fixes to the Timesketch repository.

When attempting to 



