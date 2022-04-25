# DB Creation

This is the collection of steps to generate the non-mysql databases: neo4j, fireworks, diagrams, solr, and analysisXX.bin.

## Set java 1.8 to be default
1. All of these java programs (neo4j, reactome) will not work with newer versions of java. Set 1.8 to be default:
  * `sudo zypper in java-1_8_0-openjdk`
    `sudo update-alternatives --config java`
      * Choose the 1.8 version

## Convert MySQL db to neo4j graph db

#### Copy the last test_slice_oryza_sativa_XX_ortho_osat_all msyqldump to gk_current

Now that the orthoinference process is all done, we want to copy the last dump from that to a database named gk_current
1. Drop and create empty gk_current
  * In MySQL
      * `drop database gk_current;`
      `create database gk_current;`
2. Fill in with data
  *   ```
cd ~/plant_reactome/rZZ/mysqldbs
zcat test_slice_oryza_sativa_XX_ortho_osat_all_YY.sql.gz | mysql -u user -p gk_current
```
where YY is the last runOrthoinference_YY script (last mysqldump).

#### Install neo4j (if not already installed)
1. add neo4j repo
  * `sudo zypper ar -f https://yum.neo4j.com/stable neo4j-repo`
2. We need a specific version of neo4j, install with:
  * `sudo zypper in -f neo4j-3.4.6-1.noarch`
    * This will also install cypher-shell
  * Disable the repo so newer versions won't be installed
    * `zypper lr | grep neo4j`
    * Make note of the number at the beginning of the output, then
      * `sudo zypper mr -d X`
3. Set initial admin password
  * `sudo neo4j-admin set-initial-password <password>`
  * this messes up the permissions for the neo4j folders, fix with:
    * `sudo chown -R neo4j: /var/lib/neo4`
4. Start and enable neo4j
  ```
  sudo systemctl start neo4j
  sudo systemctl enable neo4j
  ```
  * I had an issue on opensuse Tumbleweed when trying to enable neo4j. Issue is missing rcX.d folders. Can get past by doing:
  ```
  sudo mkdir /etc/init.d/rc2.d
  sudo mkdir /etc/init.d/rc3.d
  sudo mkdir /etc/init.d/rc4.d
  sudo mkdir /etc/init.d/rc5.d
  sudo systemctl enable neo4j
  ```
5. Make sure neo4j is working
  * `sudo systemctl status neo4j`
    * It's working if the last line says "Remote interface available at http://localhost:7474/"

#### Run the script to generate the neo4j db
1. Don't try to build the script, use the old version
2. An existing copy of the plant reactome graph.db needs to be in place if this is a fresh neo4j install
  * ```
  sudo systemctl stop neo4j
  cd /var/lib/neo4j/data/databases/
  sudo mv graph.db graph.org
  sudo tar zxvf ~/plant_reactome/rXX/release_files/graph.db_rXX.tgz
  ```
3. Run the BatchImporter script (neo4j does not need to be running)
```
cd ~/github/PlantReactome/graph-importer
sudo /usr/lib64/jvm/jre-1.8.0/bin/java -jar ./target/BatchImporter-jar-with-dependencies.jar -h localhost -s 3306 -d gk_current -u react-app-user -p react-app-user_pw -n /var/lib/neo4j/data/databases/graph.db -S 186860
```
  * Needs to be java 1.8, will error with missing classes if using higher version.
  * Note that react-app-user and react-app-user_pw have to be working username/password for mysql. Password must be given in command, it won't ask and will fail if not included.
  * A bunch of NPEs will occur at the beginning up to about 5% complete, they are normal and can be ignored.
4. Fix permissions for graph.db and make backup
  * ```
  sudo chown -R neo4j: /var/lib/neo4j/data/databases/graph.db
  mkdir ~/plant_reactome/rXX/release_files
  cd /var/lib/neo4j/data/databases/
  tar zcvf ~/plant_reactome/rXX/release_files/graph.db_rXX_before_schema.tgz graph.db
  ```
5. Restart neo4j
  * `sudo systemctl start neo4j`

6. **Critical** Fix the neo4j simpleLabel -> schemaClass
  * Connect using cypher-shell:
   `echo "MATCH (c:DatabaseObject) WHERE c.schemaClass IS NULL SET c.schemaClass = c.simpleLabel REMOVE c.simpleLabel;" | cypher-shell -u neo4j -p <password>`
    Have to restart neo4j to pick up the fix
    `sudo systemctl restart neo4j`
7. Backup graph.db
  ```
  cd /var/lib/neo4j/data/databases/
  tar zcvf ~/plant_reactome/r65/release_files/graph.db_r65.tgz graph.db
  ```

## Fireworks generation
Another set of files that should be copied, not built. Built versions will not work for some reason. I have copied the good one to our github repo, so should be fine with:
```
cd ~/github/PlantReactome/
git clone https://github.com/PlantReactome/fireworks-layout
```
if not already done.

* Run the fireworks generation and copy to release files:
```
cd ~/github/PlantReactome/fireworks-layout
java -jar ./prebuilt/fireworks-jar-with-dependencies.jar -u neo4j -k react-app-user_pw -f ./config -o ./output_22 -v
cd output_22
tar zcvf ~/plant_reactome/rXX/release_files/fireworks.tgz ./*
```

## Diagrams generation
Another github repo with a prebuilt jar file that should work.
```
cd ~/github.com/PlantReactome
git clone https://github.com/PlantReactome/diagram-Core
```
if not already done.

* Run the diagrams generation and copy to release files:
```
cd ~/github/PlantReactome/diagram-Core
mkdir ~/plant_reactome/rXX/diagrams
java -jar ./prebuilt/tools-jar-with-dependencies.jar Convert -d gk_current -u react-app-user -p <password> -o ~/plant_reactome/rXX/diagrams -r src/main/resources/trivialchemicals.txt
cd ~/plant_reactome/rXX/diagrams
find . -print | tar czvf ../release_files/diagrams_rXX.tgz -T -
```
The java command above will take about an hour, depending on computer specs. I did get an error:
```
org.gk.schema.InvalidAttributeException: Invalid attribute 'physicalEntity' for class 'EntityFunctionalStatus'.
        at org.gk.schema.GKSchemaClass.isValidAttributeOrThrow(GKSchemaClass.java:195)
        at org.gk.schema.GKSchemaClass.getAttribute(GKSchemaClass.java:42)
        at org.gk.model.GKInstance.getAttributeValue(GKInstance.java:239)
        at org.reactome.server.diagram.converter.graph.output.EventNode.getRegulators(EventNode.java:127)
        at org.reactome.server.diagram.converter.graph.output.EventNode.<init>(EventNode.java:36)
        at org.reactome.server.diagram.converter.graph.DiagramGraphFactory.getOrCreate(DiagramGraphFactory.java:217)
        at org.reactome.server.diagram.converter.graph.DiagramGraphFactory.getEventNodes(DiagramGraphFactory.java:200)
        at org.reactome.server.diagram.converter.graph.DiagramGraphFactory.getGraphEdges(DiagramGraphFactory.java:138)
        at org.reactome.server.diagram.converter.graph.DiagramGraphFactory.getGraph(DiagramGraphFactory.java:46)
        at org.reactome.server.diagram.converter.tools.Convertor2JsonTool.convert(Convertor2JsonTool.java:195)
        at org.reactome.server.diagram.converter.tools.Convertor2JsonTool.main(Convertor2JsonTool.java:134)
        at org.reactome.server.diagram.converter.Main.main(Main.java:57)
```
Not sure what that is, but may explain why a couple diagrams appear to be missing (9623703).

Also note that tar directly to create the tarball will not work because there are too many files and get an error "Argument list too long", so have to use `find` to pipe it to the tar command instead.

## SOLR db Creation
#### Install solr (if not already installed)
  1. Download the last solr 6 version (newer versions are not tested)
    * `wget https://archive.apache.org/dist/lucene/solr/6.6.6/solr-6.6.6.tgz`
  2. Install:
    ```
    tar zxvf solr-6.6.6.tgz
    cd solr-6.6.6
    sudo bin/install_solr_service.sh ../solr-6.6.6.tgz
    ```
    * Will get a message about chkconfig, ignore
  3. Add username and password for SOLR
    * `sudo vim /opt/solr-6.6.6/server/etc/realm.properties`
    * Add the following text to the file:
      * `solr_admin: <password>,solr-admin`
      where `<password>` is the actual password

  4. Fix java detection and set it to start
    * If java > 10 is used, the java version detection is broken and some VM options won't be recognized. Fix with the following changes to /opt/solr/bin/solr:
    ```
    justin@work:~/Downloads/solr-6.6.6> diff bin/solr /opt/solr/bin/solr
53c53
< JAVA_VER_REQ="1.8"
\---
> JAVA_VER_REQ="8"
141,142c141,142
<   JAVA_VER_NUM=$(echo $JAVA_VER | head -1 | awk -F '"' '/version/ {print $2}')
<   if [[ "$JAVA_VER_NUM" < "$JAVA_VER_REQ" ]] ; then
\---
>   JAVA_VER_NUM=$(echo $JAVA_VER | head -1 | awk -F '"' '/version/ {print $2}' | sed -e's/^1\.//' | sed -e's/[._-].*$//')
>   if [[ "$JAVA_VER_NUM" -lt "$JAVA_VER_REQ" ]] ; then
1817a1818
>       '-XX:+IgnoreUnrecognizedVMOptions' \    
    ```
    ```
    sudo su -
    cd /var/solr/data
    tar zxvf /path/to/previous/release_files/plantreacatome_solr_rYY.tgz
    systemctl daemon-reload
    systemctl start solr
    systemctl enable solr
    ```
    Check the systemctl status to make sure it started fine. Also go to localhost:8983 and check the "Core Admin" to make sure the plantreactome core is loaded.

#### Run the SOLR indexing
  1. If not already git cloned
  ```
  cd ~/github/PlantReactome/
  git clone https://github.com/PlantReactome/search-indexer
  ```
  2. This step should take about half an hour
  ```
  cd ~/github/PlantReactome/search-indexer
  java -jar prebuilt/Indexer-jar-with-dependencies.jar -a localhost -b 7474 -c neo4j -d <neo4j_password> -e http://localhost:8983/solr/plantreactome -f solr_admin -g <solr_password> -h content_service-interactors/interactors.db
  ```

  3. Backup SOLR db to release_files
    * Have to be root
   ```
    sudo su -
    cd /var/solr/data/
    tar zcvf /home/justin/plant_reactome/rXX/release_files/plantreactome_solr_rXX.tgz plantreactome
    exit
    sudo chown justin: ~/plant_reactome/rXX/release_files/plantreactome_solr_rXX.tgz
    ```

#### Generate the analysis_vXX.bin file
1. If not already git cloned
```
cd ~/github/PlantReactome
git clone https://github.com/PlantReactome/AnalysisTools
```

2. This will take about 30 minutes on my new computer, 3 hours on old.
```
cd Core/
java -jar prebuilt/tools-jar-with-dependencies.jar BUILD -d gk_current -h localhost -u react-app-user -p <mysql_password> --interactors-database-path ../../search-indexer/content_service-interactors/interactors.db -o ~/plant_reactome/rXX/release_files/analysis_vYY.bin -v`
  ```
