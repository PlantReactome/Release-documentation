# Migration to new release on server

This document outlines the process to put files on either the dev, live, or cyverse mirror server and move to a new release.

## Copy files to server
Change server location as needed.

The OICR servers are locked down to only accept ssh sessions from OSU IP addresses, so have to use palea as a "Jump Host" to get the files there.

```
cd ~/plant_reactome/rXX/
scp -r -J <user>@palea.cgrb.oregonstate.edu:<port> release_files <user>@plantreactome1.oicr.on.ca:~/rXX_files
```

## Update server with latest data

1. Get to location on server
```
ssh plantreactome1.oicr.on.ca
cd rXX_files
```
2. Stop all related services
  ```
  sudo systemctl stop solr
  sudo systemctl stop apache2
  sudo systemctl stop neo4j
  sudo systemctl stop tomcat
  ```
3. Drop old databases
  ```
  mysql -u root -p
  drop database gk_current
  ```
  If not doing dev, also drop the gk_joomla
  `drop database gk_joomla`
  If doing dev, do the "Website Mgmt" section instead after done with all data migration here

4. Create new empty databases
`create database gk_current`
if not on dev:
`create database gk_joomla`

5. Load new data to mysql
`zcat gk_current_rXX.sql.gz | mysql -u root -p gk_current`
if not on dev:
`zcat gk_joomla_rXX.sql.gz | mysql -u roo t-p gk_joomla`

6. Copy new graph.db for neo4j
```
sudo su -
cd /var/lib/neo4j/data/databases/
mv graph.db graph.db.old
tar zxvf /home/elserj/rXX_files/graph.db_rXX.tgz
chown -R neo4j: graph.db
```

7. Copy new solr data
```
cd /var/solr/data
mv plantreactome ../old_cores/plantreactome.bak
tar zxvf /home/elserj/rXX_files/plantreactome_solr_rXX.tgz
chown -R solr: plantreactome
cd ../old_cores
tar zcvf plantreacatome_solr_rYY.tgz plantreactome.bak (YY is the previous release number)
rm -fr plantreactome.bak
```

8. Put analysis_vXX.bin file in place and switch to it
```
cd /usr/local/reactomes/PlantReactome/production/AnalysisService/input/
cp /home/elserj/rXX_files/analysis_vXX.bin ./
chmod +x analysis_vXX.bin
chown tomcat7: analysis_vXX.bin
vim /etc/apache-tomcat/webapps/AnalysisService/WEB-INF/classes/analysis.properties
```
Edit the analysis.properties file to use the new analysis_vXX.bin file, should just have to change the version number.

9. Update files on download page
```
cd /usr/local/reactomes/PlantReactome/production/Website/download/current/
tar zxvf /home/elserj/rXX_files/data_exports_rXX.tar.gz
mv diagram diagrams_rYY_old
mkdir diagram
cd diagram
tar zxvf /home/elserj/rXX_files/diagrams_rXX.tgz
cp /home/elserj/rXX_files/R-OSA-9623703*.json ./ (if the HSFA7 diagram is still broken)
cd ../
mv fireworks fireworks_rYY_old
mkdir fireworks
cd fireworks
tar zxvf /home/elserj/rXX_files/fireworks.tgz
cd ../
cp /home/elserj/rXX_files/biopax3.zip ./
cp /home/elserj/rXX_files/gene_ids_by_pathway_and_species.tab.gz
gunzip gene_ids_by_pathway_and_species.tab.gz
cp /home/elserj/rXX_files/gk_current_rXX.sql.gz gk_current.sql.gz
cp /home/elserj/rXX_files/graph.db_rXX.tgz reactome.graphdb.tgz
chown -R www-data: ./
```

10. Restart services
```
systemctl start neo4j
systemctl start solr
systemctl start tomcat
systemctl start apache2
```

If on dev, do the "Website Mgmt" section. If on live, may have to schedule the domain name switch with OICR. If do this, make sure to edit the /etc/motd file to reflect which server is which.
