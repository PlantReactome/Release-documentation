# Data Gathering, Orthologs, Curated Pathways

Instructions for the initial steps of data gathering for a plant reactome release

## Release Documents prep

Get the list of new pathways from curators and add to top of document:
https://docs.google.com/document/d/1fZ9VygHyOdKxfk0ZMJ7Z_6qfA_y11DQjqlfrsTrZP_w/edit

#### Add new species
Get the list of new species and add to a new copy of "Plant Reactome projection list - Slice XX" spreadsheet:
https://docs.google.com/spreadsheets/d/1GB_WdHW_lVEZwkmd1MPijas-iQIAL8qP7ljNNKhv6RE/edit#gid=0
  * When adding, decide on a unique 2 and 4-digit abbreviation. Far as I can tell, the 2 digit is only used in the "incomparanoid_projections_slice_XX.sh" script, but the 4 digit is required for the orthoinference steps.
  * Also add the info about the "Clade". Choices are usually going to be monocot or dicot, but could also be amborella, lycopods, bryophytes, chlorophytes and rhodophytes. Have to google to figure out for each species, or ask.
  * To determine the "Recip % identity", use 40% for monocots, 30% for dicots and amborella, 28% for the others (as described on https://plantreactome.gramene.org/index.php?option=com_content&view=article&id=68&Itemid=270&lang=en)
  * If any species have been added to compara that were previously in InParanoid, change them in the spreadsheet as well.

## Request new species added to reactome
Any new species that are added will have to be added to the main Reactome project.

Send the list of species with taxon IDs to Peter D’Eustachio.

## Request, receive, and validate compara database
Sharon Wei has to receive the latest Compara orthology database dump from Ensembl Plants, and then download the relevant .rtm files and stage them on the Gramene ftp server

#### Download the .rtm files:
```
mkdir -p ~/plant_reactome/r##/orthology_files/compara/
cd ~/plant_reactome/r##/orthology_files/compara/
wget -np -nH --cut-dirs 3 -r http://ftp.gramene.org/collaborators/reactome/G##_reactom_ort_dump/
```
(where ## is the Gramene release number)

## Request curation freeze
Contact the curators (Sushma and Parul) and make sure they aren't curating any more pathways until after the release.

## Download and install gk_central
#### Get fresh copy of gk_central from Reactome Curator server (curator.reactome.org):
```
mkdir ~/plant_reactome/r##/mysqldbs
cd ~/plant_reactome/r##/mysqldbs
ssh -i <path_to_AWS_cert_user>.pem <user>@curator.reactome.org "mysqldump --no-tablespaces --skip-lock-tables -u authortool -p gk_central | gzip -9" > ./gk_central_<mmddyy>.sql.gz
```


#### Load into mysql (create empty db first):
`zcat gk_central_<mmddyy>.sql.gz | mysql -u username -p password -h host gk_central_<mmddyy>`

Next up is [Slicing](slicing.md)
