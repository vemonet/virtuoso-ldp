# Virtuoso as Linked Data Platform

Deploy the OpenLink Virtuoso triplestore with the Linked Data Platform feature enabled, using the official Virtuoso docker image `openlink/virtuoso-opensource-7:latest`

The Linked Data Platform allows you to upload relatively large RDF files over HTTP instead of using the server bulk load which require direct access to the server

RDF files can be uploaded with a simple POST requests to any subfolder at http://your-virtuoso-url/DAV/ 

RDF files added in the DAV file browser is automatically added to the triplestore in the graph corresponding to its access URL

You can then browse the uploaded RDF files through Virtuoso webDAV file browser, and anyone can download a file directly at its graph URI, e.g. https://data.registry.bio2kg.org/DAV/home/dav/bio2kg-registryEvery

## Deploy with Docker

Check for the [`docker-compose.yml`](https://github.com/vemonet/virtuoso-ldp/blob/main/docker-compose.yml) to see if there are any changes you want to make to the deployment configuration.

1. Define the password:

```bash
echo "VIRTUOSO_PASSWORD=yourpassword" > .env
```

2. Run:

```bash
docker-compose up -d
```

3. Once Virtuoso is accessible run the [`prepare_virtuoso_docker.sh`](https://github.com/vemonet/virtuoso-ldp/blob/main/prepare_virtuoso_docker.sh) script to install the VAD packages and create a `/DAV/ldp` folder publicly readable:

```bash
./prepare_virtuoso_docker.sh
```

## Deploy on OpenShift (DSRI)

1. You can easily deploy a Virtuoso Linked Data Platform on the [Data Science Research Infrastructure (DSRI)](https://maastrichtu-ids.github.io/dsri-documentation/) at Maastricht University. Search for the **Virtuoso** template in the Catalog, then deploy Virtuoso based on the `openlink/virtuoso-opensource-7:latest` image. You can check the [YAML of the Virtuoso template here](https://github.com/MaastrichtU-IDS/dsri-documentation/blob/master/applications/templates/template-virtuoso.yml).

2. Use the `oc` command line tool to login and go to your project in your laptop terminal.

```bash
oc project my-project
```

3. Once Virtuoso is available on the DSRI, run the [`prepare_virtuoso_dsri.sh`](https://github.com/vemonet/virtuoso-ldp/blob/main/prepare_virtuoso_dsri.sh) script providing the Application name you gave to Virtuoso when deploying on the DSRI, and the password for the `dba` user you defined. It will install the VAD packages for LDP, and create a `/DAV/ldp` folder publicly readable:

```bash 
./prepare_virtuoso_dsri.sh your-dsri-app-name your-password
```

## Test the LDP

Create a test RDF file:

```bash
echo "<http://subject.org> <http://pred.org> <http://object.org> ." > test-rdf.ttl*
```

First set the `$DBA_PASSWORD` variable, then run this command to upload the RDF file:

```bash
curl -u dav:$DBA_PASSWORD --data-binary @test-rdf.ttl -H "Accept: text/turtle" -H "Content-type: text/turtle" -H "Slug: test-rdf" https://triplestore-bio2kg.apps.dsri2.unimaas.nl/DAV/home/dav*
```

## Virtuoso configuration

Some relevant links to configure Virtuoso for large graphs:

*  http://vos.openlinksw.com/owiki/wiki/VOS/VirtRDFPerformanceTuning
* http://docs.openlinksw.com/virtuoso/rdfperfgeneral/
* https://github.com/tenforce/docker-virtuoso/blob/master/virtuoso.ini

Change the folders allowed for loading files through iSQL:

```bash
- VIRT_Parameters_DirsAllowed=., /data, /opt/virtuoso-opensource/vad, /opt/virtuoso-opensource/data, /usr/local/virtuoso-opensource/share/virtuoso/vad, /usr/local/virtuoso-opensource/var/lib/virtuoso/db
```

Dynamic Renaming of Local URI's: http://docs.openlinksw.com/virtuoso/rdfdynamiclocal/

```bash
- VIRT_URIQA_DynamicLocal=1
```

Link to download VAD packages (e.g. ODS Framework and Briefcase for LDP): http://download3.openlinksw.com/index.html?prefix=uda/vad-vos-packages/7.2/

## Virtuoso iSQL commands

Here are some examples and documentation for useful iSQL commands to configure Virtuoso:

* Bulk load RDF files:

```bash
# Put the RDF file in /opt/virtuoso-opensource/data in the container
docker-compose exec virtuoso isql -U dba -P $DBA_PASSWORD exec="ld_dir_all('/opt/virtuoso-opensource/data', '*.ttl', 'https://w3id.org/um/graph');"
docker-compose exec virtuoso isql -U dba -P $DBA_PASSWORD exec="rdf_loader_run();"
```

* Change folder permissions: http://docs.openlinksw.com/virtuoso/fn_dav_api_change/

```bash
docker-compose exec virtuoso isql -U dba -P $DBA_PASSWORD exec="DB.DBA.DAV_PROP_SET('/DAV/home/dav/', ':virtpermissions', '110100100R','dav', '$DBA_PASSWORD');"
```

* Delete graph in LDP: http://docs.openlinksw.com/virtuoso/fn_dav_api_add/

```bash
docker-compose exec virtuoso isql -U dba -P $DBA_PASSWORD exec="DAV_DELETE ('/DAV/ldp/mydata', 0, 'dav', '$DBA_PASSWORD');"
```

* Create user (working, but does not create the home folder automatically): http://docs.openlinksw.com/virtuoso/fn_user_create/

```bash
docker-compose exec virtuoso isql -U dba -P $DBA_PASSWORD exec="USER_CREATE ('ldp', '${DBA_PASSWORD}', vector ('SQL_ENABLE', '1', 'DAV_ENABLE', '1','HOME', '/DAV/home/ldp') );"
```

* Create ldp user via dav (not working): http://docs.openlinksw.com/virtuoso/fn_dav_api_user/

```bash
docker-compose exec virtuoso isql -U dba -P $DBA_PASSWORD exec="DB.DBA.DAV_ADD_USER ('ldp', '${DBA_PASSWORD}', 'ldp', '110100000', 0, NULL, 'LDP user', 'vincent.emonet@maastrichtuniversity.nl', 'dba', '${DBA_PASSWORD}');"
```

See also: more discussions about setting up LDP

* https://hub.docker.com/r/markw/ldp_server
* https://community.openlinksw.com/t/permissions-on-ldp-containers-and-resources/290/14

### Bulk load RDF

Start Virtuoso using `docker-compose`, or directly docker run:

```bash
docker run -it --rm --name virtuoso -v $(pwd):/data -p 8890:8890 -e DBA_PASSWORD=dba -e VIRT_Parameters_DirsAllowed=".,/usr/local/virtuoso-opensource/share/virtuoso/vad,/data" openlink/virtuoso-opensource-7:latest
```

Bulk load RDF to Virtuoso:

```bash
docker exec -i virtuoso isql -U dba -P dba exec="ld_dir('/data', '*.nq', 'http://test/'); rdf_loader_run();"
```

### Dump one graph

For no reason Virtuoso requires to copy/paste a whole file of code to enable the procedure to dump RDF data.

Create the `dump_one_graph` procedure following those instructions: http://vos.openlinksw.com/owiki/wiki/VOS/VirtRDFDatasetDump#Dump%20One%20Graph

Dump one graph:

```bash
docker exec -i virtuoso isql -U dba -P dba exec="dump_one_graph ('http://bio2rdf.org/drugbank_resource:bio2rdf.dataset.drugbank.R3', '/data/dump_drugbank', 999999999999999);"
```

