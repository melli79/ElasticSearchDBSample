# Elastic Search Database

##  Part 1: Database Initializer

We will write the database initializer in java as there is a nice frontend for ElasticSearch 5 in Java.  The main goals of the Db initializer is to set up the index and the mapping types for the database.

### Requirements

We use the following programs:

1. ElasticSearch 5.6.9 (<6.0),
2. Java 1.8,
3. Gradle 4,
4. as a developing IDE we can use IntelliJ idea (community edition is sufficient),
5. Google Chrome browser plugin ElasticSearch head to check the initialized database.

Please install the components and when installing ES note the cluster name.

### Step 1: Setting up the project

Create a directory with a gradle file with the following dependencies:

* "org.elasticsearch.client:transport:5.6.9"
* "org.slf4j:slf4j-api:1.7.25"
* "org.slf4j:slf4j-simple:1.7.25"

For better Null pointer checking you can use the library 

* "com.google.code.findbugs:jsr305:3.0.2"

Also create a directory src/main/java/com/dlr where you put a minimal DLRAdmin.java.

With these you are set up to create the IntelliJ project.  Open IntelliJ and create a New Project from existing sources.  As type say From external model: gradle.

### Step 2: Initilizing the TransportClient

Set up the main method to create an instance of the DLRClient class.  In the DLRClient class create a member variable (can be final) of type ```org.elasticsearch.client.transport.TransportClient``` .  In ElasticSearch 5.6 you initialize the TransportClient with a ```PreBuildTransportClient``` that takes a ```Settings``` with ```ClusterName.CLUSTER_NAME_SETTING.getKey()``` set to the value of your local ElasticSearch cluster.  You can read this property from an environment variable that needs to be set when calling the program (```System.getProperty("ES_CLUSTER")```).  In the constructor of the ```DLRClient``` set up the ```transportAddress``` to the *localhost* port *9300* (assuming that you have not modified the address of the ES server during setup).  You can test the client from within your main program by calling an ```isServerOk()``` method.  The core is the following execution chain: ```client.admin().cluster().prepareHealth().setTimeout(TimeValue.timeValueSeconds(3))```. You end this sequence with ```.get()``` which executes it (synchronously) and obtains the ```ClusterHealthResponse```.  With the method ```ClusterHealthResponse.getStatus()``` you can check for the status of your cluster.  A ```ClusterHealthStatus.GREEN``` indicates that your cluster is healthy, but also with a state of ```YELLOW``` you are still able to continue processing requests.  This may occur if you only set up one node (and have created a catalog/index).  If the status is ```RED``` (or the method timed out) the cluster is not in a sufficient state.

### Step 3: Creating the Index

Write two methods ```catalogExists``` and ```createCatalog``` where the first one queries the ES-Server whether a catalog with name *DLR* exists.  The general structure of administrative index requests is ```client.admin().indices().<command>``` where <command> is something like ```.prepareExists(<catalogName>)``` to inquiry whether the index exists or ```.prepareCreate(<catalogName>)``` to create the index.  The last argument in the request is the ```.get()``` which executes the request (synchronously) and obtains the request result (use the code completion of your IDE to generate an appropriate variable for the request result).  From this result you can extract either whether the catalog exists (```IndexExistsResponse.isExists()```) or (```IndexCreateResponse.isAcknowledged()```).  In the case the operation did not succeed there are additional fields in the response that indicate what went wrong.

Now is time to add the code to the main program.  For this write an if-statement that checks whether the catalog exists and if not call the ```createCatalog()``` method.

The main program should end with a ```close()``` call that closes the connection of the TransportClient. You should test this part before continuing.

### Step 4: Parsing the column descriptions

Before we can dive into creating the mapping types, we need to understand the possible column types.  This level of indirection is useful as all fields with one name for all mapping types within one index need to be of the same type.  In order to do this you need to import a JSON-library.  Fortunately the ES-TransportClient already comes with a library for that (Jackson-JSON), so we just need to use this ("com.fasterxml.jackson.core:jackson-databind:2.8.6").  First open an ```InputStream``` with the *_columnDescription.json* ```Resource```.  You obtain this stream from the class context, i.e. ```DLRClient.class.getResourceAsStream(<resourceName>)``` where the <resourceName> is the path relative to the project's *src/main/resources/* folder.

Reading the JSON-data is done via a ```com.fasterxml.jackson.databind.ObjectMapper```.  For the moment we don't need to specify any particular properties.  In order to properly read the data, we need to specify a ```com.fasterxml.jackson.core.type.TypeReference```, i.e. create an instance of this generic class.  As the type parameter we choose ```ArrayList<HashMap<String,Object> >``` as these are concrete java types that can hold the parsed data.  In order to translate the type descriptions into ```ColumDescription```s for the remainder of the program we iterate over the entries in the obtained list.

As specified each entry is a ```Map<String, Object>``` by itself.  As such it contains three important attributes: *id* the name of the column, *type* the type of the column, and *label* or *desc* a user friendly description of the column.  In order to contain the column descriptions we create a java type analogously to a ```Map```, e.g. a ```HashMap<String,Object>``` but we write getters and setters for the most frequently used properties (such as ```name```, ```originalName```, ```type```, ```label```, ```description```, ```format```, ```defaultValue``` and ```createStatement```).  In order to properly initialize the ColumnDescription we write a static create method that just takes the ```Map<String, Object>``` from the JSON-parser.  If you simply create the ColumnDescriptions from the Maps, you will notice that some columns do not contain any ```name```.  Here you have to be creative and either skip these columns (a problem may arise if any of the type specifications contains such a column) or you derive a column ```name``` from the ```label```.  You also need to check whether the ColumnDescription contains a type specification (you could generate that from the name).  The ```createStatement```s are necessary for the ```createMappingType``` calls to our ```DLRClient```.  One way to formulate them is to put a ```Map<String,Object>``` that contains a key *type* with value the mapped ES-type.  Primitive types are *integer*, *text*, *keyword*, *byte*, *geo_point*, *date* and some more.  Note that for the date type you better also specify the *format* as something like "yyyy-MM-dd HH:mm:ss.SSS" that is the joda format string for the dates as they occur in the TSV-files.

More difficult is the handling of the List types.  These can be represented as *nested* types with *properties* containing a ```Collections.singletonMap<>()``` with key the name of the column (wihtout the List) and value a ```Collections.singletonMap<>()``` of key *type* and value something like *text* (or whatever type the List elements are).

Special care needs to be taken of the *identifier* column, namely this needs to be parsed from the TSV-file, it should not be written into the JSON-Object's properties, but send separately as the identifier of the Object to be indexed.  Also this column should not be mentioned explicitly in the MappingType createStatement.  The best is to leave out the ```createStatement``` for it, but this requires the corresponding getter method to return an ```java.util.Optional<Object>```.  It is necessary to keep the ```originalName``` of the column as it is named in the JSON descriptions of the MappingTypes.

Once you have written the parsing method for the column descriptions, you should write out some ```ColumnDescription```s and see whether they are plausible.

### Step 5: Generating the MappingTypes

Next comes the parsing of the type descriptions.  These are also provided as JSON-files and should be embedded into the jar bundle.

The same as before we write one method that generates all the type descriptions.  Basically a type consists of a name (a ```String```) and a ```List<ColumnDescription>```s, so it would be sufficient for this method to return a ```Map<...>``` of those.  Within this method open an ```InputStream``` to one JSON-file (via ```try (InputStream is= new FileInputStream(new File(<fileName>))) {...}``` or ```DLRClient.class.getResourceAsStream(<resourceName>)```) and hand it together with the type name and the predefined ```columns``` to the ```createType``` method.

The ```createType``` method does three things.  First it reads the JSON-```InputStream``` into a ```Map<String,Object>``` analogous to reading the column descriptions.  Then it processes the *columns* entry and the *constants* entry into a ```List<ColumnDescription>``` and finally it tests whether the mapping type already exists and if not it generates a  PutMappingRequest for the read mapping type.

*columns* are easier to translate, namely we just need to find the named column type and more or less add the corresponding ColumnDescription to the intended *result* of the method.  We need however to check whether a column occurs multiple times.  In this case we need to wrap the column *type* into the type *nested* and add the *properties* as a ```Collections.singletonMap<String,Object>``` that has the original column name as key and the original ```createStatement``` a value.  On order to avoid column ambiguation we need to rename the original column into something indicating the plural, e.g. adding an *s*.  Make sure to put this modified ```ColumnDescription``` into your ```Map<String,ColumnDescription>```.  It may also be necessary to keep the ```originalName``` in the ```ColumnDescription``` and of course you need to replace all occurrences of the column by the new ```ColumnDescription```.

The *constants* should be translated into ```ColumnDescription```s with an additional ```defaultValue```.  It is sufficient to keep the defaultValue in String format, however you also need to decide for a *type* for the newly created ```ColumnDescription```.  If the *column* already ocurred in your list of known columns, you should stick with that type otherwise you will have to invent the type, e.g. something like *text*.

By now you have assembled the *result* value of the method.  It remains to check whether the MappingType already exists in the Catalog.  For this write another method that calls the ```client.indices().preparyTypeExists(<CatalogName>).setTypes(<typeName>)``` and ```.get()``` the response from the ES-Server.  The ```TypeExistsResponse.isExists()``` gives the desired result.

Finally for creating the MappingType (if it does not yet exist) a ```Map<String,Object>``` that takes the column ```name```s together with their ```createStatement```s is necessary.  Wrap this map in yet another Map that has two entries: *_source* for with a ```Collections.singletonMap<>()``` of key *enabled* and value *true* that allows this MappingType to be a base type and *properties* that takes the previous Map of properties for the MappingType.  The request now looks as follows ```client.admin().indices().preparePutMapping(<catalogName>).setType(<typeName>).setSource(<the above Map>)``` and should be executed with ```.get()```.  The ```PutMappingResponse.isAcknowledged()``` should be checked for success in creating the MappingType.  In particular watch out for ```ClassCastException```s that indicate some errors translating the parsed JSON-file.

Finally call the ```createTypes()``` method from the main program.  For testing you should not yet translate all MappingTypes at once, but start with one or two files.

### Step 6: Import TSV-Files

Finally the main part, importing the actual data.  In your main program iterate over the TSV-files (e.g. passed as arguments to the program).  For each file open an ```InputStream``` and pass this together with the type name and ```List<ColumnDescription>```s to the ```insertLines()``` method.  Within this method iterate over the lines in the ```InputStream```, e.g. create a ```BufferedReader reader= new BufferedReader(new InputStreamReader(is))``` and ```for(String line; (line=reader.readline())!=null;) {…}``` first split the line at tab stops, e.g. ```String[] values= line.split("\t");```  Create a ```Map<String,Object>``` with the values of the object to be indexed.  You may also need a variable ```identfier```.  Now iterate over the ```ColumnDescription```s (and keep track of the next available ```value```).  Depending on whether the column has a default value or not, you need to either insert the ```column.getName()``` with the ```defaultValue``` or insert the next available value.  You should pay special attention to two special columns:  *updatedAt* column you need to compute the value to be inserted as ```value= DateTime.now().toString(timeFormatter);``` where the ```org.joda.time.format.DateTimeFormatter``` is created from you specified ```DateTimeFormat``` string (property ```format``` of the ColumnDescription).  For the *_id* column you need to keep the ```value``` in the extra variable ```identifier```.  Also pay attention to whether the ```value``` is actually specified.  Due to the ```line.split(…)``` instruction there is always a ```String``` value, even the empty string, but if it is the empty string you need to omit the value from the Map. ```multiValue```s need to be passed a bit differently, namely as a ```List<Map<String, Object> >``` of ```Collections.singletonMap<String, Object>()``` where the key is the ```name``` you specified in the ```createStatement``` of the *nested* type and the value is the actual *value* you wish to insert.

Once you have gathered all values (including the ```defaultValue```s for the *constans*) you are ready to index the Object via ```client.prepareIndex(<catalogName>,<typeName>).setId(identifer).setSource(<the values to be written>)``` and execute it synchronously with a ```.get()```.  A return value can come in one of the two ways:  a ```MapperParsingException``` with a possible ```.getCause``` an ```IllegalArgumentException``` indicating that some passed value cannot be parsed into the required column *type* or a ```IndexResponse.getResult``` of type ```DocWrireResponse.Result``` either ```CREATED``` indicating that the Object was created or ```UPDATED``` indicating that an existing Object was updated.  Note that in the latter case unspecified columns will have kept their values.  It would be nice for the user to get some feedback of how far the indexing process has come (you could ```System.out.print(".");``` a dot for every successfully processed Object, but don't forget to ```System.out.flush()``` in order to make the dot visible immediately.)

Before importing the whole database in bulk consider testing the import with a smaller TSV-file.  Watch out for ```IllegalArgumentException```s.

It may be useful to gather the output of the importer in a log file and to check this logfile for errors in particular TSV-files.