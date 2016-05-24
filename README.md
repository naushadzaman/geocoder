# geocoder

> A geocoder that relies on offline TIGER/Line data useful for geocoding private health information.

## Installation

This software was designed and tested only on Linux Ubuntu.

### Requirements

Install required software:

	sudo apt-get install sqlite3 libsqlite3-dev flex ruby-full ruby-rubyforge
        
Install ruby gems:

	sudo gem install sqlite3
	sudo gem install json
	sudo gem install Text
        
Install R:

	sudo sh -c 'echo "deb http://cran.rstudio.com/bin/linux/ubuntu trusty/" >> /etc/apt/sources.list'
	sudo apt-get update
	sudo apt-get install r-base-core
        
    sudo sh -c 'echo "deb http://mirrors.ocf.berkeley.edu/ubuntu/ trusty-backports main restricted universe" >> /etc/apt/sources.list'
	sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E084DAB9
	sudo apt-get update
	sudo apt-get install r-base-dev

Install R packages:

	sudo su - -c "R -e \"install.packages('devtools', repos='https://cran.rstudio.com/')\""
	sudo su - -c "R -e \"devtools::install_github('cole-brokamp/CB')\""
	sudo su - -c "R -e \"install.packages('argparser', repos='https://cran.rstudio.com/')\""

### Download and Install

Download the git repo to the home directory and then compile the SQLite3 extension and the Geocoder-US Ruby gem:
	
    cd ~
	git clone https://github.com/cole-brokamp/geocoder
    cd geocoder
    sudo make install
    sudo gem install Geocoder-US-2.0.4.gem

### TIGER/Line Database

The program relies on a sqlite3 database created from TIGER/Line files that is about 4.6 GB. Download the compiled database based on 2015 TIGER/Line files into the `/opt` directory so it is accessible by all users.

	sudo wget https://colebrokamp-dropbox.s3.amazonaws.com/geocoder.db -P /opt


Alternatively, build your own database (see the section below for details).


## Usage

#### Geocoding

The program takes in one character string and parses address components in order to search the database.  To geocode an address, call ruby to run the program with the address string as the argument:

	ruby ~/geocoder/bin/geocode.rb "3333 Burnet Ave Cincinnati Ohio 45229"
    
This results in a file called `temp.json` being written to the current directory with the results. It is possible to pipe this output into another file, but the user will likely be geocoding more than one file at a time, using batch geocoding.

The `geocode.R` program uses the `argparser` package to take in command line arguments which define both the name of the CSV file and name of the column in that file which contain the address strings.

#### Output Format

- `address`: address string input to the geocoder
- `lat`: estimated latitude coordinate
- `lon`: estimated longitude coordinate
- `fips_county`: estimated county; defined by FIPS id number
- `score`: ranges from 0 to 1; higher means better match to records in database
- `precision`: either `city`, `intersection`, `range`, `street`, or `zip`; denotes the method used to geocode
- `street`, `zip`, `city`, `state`, `prenum`, `number`, `street1`, `street2`: the address fields found based on the input address string

#### Submission for Batch Geocoding

For batch geocoding, run `geocoder/bin/geocode.R`, which relies on `Rscript` to run the R program from the command line with arguments.  The R program serves as a wrapper to read the file, iterate over the address strings, output a progress bar, and write the results file as a CSV.

The first argument defines the name of the CSV file and the second argument defines the name of the column in that file which contains the address strings. 

Don't forget to `chmod` this file and optionally, symmlink it somewhere or add it to your path.  Run the program without any arguments for help:

	  > ./geocode.R 
	usage: ./geocode.R [--] [--help] file_name column_name

	offline geocoding, returns the input file with geocodes appended

	positional arguments:
	  file_name			name of input csv file
	  column_name			the name of the column in the csv file that contains the address strings

	flags:
	  -h, --help			show this help message and exit


Test the the program out on some sample addresses that are included in the git repo:

	bin/geocode.R test_addresses.csv address
    
    
The program will output a progress bar to the terminal.  The output will be merged to the original input file and written as a CSV file with `_geocoded` appended to the end of the file name. Address fields not used for an address string will be `NA`. 
    
#### Note on Quality

More work has to be done in order to use the `score` for quality assurance.  It would be useful to define some thresholds to assign `NA` rather than a wildly incorrect address.  This could also be done using the `method` field.


## Building TIGER/Line Database

Although a compiled database created from 2015 TIGER/Line files is available for download, it is possible to create your own database using alternative years for example.

#### Download TIGER/Line files

	mkdir TIGER2015 && cd TIGER2015
	wget -nd -r -A.zip ftp://ftp2.census.gov/geo/tiger/TIGER2015/ADDR/
	wget -nd -r -A.zip ftp://ftp2.census.gov/geo/tiger/TIGER2015/FEATNAMES/
	wget -nd -r -A.zip ftp://ftp2.census.gov/geo/tiger/TIGER2015/EDGES/

If the download fails, rerun with `-c` option to continue where it left off.
    
#### Unpack each TIGER/Line ZIP into a temp directory and extract/transform/load to build database
	sudo build/tiger_import /opt/geocoder.db TIGER2015

After making the database, it is safe to remove all of the TIGER files

	rm -r TIGER2015

#### Update database

Create ruby metaphones

	sudo bin/rebuild_metaphones /opt/geocoder.db

Construct database indexes

	sudo chmod +x build/build_indexes
	sudo build/build_indexes /opt/geocoder.db

Cluster the database accorindg to indexes, making lookups faster

	sudo chmod +x build/rebuild_cluster
	sudo build/rebuild_cluster /opt/geocoder.db

## R Shiny Application

## TODO

- R Shiny application
	- progress bar for lower number of geocodes
    - alternative batch submission for larger number of geocodes
- Accuracy study with score and method
- CAGIS geocoder implementation for local addresses