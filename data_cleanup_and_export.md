# Water Quality Data Cleanup
Brian High  
05/08/2015  

Get drinking water data from WA DOH, clean, and convert to CSV format.

## Setup

Load the required packages.


```r
pkgs <- c("knitr", "XLConnect")
for (pkg in pkgs) {
    if (! require(pkg, character.only=T)) { 
        install.packages(pkg, repos="http://cran.fhcrc.org", dependencies=TRUE)
        suppressPackageStartupMessages(library(XLConnect))
    }
}
```

```
## Loading required package: knitr
## Loading required package: XLConnect
## Loading required package: XLConnectJars
## XLConnect 0.2-11 by Mirai Solutions GmbH [aut],
##   Martin Studer [cre],
##   The Apache Software Foundation [ctb, cph] (Apache POI, Apache Commons
##     Codec),
##   Stephen Colebourne [ctb, cph] (Joda-Time Java library)
## http://www.mirai-solutions.com ,
## http://miraisolutions.wordpress.com
```

Configure `knitr` options.


```r
opts_chunk$set(tidy=FALSE, cache=FALSE)
```

Create the data folder if needed.


```r
datadir <- "data"
dir.create(file.path(datadir), showWarnings=FALSE, recursive=TRUE)
```

## Water Systems and Sources Data

Get the water systems and sources data from WA DOH [Water System Data for Download](http://www.doh.wa.gov/DataandStatisticalReports/EnvironmentalHealth/DrinkingWaterSystemData/DataDownload) page (updated 2015-02-09).


```r
# Link title format: Group (A|B) (general|source) data (TXT, n KB)
urlbase='http://www.doh.wa.gov/portals/1/documents/4200/'
for (type in c('general','source')) {
    for (grp in c('a', 'b')) {
        datafile <- paste(c(grp, type, '.txt'), sep='', collapse='')
        outdatafile <- paste(c(datadir, '/', grp, type, '.txt'), 
                             sep='', collapse='')
        dataurl <- paste(c(urlbase, datafile), sep='', collapse='')
        if (! file.exists(datafile)) {
            print("Downloading data file...")
            download.file(dataurl, outdatafile)
        }
    }
}
```

```
## [1] "Downloading data file..."
## [1] "Downloading data file..."
## [1] "Downloading data file..."
## [1] "Downloading data file..."
```

### Import Data

Import the data from text files into `data.frame`s.


```r
tsv_import <- function(filename) {
    read.delim(paste(c(datadir, '/', filename), sep='', collapse=''), 
    stringsAsFactors=FALSE, header=TRUE)
}

systema <- tsv_import('ageneral.txt')
systemb <- tsv_import('bgeneral.txt')
systems <- rbind(systema, systemb)
sourcea <- tsv_import('asource.txt')
sourceb <- tsv_import('bsource.txt')
sources <- rbind(sourcea, sourceb)
```

### Convert Character Encoding

Use `sapply` and `iconv` to convert character encoding encoding from latin1 to 
UTF-8.


```r
systems = as.data.frame(sapply(systems, 
                               function(x) iconv(x, "latin1", "UTF-8")), 
                        stringsAsFactors=FALSE)

sources = as.data.frame(sapply(sources, 
                               function(x) iconv(x, "latin1", "UTF-8")), 
                        stringsAsFactors=FALSE)
```

### Convert Case

Convert character values to upper-case.


```r
systems = as.data.frame(sapply(systems, toupper), stringsAsFactors=FALSE)
sources <- as.data.frame(sapply(sources, toupper), stringsAsFactors=FALSE)
```

### Shorten Zip Codes

Shorten Zip Codes to 5 digits.


```r
systems$WSZipCode <- sapply(systems$WSZipCode, 
                            function(x) return(substr(x, start=1, stop=5)))
```

### Fix Inconsistent City Names

Fix inconsistencies in PWSCity column.


```r
systems$PWSCity[systems$PWSCity == "GREEN WATER"]  <- "GREENWATER"
systems$PWSCity[systems$PWSCity == "GOLDBAR"]      <- "GOLD BAR"
systems$PWSCity[systems$PWSCity == "SILVERLAKE"]   <- "SILVER LAKE"
systems$PWSCity[systems$PWSCity == "HOODS PORT"]   <- "HOODSPORT"
systems$PWSCity[systems$PWSCity == "BATTLEGROUND"] <- "BATTLE GROUND"
systems$PWSCity[systems$PWSCity == "EAST SOUND"]   <- "EASTSOUND"
systems$PWSCity[systems$PWSCity == "NORTHBEND"]    <- "NORTH BEND"
systems$PWSCity[systems$PWSCity == "NEWCASTLE"]    <- "NEW CASTLE"
systems$PWSCity[systems$PWSCity == "LITTLEROCK"]   <- "LITTLE ROCK"
systems$PWSCity[systems$PWSCity == "LACENTER"]     <- "LA CENTER"
systems$PWSCity[systems$PWSCity == "SEATTTLE"]     <- "SEATTLE"
systems$PWSCity[systems$PWSCity == "LACONNER"]     <- "LA CONNER"
```

### Remove Thousands Separator

Remove commas from numeric columns (used for "thousands" separator).


```r
remove_commas <- function(x) {
    as.numeric(gsub(",", "", x))
}

numeric_columns <- c("ResPop", "ResConn", "TotalConn", "ApprovSvcs")
systems[numeric_columns] <- lapply(systems[numeric_columns], 
                                   function(x) remove_commas(x))
```

### Replace Slashes

Replace back-slashes with forward-slashes in source name so it will not appear 
like an escape.



```r
sources$Src_Name <- gsub("([\\])","/", sources$Src_Name)
```

### Convert Date Format

Convert dates to YYYY-MM-DD format.


```r
# Note: May have to convert in SQL after import, e.g.:
# SELECT "PWSID","SystemName","Group","County","OwnerTypeDesc","ResPop",
#        "ResConn","TotalConn","ApprovSvcs", 
#        CAST([EffectiveDate] AS DATE) AS EffectiveDate,
#        "PWSAddress1","PWSCity","WSState","WSZipCode" 
#        FROM [high@washington.edu].[table_wa_doh_dw_systems.tsv]
systems$EffectiveDate <- as.Date(systems$EffectiveDate, "%m/%d/%Y")
sources$Src_EffectieDate <- as.Date(sources$Src_EffectieDate, "%m/%d/%Y")
sources$SRC_InactiveDate <- as.Date(sources$SRC_InactiveDate, "%m/%d/%Y")
```

### Define Export Functions

These helper functions will allow for cleaner, reusable export code. They create 
a CSV for import into phpMyAdmin and a TSV for import into SQLShare. The CSV is 
zipped to work around a file size limit with phpMyAdmin.


```r
# Function: Export to CSV and ZIP for phpMyAdmin (MySQL) import
ex_csv_zip <- function(df, filename) {        
    # Write data to CSV, using \ to escape " (not per RFC 4180!) using 
    # qmethod="escape" so that phpMyAdmin can import the CSV correctly.
    write.table(df, file = filename, fileEncoding="UTF-8", 
                row.names=FALSE, quote=TRUE, sep=",", qmethod="escape")
    
    # Zip the CSV so that it will not exceed phpMyAdmin's upload size limit
    zip(paste(c(filename, '.zip'), sep='', collapse=''), filename)
}

# Function: Export to TSV for SQLShare import
ex_tsv <- function(df, filename) {        
    # Write data to TSV, using \ to escape
    write.table(df, file = filename, fileEncoding="UTF-8", 
                row.names=FALSE, quote=FALSE, sep="\t", qmethod="escape")
}
```

### Export Systems and Sources data

We may use the CSV with phpMyAdmin or the TSV with SQLShare, so we will export 
both types of output.


```r
filename_prefix <- 'wa_doh_dw_'

# Zipped CSV for pypMyAdmin
ex_csv_zip(systems, paste(c(datadir, '/', filename_prefix, 'systems.csv'), 
                          sep='', collapse=''))
ex_csv_zip(sources, paste(c(datadir, '/', filename_prefix, 'sources.csv'), 
                          sep='', collapse=''))

# TSV for SQLShare
ex_tsv(systems, paste(c(datadir, '/', filename_prefix, 'systems.tsv'), 
                      sep='', collapse=''))
ex_tsv(sources, paste(c(datadir, '/', filename_prefix, 'sources.tsv'), 
                      sep='', collapse=''))
```

## Cleanup Fluoride Data

Get the fluoride data (XLSX) from WA DOH [Fluoride in Drinking Water](http://www.doh.wa.gov/DataandStatisticalReports/EnvironmentalHealth/DrinkingWaterSystemData/FluorideinDrinkingWater) page (Excel, 06/13).


```r
# Link title: Lists of Water Systems with fluoride (Excel, 06/13)
datafile <- 'fluoride-data.xlsx'
datafileout <- paste(c(datadir, '/', datafile), sep='', collapse='')
dataurl <- paste(c(urlbase, datafile),sep='', collapse='')

if (! file.exists(datafile)) {
    print("Downloading data file...")
    download.file(dataurl, datafileout)
}
```

```
## [1] "Downloading data file..."
```

### Import Worksheet

We need to get sheet number four, excluding the first few rows.


```r
# Import worksheet
library(XLConnect)
fluoride <- readWorksheetFromFile(datafileout, sheet=4, header=FALSE,
                                  startRow=3, endRow=420)
```

### Apply Character Encoding

Use `sapply` and `iconv` to convert character encoding encoding from latin1 to 
UTF-8. This will make the resulting output files larger since more bytes will 
be used per character. The benefit is that we "normalizing" on a common 
character set. The UTF-8 character set is the default for import into phpMyAdmin.


```r
fluoride = as.data.frame(sapply(fluoride, 
                               function(x) iconv(x, "latin1", "UTF-8")), 
                        stringsAsFactors=FALSE)
```

### Specify Column Names

Since we imported the worksheet ignoring the header, we will need to specify the 
column names manually.


```r
colnames(fluoride) <- c("County", "PwsID", "Pws_SystemName", "CllctDate", 
                        "mgL", "ResPop")
```

### Convert to Upper-Case

Convert character values to upper-case.


```r
fluoride <- as.data.frame(sapply(fluoride, toupper), stringsAsFactors=FALSE)
```

### Fix Merged Cell

Fix the merged cell found in last row, from an error in original XLSX worksheet.


```r
fluoride[418, 3] <- 'YAK CO - RAPTOR LANE WATER'
fluoride[418, 5] <- '0.6'
fluoride[418, 6] <- '0'
```

### Convert Numeric Columns

Convert numeric columns to the numeric data type.


```r
numeric_columns <- c("mgL","ResPop")
fluoride[numeric_columns] <- lapply(fluoride[numeric_columns], 
                                    function(x) as.numeric(x))
```

### Export Fluoride

We may use the CSV with phpMyAdmin or the TSV with SQLShare, so we will export 
both types of output.


```r
# Export data to Zipped CSV for import into phpMyAdmin
ex_csv_zip(fluoride, paste(c(datadir, '/', filename_prefix, 'fluoride.csv'), 
                           sep='', collapse=''))

# Export data to TSV for import into UW SQLShare
ex_tsv(fluoride, paste(c(datadir, '/', filename_prefix, 'fluoride.tsv'), 
                       sep='', collapse=''))
```