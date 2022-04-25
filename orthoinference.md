# OrthoInference

## Export Curated gene list (inc RGP)
For now, this can only be ran via IntelliJ, the Java IDE, community version is fine.
![IntelliJ config](https://i.imgur.com/ZWZ0iqF.png)

Above image hopefully shows the configuration needed to run it.

Code is downloaded from github with `git clone https://github.com/PlantReactome/Pathway-Exchange`

The CuratorUtlities.listRiceRGPSs needs to be ran twice, once with the arguments `(true, true)` and again with `(true, false)`. Before running, the db needs to be changed in "CuratorUtilities.xml" to the test_slice_oryza_sativa_XX db saved in the Slicing step. To save directly to a file, in the `Run/Debug Configurations` IntelliJ options (scroll down in the front window in the above image), choose the file to save to with the option `Save console output to file:`.The 2 files should be named differently, but the names don't really matter.

## Generate MSU_RAP-Uniprot mappings; Edit orthopair dupes
This step now has a script. The script is at `Pathway-Exchange/bin/combine_curated_riceRGPs.sh` and has an associated resource file that contains all the old Uniprot IDs to remove. Script is called with
`bin/combine_curated_riceRGPs.sh file_1 file_2 slice_XX`
where file_1 and file_2 are the files created from the 2 runs of listRiceRGPs above, and the `XX` is the slice number. The script will output several files and will check to see if any still have duplicates left. If the script does indicate a dupe is still left, one of the UniProt IDs will have to chosen and the other(s) added to the `resources/RGP_dupes_list.lst` file. **Do not have a blank line in the RGP_dupes_list.lst file**, it will cause it to remove everything. Make sure if any more dupes are added they are committed to the repository. If a dupe was found, the script will need to be reran.

  * Choosing which dupe to drop:
    * Check both UniProt IDs at https://www.uniprot.org/uniprot/B9EUK4 (B9EUK4 in this case) and look for a gold star in the Status bar. Choose the one that doesn't have the gold star to remove, ie, add it to the RGP_dupes_list.lst file.

As long as there are no dupes still left in the file as seen in the script output, the file that is to be used will be named `os_loc_2_os_uniprot_slice_XX_no_dupes.txt`.

## Extract Inparanoid data: validate
This step uses a modified version of the `find_ortho_super.pl` script called `find_ortho_super_PR_current.pl`. Because this script interacts directly with the MySQL database on floret, it will need to be ran from a machine on the OSU campus or it will be blocked by the campus firewall.

### Todo // finish out this section

## Orthopair pre-processing (rename and move ortho file into place)
The script to do this is `move_and_rename_orthology_files.py` and is in the github repo `Release/scripts/release/orthopairs/move_and_rename_orthology_files.py` in the `develop` branch. I modified the script so that it won't error if the directories don't exist.
It should be ran with:


`python3 /path/to/move_and_rename_orthology_files.py -c Plant_Reactome_projection_list-Slice_XX.tsv -e compara -i inparanoid -o rice_to -s XX -r ZZ -v`

where XX is the slice number and ZZ is the ensembl release number. The `-e` option designates the location of the compara .rtm files, while `-i` is the inparanoid files from `find_ortho_super_PR_current.pl` from the previous step. `-o` is the output directory files will be moved to.

If the script errors with a missing file, find the file and move it into place. Some compara files are dumped from previous releases, so grab it from the previous release files. Some may have to be renamed as the script does not do well with more than genus + species sometimes.

## Generate orthopair files and script
The main script used for this is called `incomparanoid_projections_slice_XX.sh` where XX is the current slice number. To generate, grab the previous slice version of this file and modify by adding any new species to the top. The command in each file are one per species, and are different depending on if the orthology data source is Ensembl compara or InParanoid. Also change for any species that were moved from InParanoid to compara.

The script requires 3 files to already exist in specific directories (relative to the script source directory):
`loc_rgp_lists/LOC_RGPs_slice_XX.txt`
`loc_to_rap/RAP-MSU.txt`
`loc_to_uniprot/os_loc_2_os_uniprot_slice_XX_no_dupes.txt`

where XX is the current slice number.  The "LOC_RGPs_slice_XX.txt" and "os_loc_2_os_uniprot_slice_XX_no_dupes.txt" files are to be copied from the LOC_RGPs_slice_XX_expanded_sort_uniq.txt file from the "Generate MSU_RAP-Uniprot mappings" step above. Copy the "RAP-MSU.txt" file from the previous release, this file shouldn't change.

I modified the `incomparanoid_projections_slice_XX.sh` script to accept args for the slice and ensembl release number to make it easier to run, so it should be run with:
`./incomparanoid_projections_slice_XX.sh -s XX -e YY 1> output.txt 2> error.txt`

where `XX` is the slice number, and `YY` the ensembl release number (21 and 48 respectively for the release at time of writing).
  * Check the error.txt for any issues. One issue that came up was "KeyError: ....". This indicates that a uniprot ID in the `src/main/resources/RGP_dupes_list.lst` file is no longer actually a duplicate. If this is the case, remove it from the dupes list file and rerun.

The script will generate 3 files for each species, in a `rice_to/slice_XX/ZZZZ` folder, where `ZZZZ` is the 2-4 char species identifier. These 3 files will be used for the actual orthoinference step.

## Projection config file updates
File to be changed is `PlantReactomeSpecies_full_esc.json`. This is a config file in the `src/main/resources` folder for orthoinference. This was generated using a script, but future changes will probably be made manually. If any new species are to be added, they need to be added to this file. Just copy/paste a similar entry and make your changes.
#### Important: The species 'name' for each species needs to match ***exactly*** how it was added in gk_current.
If it doesn't, there will be issues later and orthoinference will need to be reran again. This caused quite a few headaches for Panicum hallii var hallii and Zea mays ver5 previously. IIRC, the issue will be that multiple species will be added to the gk_current instance, and the fireworks generation will fail with an NPE. Even if you get past that issue, it will cause other issues with generated json files later.

Specific error messages related to incorrect names include:
```
08:46:12.215 [main] DEBUG org.neo4j.ogm.context.GraphEntityMapper - Unable to find property: schemaClass on class: org.reactome.server.graph.domain.model.Species for writing
08:46:12.215 [main] DEBUG org.neo4j.ogm.context.register.EntityRegister - Added object to node registry: 2597, Species {dbId=9659284, displayName='Zoysia japonica'}
Exception in thread "main" java.lang.NullPointerException
       at org.reactome.server.graph.service.SpeciesService.getSpecies_aroundBody0(SpeciesService.java:25)
       at org.reactome.server.graph.service.SpeciesService.getSpecies_aroundBody1$advice(SpeciesService.java:33)
       at org.reactome.server.graph.service.SpeciesService.getSpecies(SpeciesService.java:1)
       at org.reactome.server.fireworks.Main.main(Main.java:63)
```
Note that the above will error for all species, not just the one that is incorrect, making it hard to diagnose where the problem actually is.

## Run ortho-inference
This step is the most time consuming as far as processing, but doesn't take a lot of prep to get started. This step currently takes about 96 hours to complete, and will take longer with each additional species.

To change the database that is being written to, change the file `PlantReactome/data-release-pipeline/orthoinference/src/main/resource/config.properties` so that the db is "test_slice_oryza_sativa_XX_ortho_osat_all". Also change the releaseNumber and dateOfRelease.

Need to copy the test slice database to a new one:
In mysql:
`create database test_slice_oryza_sativa_XX_ortho_osat_all;`
From terminal:
```
cd ~/plant_reactome/rYY/mysqldbs/
mysqldump -u justin -p test_slice_oryza_sativa_XX | gzip > test_slice_oryza_sativa_XX_after_species.sql.gz
zcat test_slice_oryza_sativa_XX_after_species.sql.gz | mysql -u justin -p test_slice_oryza_sativa_XX_ortho_osat_all
```

The orthopair files from the "Generate orthopair files and script" need to be moved in to place.
```
mv src/main/resources/orthopairs src/main/resources/orthopairs_slice_21
mkdir src/main/resources/orthopairs
cp ~/plant_reactome/r64/orthology_data/rice_to/slice_21/*/*mapping.txt ./src/main/resources/orthopairs/
```

This step is broken up in to several sub steps, all of which can be called by the master script `runOrthoinference_master_slice_XX.sh`. This script in turn then calls each of the `runOrthoinference_ZZ.sh` scripts.

Modify the runOrthoinference_master_slice_XX.sh script with the current slice and release numbers and add any new runOrthoinference_ZZ.sh scripts. Note: it looks like the orthoinference script does require java 1.8, get exceptions with 17
```
Exception in thread "main" java.lang.ExceptionInInitializerError
Caused by: java.lang.UnsupportedOperationException: No class provided, and an appropriate one cannot be found.
        at org.apache.logging.log4j.LogManager.callerClass(LogManager.java:555)
        at org.apache.logging.log4j.LogManager.getLogger(LogManager.java:580)
        at org.apache.logging.log4j.LogManager.getLogger(LogManager.java:567)
        at org.reactome.orthoinference.Main.<clinit>(Main.java:11)
```

Call the script with `runOrthoinference_master_slice_XX.sh`.

Previously, it was suggested to add new species to the first runOrthoinference_1.sh script, but I think a better way in the future is to do a quick test run with any new species, make sure it works, then add the new species to the end of the runOrthoinference_XX.sh scripts and start over with a clean db.  This way if there are later problems with the new species when doing the "Data Conversion" steps, we don't have to rerun the entire orthoinference process, just the last step.
**Note:** If species are switched from InParanoid to compara, test them, but make sure they aren't in the runOrthoinference_XX scripts twice. A quick command to test if there are duplicates `egrep "^projSpecies" runOrthoinference_* | awk -F '=' '{print $2}' | sed 's/(//' | sed 's/)//' | tr " " "\n" | sort | uniq -d`

To add new species, simply add the 4 char species identifier to the runOrthoinference_XX.sh file in the "projSpecies=" line. After each runOrthoinference file is ran, it will take a mysqldump of the database so that we have periodic backups to go to in case something goes wrong.

#### One time, there was an NPE when running this, but starting that script after reloading the db to the previous good backup didn't have issues.
So, just need to watch to make sure the output looks good for each.
  * An NPE for inferred events may indicate the 4-digit abbreviation is wrong or missing mapping data.

Each runOrthoinference_ZZ step will take progressively longer. As the database grows, so does the time to enter the data. The first one should take less than an hour, second more than 2 hours...

This should be everything needed for the OrthoInference step of the Plant Reactome release process.
