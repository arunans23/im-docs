Writing your own data source
================================

The aim of this tutorial is to create a new source. Often this will be to import data from an InterMine Items XML format file that you create, though other types of source can also be created (e.g. a source that extends InterMine's existing GFF3 importer. 

There are three parts to creating a new source:

1. Create the directory structure in InterMine that contains files that describe the source.
2. Configure the mine to use this source (make an entry in your `project.xml`).
3. Write code to parse the source. You can either do this in InterMine directly, by extending the `DataConverter` class, or you can use some other language to generate a standalone InterMine Items XML file, and set `have.file.xml.tgt = true` in the source's project.properties file.  See `this page <../apis/index.html>`_ for more information on the InterMine Items XML file format and links to language-specific APIs (Perl, Python, etc.) that can help create it.

Create source files
-----------------------

Run make_source script
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This script creates the basic skeleton for a source.   It should be run in the top level directory of an InterMine checkout, like this:

.. code-block:: bash

  $ ./bio/scripts/make_source <source-name> <source-type>


Possible source types
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Run `make_source` with no arguments to get a full list of source types.

custom-file
""""""""""""""

This a source that reads from a file in a custom format.  A custom FileConverter will be needed.  The `make_source` script will
create a skeleton `FileConverter` in `bio/sources/<source-name>/main/src/org/intermine/bio/dataconversion`.  Edit this code to process the particular file you need to load, using the internal :doc:`/database/data-sources/apis/java-items-api` to create and store items to the database.

intermine-items-xml-file
""""""""""""""""""""""""""""

This type of source can read a file in InterMine Items XML format and store the data in a mine.  The `project.xml` configuration is as below:

.. code-block:: xml

    <source name="my-new-source-name" type="my-new-source">
      <property name="src.data.file" location="/some/directory/objects_in_intermine_format.xml"/>
    </source>

See `this page <../apis/index.html>`_ for more information on the Items XML format and links to APIs that can generate it. This source type doesn't generate any stub Java code.

intermine-items-large-xml-file
""""""""""""""""""""""""""""""""""""""""""

This source works as above but writes the XML to an intermediate items database to avoid reading the whole file into memory at once.  This is the best choice for large XML files, where large is several hundred megabytes (although this depends on the amount of RAM specified in your `ANT_OPTS` environment variable).  

db
""""""""""""""""""""""""""""

This source reads directly from a relational database, it will generate a skeleton `DBConverter` in `bio/sources/<source-name>/main/src/org/intermine/bio/dataconversion`.  To connect to the database you need to add properties in xxxmine.properties with the prefix `db.sourcename`.  This is tested for PostgreSQL and MySQL.

Common properties:

.. code-block:: xml

  db.sourcename.datasource.dataSourceName=db.sourcename
  db.sourcename.datasource.maxConnections=10
  db.sourcename.datasource.serverName=SERVER_NAME
  db.sourcename.datasource.databaseName=DB_NAME
  db.sourcename.datasource.user=USER_NAME
  db.sourcename.datasource.password=USER_PASSWORD

Add these for PostgreSQL:

.. code-block:: xml

  db.sourcename.datasource.class=com.zaxxer.hikari.HikariDataSource
  db.sourcename.datasource.dataSourceClassName=org.postgresql.ds.PGSimpleDataSource
  db.sourcename.driver=org.postgresql.Driver
  db.sourcename.platform=PostgreSQL

Add these for MySQL:

.. code-block:: xml

  db.sourcename.datasource.class=com.mysql.jdbc.jdbc2.optional.MysqlConnectionPoolDataSource
  db.sourcename.driver=com.mysql.jdbc.Driver
  db.sourcename.platform=MySQL

.. toctree::
    :maxdepth: 1

    oracle

It is good practice to put the properties that won't change in `MINE_NAME/default.intermine.integrate.properties` and those that may change (`serverName`, `databaseName`, `user`, `password`) in `~/.intermine/MINE_NAME.properties`.

The db value has to match the '''source.db.name''' in your project XML entry, for example:

.. code-block:: xml

    # project XML
    <source name="chado-db-flybase-dmel" type="chado-db">
      <property name="source.db.name" value="flybase"/>
      ...
    </source>

.. code-block:: properties

  # flymine.properties

  db.flybase.datasource.class=com.zaxxer.hikari.HikariDataSource
  db.flybase.datasource.dataSourceClassName=org.postgresql.ds.PGSimpleDataSource
  db.flybase.datasource.dataSourceName=db.flybase
  db.flybase.datasource.serverName=LOCALHOST
  db.flybase.datasource.databaseName=FB2011_01
  db.flybase.datasource.user=USERNAME
  db.flybase.datasource.password=SECRET
  db.flybase.datasource.maxConnections=10
  db.flybase.driver=org.postgresql.Driver
  db.flybase.platform=PostgreSQL

gff
""""""""""""""""""""""""""""

Create a gff source to load genome annotation in GFF3 format.  This creates an empty `GFF3RecordHandler` in `bio/sources/<source-name>/main/src/org/intermine/bio/dataconversion`.  The source will work without any changes but you can edit the `GFF3RecordHandler` to read specific attributes from the last column of the GFF3 file.  See the InterMine tutorial for more information on integrating GFF3.

obo
""""""""""""""""""""""""""""

Create a obo source to load ontology in BO format.

Additions file 
~~~~~~~~~~~~~~~~~~~~~~~~~~

Update the file in the source folder called `new-source_additions.xml`.  This file details any extensions needed to the data model to store data from this source, everything else is automatically generated from the model description so this is all we need to do to add to the model.  The file is in the same format as a complete Model description.

To add to an existing class the contents should be similar to the example code below. The class name is a class already in the model, the attribute name is the name of the new field to be added and the type describes the type of data to be stored. In the example the `Protein` class will be extended to include a new attribute called `extraData` which will hold data as a string.   

.. code-block:: xml

  <?xml version="1.0"?>
  <classes>
    <class name="Protein>" is-interface="true">
      <attribute name="extraData" type="java.lang.String"/>   
    </class>
  </classes>

To create a new class the `new-source_additions.xml` file should include contents similar to the example below:

.. code-block:: xml

  <?xml version="1.0"?>
  <classes>
    <class name="NewFeature" extends="SequenceFeature" is-interface="true">
      <attribute name="identifier" type="java.lang.String"/>  
      <attribute name="confidence" type="java.lang.Double"/>
    </class>
  </classes>

The extends clause is optional and is used to inherit (include all the attributes of) an existing class, in this case we extend `SequenceFeature`, an InterMine class that represents any genome feature.  `is-interface` should always be set to true.  The attribute lines as before define the names and types of data to be stored.  A new class will be created with the name `NewFeature` that extends `SequenceFeature`. 

To cross reference this with another class, similar XML should be used as the example below:

.. code-block:: xml

  <class name="NewFeature" extends="SequenceFeature" is-interface="true">
    <reference name="protein" referenced-type="Protein" reverse-reference="features"/>
  </class>

In the example above the we create a link from NewFeature to the Protein class via the reference named protein. To complete the link a reverse reference may be added to Protein to point back at the NewFeature, this is optional - the reference could be one-way.  Here we define a collection called features, this means that for every NewFeature that references a Protein, that protein will include it in its features collection.  Note that as this is a collection a Protein can link to multiple NewFeatures but NewFeature.protein is a reference so each can only link to one Protein.  

The reverse entry needs to be added to Protein (still in the same file):

.. code-block:: xml

  <class name="Protein" is-interface="true">
    <collection name="features"  referenced-type="NewFeature" reverse-reference="protein"/>
  </class>

The final additions XML should look like:

.. code-block:: xml

  <?xml version="1.0"?>
  <classes>
    <class name="Protein>" is-interface="true">
      <attribute name="extraData" type="java.lang.String"/> 
      <collection name="features"  referenced-type="NewFeature" reverse-reference="protein"/>  
    </class>
    <class name="NewFeature" extends="SequenceFeature" is-interface="true">
      <attribute name="identifier" type="java.lang.String"/>  
      <attribute name="confidence" type="java.lang.Double"/>
      <reference name="protein" referenced-type="Protein" reverse-reference="features"/>
    </class>
  </classes>

.. note::

  If all the data you wish to load is already modelled in InterMine then you don't need an additions file.

Properties
~~~~~~~~~~
Any properties you define in a source entry in your mine's project.xml will be set on that source's converter or post-processing class, providing that there is a setter with an appropriate name.

This applies to any class that inherits from

* org.intermine.dataconversion.DataConverter

* org.intermine.dataconversion.DBConverter

* org.intermine.dataconversion.DirectoryConverter

* org.intermine.dataconversion.FileConverter

* org.intermine.postprocess.PostProcessor

For instance, if you have the source entry

.. code-block:: xml

    <source name="my-new-source-name" type="my-new-source">
      <property name="fooFile" location="/some/directory/objects_in_intermine_format.xml"/>
      <property name="bar.info" location="baz"/>
      <property name="bazMoreInfo" name="hello-world"/>
    </source>

in your project.xml file and a class that extends org.intermine.postprocess.PostProcessor, then before post-processing the following methods will be called on that class with these parameters

.. code-block:: java

  myPostProcessor.setFooFile(new File("/some/directory/objects_in_intermine_format.xml"));
  myPostProcessor.setBarInfo("baz");
  myPostProcessor.setBazMoreInfo("hello-world");


Keys file
~~~~~~~~~~~~~~~~~~~~~~~~~~

Within the `resources` directory is a file called `new-source_keys.properties`.  Here we can define primary keys that will be used to integrate data from this source with any exiting objects in the database.  We want to integrate proteins by their (UniProt) primaryAccession attribute so we define that this source should use the key:

.. code-block:: properties

  Protein=key_primaryacc

Note that we don't expect any other data sources to provide interesting features so we don't need to integrate them - no key is defined.  The possible keys are defined in `dbmodel/resources/genomic_keyDefs.properties`, new keys can be added if necessary.  

Including your source in a Mine
----------------------------------------------

Project XML
~~~~~~~~~~~~~~~~~~~~~~~~~~

In the `project.xml` file, in the root of your mine directory (e.g. /malariamine), the following entries should be added and altered accordingly:

.. code-block:: xml

  <source name="new-source-name" type="new-source">
    <property name="src.data.file" location="my_data_dir/example.xml"/>
  </source>

If you have more that one file you can set this up to point at a '''directory''':

.. code-block:: xml

  <source name="new-source-name" type="new-source">
    <property name="src.data.dir" location="my_data_dir/source_files/"/>
  </source>

The first line defines the name you wish to give to the of the source and the type - the name of the directory in 'bio/sources'.  The second line defines the location and name of the data file.

If you are using data from a database:

.. code-block:: xml

    <source name="new-source-name" type="new-source">
      <property name="source.db.name" value="db.NAME"/>
      ...
    </source>

The value of `source.db.name` must match the value set in the MINE_NAME.properties file.

Run build-db
~~~~~~~~~~~~~~~~~~~~~~~~~~

Create the database as usual. The source should now be included when building the mine.

.. note::

  Unless the 'clean' is run (which deletes the build directory) in `MINE_NAME/dbmodel` any changes will append to the current model structure and any unwanted classes/attributes will remain.

.. index:: writing a custom data source, custom data source
