# Curam Batch Process Development Guide

A batch process in Curam allows a user to automate certain processes that may need to be run at scheduled interviews.

More information can be found in the [IBM Curam Docs](https://www.ibm.com/support/pages/ibm-c%C3%BAram-social-program-management-pdf-library), specifically: 
- Curam Batch Processing Guide 
- Curam Batch Streaming Developers Guide
- Curam Batch Performance Mechanisms

At a high level, the batch process takes a list of records, splits these records into several chunks, then records in a chunk are processed using one or more streams. Once the batch process is finished running, a report may be generated.

## Relevant Models and Associations

Each batch process requires three «struct» classes to be modeled:

### «struct» [ID]BatchProcessKey
This can be used to pass arguments to the batch process before running (note: only accepted date format is yyyymmdd)
- ``«default» instanceID`` (dd: BATCH_PROCESS_INSTANCE_ID) - Tells curam which batch process to run. Will be set to name given to the batch in the **BATCHPROCESSNAME.ctx**
- ``«default» processingDate`` (dd: CURAM_DATE)

### «struct» [PROJECT NAME]Result
This struct will hold information about each record processed. If there is any data you want to add to batch reports. Some example of attributes that may be useful in this struct include:
- ``«default» concernRoleID`` (dd: INTERNAL_ID) - Hold the concern role ID for a given processed record
- ``«default» casesSkippedCount`` (dd: BATCH_PROCESS_NUMBER_OF_CHUNKS) - Track if an error caused a given record to be skipped; Can be summed to report total number of errors
- ``«default» errorMessage`` (dd: custom DD that maps to SVR_STRING w/ maximum_size=512) - Store error message for failed records so they can be added to a batch report
- ``«default» casesProcessed`` (dd: BATCH_PROCESS_NUMBER_OF_CHUNKS) - Useful if you would like to track the total number of records processed

### «struct» [PROJECT NAME]ResultList
This struct holds all Results generated during processing
- Aggregates with «struct» [PROJECT NAME]Result
- **«aggregation»** relationship has a 1 ([PROJECT NAME]ResultList) to many ([PROJECT NAME]Result) cardinality 

Each batch process requires two «process» classes to be modeled:

### «process» [PROJECT NAME]Batch
This models your chunker. Contains the following modeled operataions:
- ``«batch» process()``
    - Arguments: [ID]BatchProcessKey struct
    - Returns: void
- ``«default» decodeProcessChunkResult()``
    - Arguments: BATCH_PROCESS_CHUNK_RESULT_SUMMARY
    - Returns: [PROJECT NAME]ResultList struct
- ``«default» sendBatchReport()``
    - Arguments: BATCH_DESCRIPTION, batchProcess (shadow: Dtls)
    - Returns: BatchProcessChunk (for processed records; shadow: DtlsList), BatchProcessChunk (for unprocessed records; shadow: DtlsList)
- ``«default» doExtraProcessing()``
    - Arguments: BatchProcessStreamKey, BATCH_PROCESS_PARAM
    - Returns: BatchProcessingResult


### «process» [PROJECT NAME]BatchStream
This models your streamer. Contains the following modeled operataions:
- ``«batch» process()``
    - Arguments: BatchProcessStreamKey
    - Returns: void
- ``«default» getChunkResult()``
    - Arguments: RECORD_COUNT
    - Returns: BATCH_PROCESS_CHUNK_RESULT_SUMMARY
- ``«default» processRecord()``
    - Arguments: BatchProcessingKey struct, BatchProcessingID
    - Returns: BatchProcessingSkippedRecord
- ``«default» processSkippedCases()``
    - Arguments:BatchProcessingSkippedRecordList
    - Returns: void

## Relevant Configurations

### CT_BatchProcessName.ctx
In `component/custom/codetable/` add this file:
```
<?xml version="1.0" encoding="UTF-8"?>
<codetables package="curam.codetable">
  <codetable
    java_identifier="BATCHPROCESSNAME"
    name="BatchProcessChunkName"
  >
    <code
      default="true"
      java_identifier="<IDENTIFIER FOR TABLE>"
      status="ENABLED"
      value="<ID IN TABLE, prefix with RM>"
    >
      <locale
        language="en"
        sort_order="0"
      >
        <description>[A DESCRIPTION OF THE TABLE]</description>
        <annotation/>
      </locale>
    </code>

  </codetable>
</codetables>

```


### Application.prx
In `component/properties/` add this file:
```
<?xml version="1.0" encoding="UTF-8"?>
  <root>
  	<property name="curam.custom.batch.[PACKAGE NAME].chunksize" dynamic="yes" constant="CUSTOM_BATCH_AGE_RANGE_CHUNK_SIZE">
      <type>INT32</type>
      <value>1</value>
      <default-value>1</default-value>
      <category>APP_BATCH</category>
      <locales>
        <locale language="en">
          <display-name>curam.custom.batch.[PACKAGE NAME].chunksize</display-name>
          <description>This parameter controls the number of records in each chunk for PDC Closure batch.</description>
        </locale>
      </locales>
    </property>

    <property name="curam.custom.batch.[PACKAGE NAME].dontrunstream" dynamic="yes" constant="CUSTOM_BATCH_AGE_RANGE_DONT_RUN_STREAM">
      <type>BOOLEAN</type>
      <value>true</value>
      <default-value>true</default-value>
      <category>APP_BATCH</category>
      <locales>
        <locale language="en">
          <display-name>curam.custom.batch.[PACKAGE NAME].dontrunstream</display-name>
          <description>This parameter controls whether or not a streamer is triggered automatically from chunker after chunker picks the records for processing.</description>
        </locale>
      </locales>
    </property>

    <property name="curam.custom.batch.[PACKAGE NAME].startchunkkey" dynamic="yes" constant="CUSTOM_BATCH_AGE_RANGE_START_CHUNKKEY">
      <type>INT64</type>
      <value>1</value>
      <default-value>1</default-value>
      <category>APP_BATCH</category>
      <locales>
        <locale language="en">
          <display-name>curam.custom.batch.[PACKAGE NAME].startchunkkey</display-name>
          <description>Parameter specifies the key value for the first chunk key for RM Age Range Batch.</description>
        </locale>
      </locales>
    </property>

    <property name="curam.custom.batch.[PACKAGE NAME].unprocessedchunkreadwait" dynamic="yes" constant="CUSTOM_BATCH_AGE_RANGE_CHUNK_READ_WAIT">
      <type>INT32</type>
      <value>1000</value>
      <default-value>1000</default-value>
      <category>APP_BATCH</category>
      <locales>
        <locale language="en">
          <display-name>curam.custom.batch.[PACKAGE NAME].unprocessedchunkreadwait</display-name>
          <description>Read wait time (in milliseconds) for unprocessed chunks for RM Age Range Batch.</description>
        </locale>
      </locales>
    </property>

    <property name="curam.custom.batch.[PACKAGE NAME].processunprocessedchunks" dynamic="yes" constant="CUSTOM_BATCH_AGE_RANGE_PROCESS_UNPROCESSED_CHUNKS">
      <type>BOOLEAN</type>
      <value>false</value>
      <default-value>false</default-value>
      <category>APP_BATCH</category>
      <locales>
        <locale language="en">
          <display-name>curam.custom.batch.[PACKAGE NAME].processunprocessedchunks</display-name>
          <description>Parameter specifies if the batch should attempt to reprocess unprocessed chunks for Age Range Batch.</description>
        </locale>
      </locales>
    </property>
  </root>

```

Build the database after adding these files, so the changes will be picked up in your application.

## Relevant Java Classes
Once the relevant models/tables have been created you're ready to write some Java. Our convention is to put the Java files in `/components/custom/source/curam/<Project Package>/batch/impl`

### [PROJECT]Batch.java
Batch is where your chunker is implemented. This class must extend the Batch process modeled earlier: `curam.[custom package].base.[PROJECT]Batch`. You will implement the following methods:

- `public void process([ID]BatchProcessKey key)`
    - This is where your chunker is set up
    - Largely boilerplate that will be reused from project to project

    ```
    @Override
    public void process([ID]BatchProcessKey key) throws AppException, InformationalException {

        final [PROJECT]BatchWrapper chunkerWrapper = new [PROJECT]BatchWrapper(this);
        final [PROJECT]BatchStreamWrapper streamWrapper = new [PROJECT]BatchStreamWrapper(new [PROJECT]BatchStream());

        BatchProcessingIDList batchProcessingIdList = new BatchProcessingIDList();

        SecurityImplementationFactory.register();

        final ChunkMainParameters chunkMainParameters = getChunkMainParameters();
        // ^^^ Utility method for setting chunkParameters from environment variables
        batchProcessingIdList = getBatchProcessingIDList(key);
        // ^^^ Utility method to get the list of ids for the records to be processed

        totalRecordsToBeProcessed = batchProcessingIdList.dtls.size();

        batchStreamHelper = new BatchStreamHelper();
        batchStreamHelper.setStartTime();

        loadPropertyValues(key); //Utility method for retrieving arguments from key

        batchStreamHelper.runChunkMain(BATCHPROCESSNAME.[JAVA IDENTIFIER FROM .CTX FILE], key, chunkerWrapper, batchProcessingIdList,
        chunkMainParameters, streamWrapper);

    }

    ```
- `private ChunkMainParameters getChunkMainParameters()`
    - Used to set chunk configurations
    - Set using the environment variables created in `Application.prx`

    ```
    private ChunkMainParameters getChunkMainParameters() {
        // Instantiate chunk parameters object
        ChunkMainParameters mainParameters = new ChunkMainParameters();

        // Set parameter properties on object
        mainParameters.chunkSize = Configuration.getIntProperty(EnvVars.CUSTOM_BATCH_AGE_RANGE_CHUNK_SIZE);
        mainParameters.dontRunStream = Configuration.getBooleanProperty(EnvVars.CUSTOM_BATCH_AGE_RANGE_DONT_RUN_STREAM);
        mainParameters.startChunkKey = Long.parseLong(Configuration.getProperty(EnvVars.CUSTOM_BATCH_AGE_RANGE_START_CHUNKKEY));
        mainParameters.unProcessedChunkReadWait = Configuration.getIntProperty(EnvVars.CUSTOM_BATCH_AGE_RANGE_CHUNK_READ_WAIT);
        mainParameters.processUnProcessedChunks = Configuration.getBooleanProperty(EnvVars.CUSTOM_BATCH_AGE_RANGE_PROCESS_UNPROCESSED_CHUNKS);

        // return parameters object to caller (process())
        return mainParameters;
    }
    ```
- `private BatchProcessingIDList getBatchProcessingIDList([ID]BatchProcessKey key)`
    - Used to get the list of IDs for records that will be processed in the batch

    ``` 
    private BatchProcessingIDList getBatchProcessingIDList([ID]BatchProcessKey key)
    throws AppException, InformationalException {

    BatchProcessingIDList idList = new BatchProcessingIDList();
    [Entity] [entity] = [Entity]Factory.newInstance();
    [Entity]DtlsList [entity]DtlsList = [entity].readAll();
    [entity]DtlsList.dtls.forEach([entity]Dtls -> {
      BatchProcessingID batchProcessingId = new BatchProcessingID();
      batchProcessingId.recordID = [entity]Dtls.concernRoleID;
      idList.dtls.add(batchProcessingId);
    });
    return idList;
  }
    ```
- `public void sendBatchReport(String instanceID, BatchProcessDtls batchProcessDtls, BatchProcessChunkDtlsList processedBatchProcessChunkDtlsList, BatchProcessChunkDtlsList unprocessedBatchProcessChunkDtlsList)`
    - Called when streamers finish running to send batch report
    - Loops through the processed chunk details list and sends each detail string to `decodeProcessChunkResult()` to translate each string into a meaningful message
    - Example in *[Developing Batch Reports](/batchProcessDevelopment/BatchProcessDevelopmentGuide/#TK)* section

- `public [PROJECT NAME]ResultList decodeProcessChunkResult(String key)`
    - This method uses the key string generated by the streamers to populate batch reports
    - Example in *[Developing Batch Reports](/batchProcessDevelopment/BatchProcessDevelopmentGuide/#TK)* section

- `public BatchProcessingResult doExtraProcessing(BatchProcessStreamKey keyParam, Blob param2)`
    - Used to do additional processing after batch has run(?)
    - Not used often, probably don't need to worry about it too much

- Some additional utility methods are used in RM projects. Worth knowing about but are not required (will be discussed more in *[Developing Batch Reports](/batchProcessDevelopment/BatchProcessDevelopmentGuide/#TK)*)
    - `loadPropertyValues([ID]BatchProcessKey key)`
        - Sets values needed by other parts of the chunker (i.e. names for the generated log files)
    - `initializeFilesAndCount()`
        - Initializes log files and creates report objects
    - `updateReportsWithCounts()`
        - Writes data to log files

### [PROJECT]BatchStream.java

### [PROJECT]BatchWrapper.java

### [PROJECT]BatchStreamWrapper.java

## Developing Batch Reports