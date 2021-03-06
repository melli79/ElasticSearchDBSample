# Elastic Search Database

##  Part 1: Database Initializer

We will write the database initializer in Java as there is a nice frontend for ElasticSearch 5 in Java.  The main goals of the Db initializer is to set up the index and the mapping types for the database.

### Requirements

We use the following programs:

1. ElasticSearch 5.6.9 (<6.0),
2. Java 1.8,
3. Gradle 4,
4. as a developing IDE we can use IntelliJ idea (community edition is sufficient),
5. Google Chrome browser plugin *ElasticSearch head* to check the initialized database.

Please install the components and when starting ES note the cluster name.

### Step 1: Setting up the project

Create a directory with a gradle file with the following dependencies:

* "org.elasticsearch.client:transport:5.6.9"
* "org.slf4j:slf4j-api:1.7.25"
* "org.slf4j:slf4j-simple:1.7.25"

For better Null pointer checking you can use the library 

* "com.google.code.findbugs:jsr305:3.0.2"

In order to keep the jar-file sorted neatly, you can

* ```apply plugin: 'org.springframework.boot'``` from the 
* ```buildscript { dependencies {  classpath("org.springframework.boot:spring-boot-gradle-plugin:2.0.1.RELEASE")} }```.

Also create a directory src/main/java/com/dlr where you put a minimal DLRAdmin.java, e.g.:

```
package com.dlr;

import javax.annotation.Nonnull;

public class DLRAdmin {
    public static void main(@Nonnull String... args) {
        
    }
}
```

With these you are set up to create the IntelliJ project.  Open IntelliJ and create a *New Project from existing sources*.  As type say *From external model*: *gradle*.

### Step 2: Initializing the TransportClient

Set up the main method to create an instance of the ```DLRAdmin``` class.  In the ```DLRAdmin``` class create a member variable (can be final) of type ```org.elasticsearch.client.transport.TransportClient``` .  In ElasticSearch 5.6 you initialize the TransportClient with a ```PreBuildTransportClient``` that takes a ```Settings``` with ```ClusterName.CLUSTER_NAME_SETTING.getKey()``` set to the value of your local ElasticSearch cluster.  You can read this property from an environment variable that needs to be set when calling the program (```System.getProperty("ES_CLUSTER")```).  In the constructor of the ```DLRAdmin``` set up the ```transportAddress``` to the *localhost* port *9300* (assuming that you have not modified the address of the ES server during setup).  You can test the client from within your main program by calling an ```isServerOk()``` method.  The core is the following execution chain: ```client.admin().cluster().prepareHealth().setTimeout(TimeValue.timeValueSeconds(3))```. You end this sequence with ```.get()``` which executes it (synchronously) and obtains the ```ClusterHealthResponse```.  With the method ```ClusterHealthResponse.getStatus()``` you can check for the status of your cluster.  A ```ClusterHealthStatus.GREEN``` indicates that your cluster is healthy, but also with a state of ```YELLOW``` you are still able to continue processing requests.  This may occur if you only set up one node (and have created a catalog/index).  If the status is ```RED``` (or the method timed out) the cluster is not in a sufficiently healthy state.

### Step 3: Creating the Index

Write two methods ```catalogExists``` and ```createCatalog``` where the first one queries the ES-Server whether a catalog with name *DLR* exists.  The general structure of administrative index requests is ```client.admin().indices().<command>``` where <command> is something like ```.prepareExists(<catalogName>)``` to inquiry whether the index exists or ```.prepareCreate(<catalogName>)``` to create the index.  The last argument in the request is the ```.get()``` which executes the request (synchronously) and obtains the request result (use the code completion of your IDE to generate an appropriate variable for the request result).  From this result you can extract either whether the catalog exists (```IndexExistsResponse.isExists()```) or (```IndexCreateResponse.isAcknowledged()```).  In the case the operation did not succeed there are additional fields in the response that indicate what went wrong.

Now is time to add the code to the main program.  For this write an if-statement that checks whether the catalog exists and if not call the ```createCatalog()``` method.

The main program should end with a ```close()``` call that closes the connection of the TransportClient. You should test this part before continuing.

### Step 4: Parsing the column descriptions

Before we can dive into creating the mapping types, we need to understand the possible column types.  This level of indirection is useful as all fields with one name for all mapping types within one index need to be of the same type.  In order to do this you need to import a JSON-library.  Fortunately the ES-TransportClient already comes with a library for that (Jackson-JSON), so we just need to use this ("com.fasterxml.jackson.core:jackson-databind:2.8.6").  First open an ```InputStream``` with the *_columnDescription.json* ```Resource```.  You obtain this stream from the class context, i.e. ```DLRAdmin.class.getResourceAsStream(<resourceName>)``` where the <resourceName> is the path relative to the project's *src/main/resources/* folder.

Reading the JSON-data is done via a ```com.fasterxml.jackson.databind.ObjectMapper```.  For the moment we don't need to specify any particular properties.  In order to properly read the data, we need to specify a ```com.fasterxml.jackson.core.type.TypeReference```, i.e. create an instance of this generic class.  As the type parameter we choose ```ArrayList<HashMap<String,Object> >``` as these are concrete java types that can hold the parsed data.  In order to translate the type descriptions into ```ColumDescription```s for the remainder of the program we iterate over the entries in the obtained list.

As specified each entry is a ```Map<String, Object>``` by itself.  As such it contains three important attributes: *id* the name of the column, *type* the type of the column, and *label* or *desc* a user friendly description of the column.  In order to contain the column descriptions we create a java type analogously to a ```Map```, e.g. an ```public class ColumnDescription extends HashMap<String,Object>``` but we write getters and setters for the most frequently used properties (such as ```name```, ```originalName```, ```type```, ```label```, ```description```, ```format```, ```defaultValue``` and ```createStatement```).  In order to properly initialize the ColumnDescription we write a static create method that just takes the ```Map<String, Object>``` from the JSON-parser.  If you simply create the ```ColumnDescription```s from the Maps, you will notice that some columns do not contain any ```name```.  Here you have to be creative and either skip these columns (a problem may arise if any of the type specifications contains such a column) or you derive a column ```name``` from the ```label```.  You also need to check whether the ```ColumnDescription``` contains a type specification (you could generate that from the name).  The ```createStatement```s are necessary for the ```createMappingType``` calls to our ```DLRAdmin```.  One way to formulate them is to put a ```Map<String,Object>``` that contains a key *type* with value the mapped ES-type.  Primitive types are *integer*, *text*, *keyword*, *byte*, *geo_point*, *date* and some more.  Note that for the date type you better also specify the *format* as something like "yyyy-MM-dd HH:mm:ss.SSS" that is the joda format string for the dates as they occur in the TSV-files.

More difficult is the handling of the List types.  These can be represented as *nested* types with *properties* containing a ```Collections.singletonMap<>()``` with key the name of the column (wihtout the "List") and value a ```Collections.singletonMap<>()``` of key *type* and value something like *text* (or whatever type the List elements are).

Special care needs to be taken of the *identifier* column, namely this needs to be parsed from the TSV-file, it should not be written into the JSON-Object's properties, but send separately as the identifier of the Object to be indexed.  Also this column should not be mentioned explicitly in the MappingType createStatement.  The best is to leave out the ```createStatement``` for it, but this requires the corresponding getter method to return an ```java.util.Optional<Object>```.  It is necessary to keep the ```originalName``` of the column as it is named in the JSON descriptions of the MappingTypes.

Once you have written the parsing method for the column descriptions, you should write out some ```ColumnDescription```s and see whether they are plausible.

### Step 5: Generating the MappingTypes

Next comes the parsing of the type descriptions.  These are also provided as JSON-files and should be embedded into the jar bundle.

The same as before we write one method that generates all the type descriptions.  Basically a type consists of a name (a ```String```) and a ```List<ColumnDescription>```s, so it would be sufficient for this method to return a ```Map<...>``` of those.  Within this method open an ```InputStream``` to one JSON-file (via ```try (InputStream is= new FileInputStream(new File(<fileName>))) {...}``` or ```DLRAdmin.class.getResourceAsStream(<resourceName>)```) and hand it together with the type name and the predefined ```columns``` to the ```createType``` method.

The ```createType``` method does three things.  First it reads the JSON-```InputStream``` into a ```Map<String,Object>``` analogous to reading the column descriptions.  Then it processes the *columns* entry and the *constants* entry into a ```List<ColumnDescription>``` and finally it tests whether the mapping type already exists and if not it generates a  PutMappingRequest for the read mapping type.

*columns* are easier to translate, namely we just need to find the named column type and more or less add the corresponding ColumnDescription to the intended *result* of the method.  We need however to check whether a column occurs multiple times.  In this case we need to wrap the column *type* into the type *nested* and add the *properties* as a ```Collections.singletonMap<String,Object>``` that has the original column name as key and the original ```createStatement``` a value.  In order to avoid column ambiguity we need to rename the original column into something indicating the plural, e.g. adding an *s*.  Make sure to put this modified ```ColumnDescription``` into your ```Map<String,ColumnDescription>```.  It may also be necessary to keep the ```originalName``` in the ```ColumnDescription``` and of course you need to replace all occurrences of the column by the new ```ColumnDescription```.

The *constants* should be translated into ```ColumnDescription```s with an additional ```defaultValue```.  It is sufficient to keep the ```defaultValue``` in String format, however you also need to decide for a *type* for the newly created ```ColumnDescription```.  If the *column* already ocurred in your list of known columns, you should stick with that type, otherwise you will have to invent the type, e.g. something like *text*.

By now you have assembled the *result* value of the method.  It remains to check whether the MappingType already exists in the Catalog.  For this write another method that calls the ```client.indices().preparyTypeExists(<CatalogName>).setTypes(<typeName>)``` and ```.get()``` the response from the ES-Server.  The ```TypeExistsResponse.isExists()``` gives the desired result.

Finally for creating the MappingType (if it does not yet exist) a ```Map<String,Object>``` that takes the column ```name```s together with their ```createStatement```s is necessary.  Wrap this map in yet another Map that has two entries: *_source* with a ```Collections.singletonMap<>()``` of key *enabled* and value *true* that allows this MappingType to be a base type and *properties* that takes the previous Map of properties for the MappingType.  The request now looks as follows ```client.admin().indices().preparePutMapping(<catalogName>).setType(<typeName>).setSource(<the above Map>)``` and should be executed with ```.get()```.  The ```PutMappingResponse.isAcknowledged()``` should be checked for success in creating the MappingType.  In particular watch out for ```ClassCastException```s that indicate some errors translating the parsed JSON-file.

Finally call the ```createTypes()``` method from the main program.  For testing you should not yet translate all MappingTypes at once, but start with one or two files.

### Step 6: Import TSV-Files

Finally the main part, importing the actual data.  In your main program iterate over the TSV-files (e.g. passed as arguments to the program).  For each file open an ```InputStream``` and pass this together with the type name and ```List<ColumnDescription>```s to the ```insertLines()``` method.  Within this method iterate over the lines in the ```InputStream```, e.g. create a ```BufferedReader reader= new BufferedReader(new InputStreamReader(is))``` and ```for(String line; (line=reader.readline())!=null;) {…}``` first split the line at tab stops, e.g. ```String[] values= line.split("\t");```  Create a ```Map<String,Object>``` with the values of the object to be indexed.  You may also need a variable ```identfier```.  Now iterate over the ```ColumnDescription```s (and keep track of the next available ```value```).  Depending on whether the column has a default value or not, you need to either insert the ```column.getName()``` with the ```defaultValue``` or insert the next available ```value```.  You should pay special attention to two special columns:  *updatedAt* column you need to compute the value to be inserted as ```value= DateTime.now().toString(timeFormatter);``` where the ```org.joda.time.format.DateTimeFormatter``` is created from your specified ```DateTimeFormat``` string (property ```format``` of the ColumnDescription).  For the *_id* column you need to keep the ```value``` in the extra variable ```identifier```.  Also pay attention to whether the ```value``` is actually specified.  Due to the ```line.split(…)``` instruction there is always a ```String``` value, at least the empty string, but if it is the empty string, the value is not specified and you need to omit it from the Map. ```multiValue```s need to be passed a bit differently, namely as a ```List<Map<String, Object> >``` of ```Collections.singletonMap<String, Object>()``` where the key is the ```name``` you specified in the ```createStatement``` of the *nested* type and the value is the actual ```value``` you wish to insert.

Once you have gathered all values (including the ```defaultValue```s for the *constans*) you are ready to index the Object via ```client.prepareIndex(<catalogName>,<typeName>).setId(identifer).setSource(<the values to be written>)``` and execute it synchronously with a ```.get()```.  A return value can come in one of the two ways:  a ```MapperParsingException``` with a possible ```.getCause()``` an ```IllegalArgumentException``` indicating that some passed value cannot be parsed into the required column *type* or a ```IndexResponse.getResult``` of type ```DocWrireResponse.Result``` either ```CREATED``` indicating that the Object was created or ```UPDATED``` indicating that an existing Object was updated.  Note that in the latter case unspecified columns will have kept their values.  It would be nice for the user to get some feedback of how far the indexing process has come (you could ```System.out.print(".");``` a dot for every successfully processed Object, but don't forget to ```System.out.flush()``` in order to make the dot visible immediately.)

Before importing the whole database in bulk consider testing the import with a smaller TSV-file.  Watch out for ```IllegalArgumentException```s.

It may be useful to gather the output of the importer in a log file and to check this logfile for errors in particular TSV-files.


## Part 2: Writing a frontend for querying the Database
### Components
We will use the following program to write the client:
* AngularJS 6.0/Node.js 10.0 (https://angular.io/tutorial) (downloads via the node commands npm/ng)
* as an IDE you could use intelliJ idea as well (with the AngularJS plugin)
* Apache web server to deliver the client to external browsers.
* recent browser with java script support (e.g. Chrome, Firefox, Edge)

### Step 1: Setting up the project
Install and update the node.js command line tools:
```nmp install "@angular/cli"```

Then you create a new angularJS project by executing ```ng create webclient``` where ''webclient'' is the name of the project.  You can now open this project in your favorite java script/ type script editor, e.g. IntelliJ Idea with angularJS plugin.  Open the file src/app/app.component.ts and in the defined class introduce the two readonly variables title and version.  You may set the latter to something like 0.1.0 as this is just the initialization of the project.  Then edit the file src/app/app.component.ts and replace the angular placeholder with something like
```<h1>{{title}}</h1>``` and maybe a copyright notice and ```{{version}}```.

### Step 2: Setting up the user interface
On the command line run the command ```ng generate module search```.  Once the generation is complete, you can modify the '''src/app/app.component.html''' to include the newly created module: ```<app-component></app-component>```.

In the file '''src/app/search/search.component.ts''' define a (public) variable types of type MappingType[] with initial value null. Then create a new file '''src/app/mappingType.ts''' in which you define the '''export interface''' MappingType with the fields '''readonly''' ''name'' of type '''string''', ''desc''? of type '''string''' and fields of type ColumnType[].  This looks as follows:
```
  '''export interface''' MappingType {
    '''readonly''' name :'''string''';
    desc? :'''string''';
    fields :ColumnType[];
  }
```

Create a new file '''src/app/columnType.ts''' and define the '''export interface''' ColumnType with the fields '''readonly''' ''id'' of type '''string''', ''label''? of type '''string''' and ''desc''? of type '''string'''.  Now return to the file '''src/app/MappingType.ts''' and trigger an autocomplete on the unknown type ColumnType (this should ```import {ColumnType} from './columnType';```).

Return to the file '''src/app/search/search.component.ts''' and trigger an autocomplete on the unknown MappingType.  Add to the function ''ngOnInit()'' a line to initialize the ''types'' variable with a list of one MappingType of ''name'' 'airrsgtc' and columns {'_id', 'uniqueID'}, {'sensor0'}, {'utcstarttime0'}.

Add more variables to the class: ''selectedType'' of MappingType with initial value '''null''' and ''selectedColumns'' of ColumnType[] and initial value '''null'''.

Now open the file ''src/app/search/search.component.html'' and add the structure for selecting a MappingType to query:  A good initial choice would be a ```<form>``` with a ```<select *ngIf="types" data-ng-model="selectedType">``` a default option ```<option>&lt;Choose a MappingType&gt;</option>``` and generated options ```"let t of types"``` with ```value="t"``` and description ```{{t.name}}```.  This should look as follows:
```
  <form>
    <select *ngIf="types" data-ng-model="selectedType">
      <option>&lt;Choose a MappingType&gt;</option>
      <option *ngFor="let t of types" value="t">{{t.name}}</option>
    <select>
    <button type="button" (click)="loadData()">Load</button>
  </form>
```
Here we have added a button to load the table for the selected MappingType.  Make sure you specify the ```type="button"``` otherwise your app will reload whenever you click this button (```type="submit"```, the default behavior).

Define a method ''loadData() :void'' in the class (*.ts file), but leave it empty for now.

You can test the web client by running ```ng serve``` and poiting your browser to 'http://localhost:4200/'.

### Step 3: Setting up the elasticSearch service
Run the command ```ng generate service ElasticSearch``` which creates a file ''src/app/elasticSearch.service.ts'' and registers the service in the ''src/app/app.module.ts''.  Open this file and add to imports the HttpClientModule.  Click autocomplete on the unknown type HttpClientModule and it should add the ```import { HttpClientModule } from '@angular/common/http';```

Now open the previously generated file ''src/app/elasticSearch.service.ts'' and add a '''private''' argument ''httpClient'' of type HttpClient to the constructor.  Press autocomplete on HttpClient to add the ```import { HttpClient } from '@angular/common/http';```  Now add a private variable ''url'' of value ```'http://localhost:9200/dlr/'```.  This will be the web access to the ElasticSearch database.

Create a (public) method ''query'' with parameters ''selectedType'' of MappingType, ''page'' of default value 0, and page size of default value 20.  The method just returns an http-query as follows:  Call ```httpClient.get<QueryResult<ResultType> >(this.url+selectedType.name+'/_search?from='+page*size+'&size='+size)```  Click autocomplete on the unknown type QueryResult and generate a local type definition as:
```
  interface QueryResult<T> {
    took: number;
    timed_out: boolean;
    _shards: {
      total: number,
      successful: number,
      skipped: number,
      failed: number
    };
    hits: {
      total: number,
      max_score: number,
      hits: T[]
    };
  }
```

Create a new file ''src/app/abstractType.ts'' and define '''exported interface''' AbstractType which has a variable ''_id'' of type '''string''' and [''name'' :string] of type any.  Define furthermore '''exported interface''' ResultType which '''extends''' AbstractType and adds the fields ''_index'', ''_type'' and ''_score'' of type '''string''' and
''_source'' of type '''Object'''.

Now return to the file ''src/app/elasticSearch.service.ts'' and click autocomplete on the unknown type ResultType.  This should add the ```import { ResultType } from '../abstractType.ts;```  Add a ```,AbstractType``` here to also import this type.

You will notice that the return statement is red underlined.  The reason is that the return type of the query is not the return type of the ''query'' method.  We will fix this by ```.pipe()```ing the result through a ```map()```.  The mapping expression is:
```
   result => {
     result.hits.hits = result.hits.hits.map(entry => {
       const item = entry._source as ResultType;
       item['_id'] = entry._id;
       return item;
     });
     return result as QueryResult<AbstractType>;
   }
```
