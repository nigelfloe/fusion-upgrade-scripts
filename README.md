**Upgrade notes for upgrading from 1.2.0, 1.2.1 to 1.2.3**.

TODO: Notify customers to have enough disk space. More disk space than what current fusion directory occupies

TODO: Write how customers can download the upgrade scripts ?

Pre-requisites
--------------

1. Install Python 2.7.2+, Install python Kazoo library (https://kazoo.readthedocs.org/en/latest/install.html)
2. `cd` to the base directory of an existing Fusion install. (E.g, if `fusion` is installed at `/opt/lucidworks/fusion`, then do `cd /opt/lucidworks`).
3. All the below commands assume the user is in the base directory of an existing Fusion install. Copy the latest version of fusion to the base directory.
4. Download the upgrade scripts from the Git repo

    ```
    curl -L -o fusion-upgrade-scripts.tar.gz http://github.com/Lucidworks/fusion-upgrade-scripts/tarball/1.2.3/
    mkdir fusion-upgrade-scripts
    tar -C fusion-upgrade-scripts --strip-components=1 -xf fusion-upgrade-scripts.tar.gz
    ```

Steps to Upgrade
----------------

1. Ensure that no administrators are making changes to Fusion config. Querying may continue.
2. Install the new version in a folder next to the old version

    ```
      mkdir fusion-new
      tar -C fusion-new --strip-components=1 -xf fusion-1.2.n.tar.gz (Substitute fusion-1.2.n.tar.gz with the actual tar file with the new Fusion install) 
    ```
3. Stop all services of Fusion except ZK

   ```
      fusion/bin/ui stop
      fusion/bin/connectors stop
      fusion/bin/api stop
   ```

4. cd to the upgrade scripts directory

    ```
       cd fusion-upgrade-scripts
    ```

5. Run the export script against the running zookeeper

    ```
      python zk_export.py {zk_address} {output_filename}
      E.g., python zk_export.py localhost:9983 zk_fusion.export
    ```

6. Run the update script against the exported file

    ```
       python convert_zk_data.py {exported file from previous command} {introspect file shipped with scripts} {new file to save updates}
       E.g., python convert_zk_data.py zk_fusion.export introspect.json zk_fusion_updated.export
    ```

7. Change back to the base directory

    ```
       cd ..
    ```

8. Copy any changes (after old installation) from `fusion/bin/config.sh` to the new config `fusion-new/bin/config.sh`. Be careful and do *NOT* copy the files as it is since the new config file might have been updated. If you haven't changed anything in your `config.sh`, then there is no need to make any changes to the new instance `config.sh`. Also, check your bin scripts for any changes and substitute them in config.sh
    * E.g., 1.2.0 has 'JAVA_OPTIONS' setting defined in each individual bash scripts `fusion/bin/solr`, `fusion/bin/connectors`, `fusion/bin/api`, `fusion/bin/ui`. In 1.2.3, we have moved the JAVA_OPTIONS from bin scripts to config.sh. You will see `API_JAVA_OPTIONS`, `CONNECTORS_JAVA_OPTIONS`, `SOLR_JAVA_OPTIONS`, `UI_JAVA_OPTIONS` in `fusion-new/bin/config.sh`. So, if you have made any changes to the `JAVA_OPTIONS` in individual bin scripts, please update the related config in `fusion-new/bin/config.sh`

9. Copy crawldb from old to new Fusion

  ```
      cp -r fusion/data/connectors/crawldb/* fusion-new/data/connectors/crawldb/
  ```

10. Copy uploaded JDBC drivers

  ```
     cp -r fusion/data/connectors/lucid.jdbc/* fusion-new/data/connectors/lucid.jdbc/
  ```

11. Please follow the different deployment scenarios below
  
   a. If you are using the Fusion-embedded Solr and Zookeeper (Out of the box Fusion):
      * Stop the Solr service if not already stopped `fusion/bin/solr stop`
      * Copy ZK configuration data to the new installation `cp -R fusion/solr/zoo* fusion-new/solr/`
      * move Solr collection data to the new installation: (This might take a while depending on the amount of data in Solr)
          ```
              find fusion/solr -depth 1 | grep -v -E "zoo*" | while read f ; do cp -r  $f fusion-new/solr/; done
          ```

   b. If you are using external Zookeeper and using the Solr inside `fusion` directory:
      * Stop the Solr service if not already stopped `fusion/bin/solr stop`
      * move Solr collection data to the new installation:
          ```
              find fusion/solr -depth 1 | grep -v -E "zoo*" | while read f ; do cp -r  $f fusion-new/solr/; done
          ```

   c. If you are using external Zookeeper and external Solr. (not using Solr, ZK in the fusion directory):
      * No need to do anything. Proceed to the next step

12. If you are using embedded ZK, start ZK from new Fusion instance. After starting, wait for Solr to show up in the Admin UI at port 8983 (http://localhost:8983/solr)

    ```
       fusion-new/bin/solr start
    ```

13. Change to upgrade-scripts and run the script to update data inside ZK

    ```
      cd fusion-upgrade-scripts
      python {zk_host} {updated_exported_file_name}
      E.g., python update_zk_data.py localhost:9983 zk_fusion_updated.export
    ```

14. Start the other fusion services in the new instance. (Start each service individually and check the individual logs to make sure there are no errors)

   ```
     cd ..
     fusion-new/bin/api start
     fusion-new/bin/connectors start
     fusion-new/bin/ui start
   ```

