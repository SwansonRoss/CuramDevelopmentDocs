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
    - Example in *[Developing Batch Reports](/batchDocumentation/batchProcessDevelopment/BatchProcessDevelopmentGuide#developing-batch-reports)* section

- `public [PROJECT NAME]ResultList decodeProcessChunkResult(String key)`
    - This method uses the key string generated by the streamers to populate batch reports
    - Example in *[Developing Batch Reports](/batchDocumentation/batchProcessDevelopment/BatchProcessDevelopmentGuide#developing-batch-reports)* section

- `public BatchProcessingResult doExtraProcessing(BatchProcessStreamKey keyParam, Blob param2)`
    - Used to do additional processing after batch has run(?)
    - Not used often, probably don't need to worry about it too much

- Some additional utility methods are used in RM projects. Worth knowing about but are not required (will be discussed more in *[Developing Batch Reports](/batchDocumentation/batchProcessDevelopment/BatchProcessDevelopmentGuide#developing-batch-reports)*)
    - `loadPropertyValues([ID]BatchProcessKey key)`
        - Sets values needed by other parts of the chunker (i.e. names for the generated log files)
    - `initializeFilesAndCount()`
        - Initializes log files and creates report objects
    - `updateReportsWithCounts()`
        - Writes data to log files

### [PROJECT]BatchStream.java
Stream is where the streamer is implemented. This class must extend the BatchStream process modeled earlier: `curam.[custom package].base.[PROJECT]BatchStream`. You will implement the following methods:

- The constructor
    - The stream class requires you to inject a BatchStreamHelper object. 
    ```
    import com.google.inject.Inject;
    import curam.util.persistence.GuiceWrapper;
    ...
    @Inject
    private BatchStreamHelper batchStreamHelper;

    public [PROJECT]BatchStream() {
      GuiceWrapper.getInjector().injectMembers(this);
    }
    ```

- `public void process(BatchProcessStreamKey key)`
    - Mostly boilerplate that will be reused from project to project
    - Tells the batchStreamHelper to start the stream
    ```
    @Override
    public void process(BatchProcessStreamKey key) throws AppException, InformationalException {

      if (key.instanceID.isEmpty()) {
        key.instanceID = BATCHPROCESSNAME.[JAVA IDENTIFIER FROM .CTX];
      }

      [PROJECT]BatchStreamWrapper streamObj = new [PROJECT]BatchStreamWrapper(this);
      this.batchStreamHelper.runStream(key, streamObj);

    }
    ```
- `BatchProcessingSkippedRecord processRecord([ID]BatchProcessKey key, BatchProcessingID batchProcessingID)`
    - This is where the logic for processing an individual record will go
    ```
    @Override
    public BatchProcessingSkippedRecord processRecord([ID]BatchProcessKey key,   BatchProcessingID batchProcessingID) {

      BatchProcessingSkippedRecord skippedRecord = null;
      this.recordResult = new [PROJECT]Result();
      this.recordResult.concernRoleID = batchProcessingID.recordID;

      try {
        Date batchProcessingDate = Date.getCurrentDate();

        if (key.processingDate != null && key.processingDate != Date.kZeroDate) {
          batchProcessingDate = key.processingDate;
        }

        /* LOGIC FOR PROCESSING RECORD GOES HERE */

        this.recordResult.casesProcessed = CuramConst.gkOne;
        // ^^^ records that a record has be processed successfully, used later for reports

      } catch (Exception e) {
        // Logic to handle errors in processing a record
        skippedRecord = new BatchProcessingSkippedRecord();
        skippedRecord.recordID = batchProcessingID.recordID;
        skippedRecord.errorMessage = e.getMessage();
      }

      this.resultList.dtls.add(recordResult);

      return skippedRecord;
    }
    ```

- `public void processSkippedCases(BatchProcessingSkippedRecordList skippedRecordList)`
    - Handles all skipped records once streamer is done processing chunk
    ```
      @Override
  public void processSkippedCases(BatchProcessingSkippedRecordList skippedRecordList)
    throws AppException, InformationalException {

      if (skippedRecordList.dtls.size() > 0) {
        // iterate through all skipped records
        for (BatchProcessingSkippedRecord dtls : skippedRecordList.dtls) {
          [PROJECT]Result recordResult = new   [PROJECT]Result();

          // store recordID, error message and update casesSkippedCount to 1
          recordResult.concernRoleID = dtls.recordID;
          recordResult.errorMessage = dtls.errorMessage;
          recordResult.casesSkippedCount = CuramConst.gkOne;
          recordResult.casesUpdated = CuramConst.gkZero;

          // store results
          this.resultList.dtls.add(recordResult);
        }
      }

    }
    ```

- `public String getChunkResult(int recordCount)`
    - Generates the strings that will be used to populate the Batch Report once everything's been processed
    ```
    @Override
    public String getChunkResult(int recordCount) throws AppException,   InformationalException {

      String chunkResult = new String();

      for (int i = 0; i < this.resultList.dtls.size(); i++) {

        [PROJECT]Result dtls = this.resultList.dtls.get(i);
        // Serialize the result elements of this record into a string and
        // add it to the final return
        chunkResult += dtls.concernRoleID + RMBatchConstants.kNewElement + dtls.  errorMessage
          + RMBatchConstants.kNewElement + dtls.casesProcessed + RMBatchConstants.  kNewElement + dtls.casesSkippedCount
          + RMBatchConstants.kNewElement + dtls.casesAdded + RMBatchConstants.kNewElement   + dtls.casesUpdated;
        //Generates a comma sepated list that will be decoded in the Batch object
        // for this case "[concernRoleID],[Error message],[casesProcessed],[casesSkipped],[casesAdded],[casesUpdated]"
        // Which would look something like: "101,,1,,,1"
        /* Which translates to:
        * ConcernRole 101
        * No error messages (because this is an empty list entry - could also be 0)
        * case was processed
        * case was not skipped
        * case was not added
        * case was updated
        */

        // If this is not the last item in the record list then a item
        // separator needs to be added to the return string.
        if (i != this.resultList.dtls.size() - 1) {
          chunkResult += RMBatchConstants.kNewItem;
        }
      }

      // clear results of this chunk
      this.resultList = new [PROJECT]ResultList();
      return chunkResult;
    }

    ```
### [PROJECT]BatchWrapper.java
The batch wrapper allows the chunker to be called outside of Curam. It's mostly boilerplate that you just need to update with project specifics. It looks like this:
```
package curam.[PACKAGE].impl;

import curam.core.impl.BatchMain;
import curam.core.struct.BatchProcessChunkDtlsList;
import curam.core.struct.BatchProcessDtls;
import curam.core.struct.BatchProcessStreamKey;
import curam.core.struct.BatchProcessingResult;
import curam.util.exception.AppException;
import curam.util.exception.InformationalException;
import curam.util.type.Blob;

public class [PROJECT]BatchWrapper implements BatchMain {

  private [PROJECT]Batch [PROJECT]BatchObj;

  
  public [PROJECT]BatchWrapper([PROJECT]Batch [PROJECT]Batch) {

    [PROJECT]BatchObj = [PROJECT]Batch;
  }

  /**
   * OOTB hook method to add custom extra processing if required. This batch do not have any such requirement.
   */
  @Override
  public BatchProcessingResult doExtraProcessing(BatchProcessStreamKey batchProcessStreamKey,
    Blob batchProcessParameters) throws AppException, InformationalException {

    return null;
  }

  @Override
  public void sendBatchReport(String instanceID, BatchProcessDtls batchProcessDtls,
    BatchProcessChunkDtlsList processedBatchProcessChunkDtlsList,
    BatchProcessChunkDtlsList unprocessedBatchProcessChunkDtlsList) throws AppException, InformationalException {

    [PROJECT]BatchObj.sendBatchReport(instanceID, batchProcessDtls, processedBatchProcessChunkDtlsList,
      unprocessedBatchProcessChunkDtlsList);

  }

}
```

### [PROJECT]BatchStreamWrapper.java
Same deal as the batch wrapper, but for the streamer.
```
package curam.[PACKAGE].impl;

import curam.core.impl.BatchStream;
import curam.core.struct.BatchProcessingID;
import curam.core.struct.BatchProcessingSkippedRecord;
import curam.core.struct.BatchProcessingSkippedRecordList;
import curam.[PACKAGE].struct.[ID]BatchProcessKey;
import curam.util.exception.AppException;
import curam.util.exception.InformationalException;

public class [PROJECT]BatchStreamWrapper implements BatchStream {

  private [PROJECT]BatchStream [PROJECT]BatchStreamObj;

  public [PROJECT]BatchStreamWrapper([PROJECT]BatchStream [PROJECT]BatchStream) {

    [PROJECT]BatchStreamObj = [PROJECT]BatchStream;
  }

  @Override
  public BatchProcessingSkippedRecord processRecord(BatchProcessingID batchProcessingID, Object parameters)
    throws AppException, InformationalException {

    return [PROJECT]BatchStreamObj.processRecord(([ID]BatchProcessKey) parameters, batchProcessingID);
  }

  @Override
  public void processSkippedCases(BatchProcessingSkippedRecordList batchProcessingSkippedRecordList)
    throws AppException, InformationalException {

    [PROJECT]BatchStreamObj.processSkippedCases(batchProcessingSkippedRecordList);

  }

  @Override
  public String getChunkResult(int skippedCasesCount) throws AppException, InformationalException {

    return [PROJECT]BatchStreamObj.getChunkResult(skippedCasesCount);
  }

}
```

## Developing Batch Reports
Once all chunks have been processed, the Batch class can generate reports on how the batch run went. These reports can be emailed or, as shown below, saved to logs. I will walk through my best approximation of how this process works.

There will be references to `RMBatchFile`, `RMBatchFileKey`, and `RMBatchConstants`. These are just utilities for creating the report files.

`private void loadPropertyValues([ID]BatchProcessKey key)`
- Sets values needed by other parts of the chunker (i.e. names for the generated log files)
- Calls `initializeFilesAndCount()` to generate log file objects
```
private void loadPropertyValues(RMBatchProcessKey key) throws AppException {

    batchStartDateTime = DateTime.getCurrentDateTime();
    batchParameters = new StringBuilder();

    if (key.instanceID.isEmpty()) {
      key.instanceID = BATCHPROCESSNAME.[JAVA IDENTIFIER FROM .CTX];
    }

    if (key.processingDate != null && key.processingDate != Date.kZeroDate) {
      key.processingDate = Date.getCurrentDate();
    }
    else {
      batchParameters.append("processingDate=");
      batchParameters.append(key.processingDate);
    }

    DateFormat dateTimeFormat = new SimpleDateFormat("MM.dd.yyyy HH.mm.ss", Locale.US);

    Calendar cal = batchStartDateTime.getCalendar();
    String timeStamp = dateTimeFormat.format(cal.getTime());

    controlReportFileName =
      BATCHPROCESSNAMEEntry.[JAVA IDENTIFIER FROM .CTX].toUserLocaleString() + " (Control Report " + timeStamp + ").log";

    errorReportFileName =
      BATCHPROCESSNAMEEntry.[JAVA IDENTIFIER FROM .CTX].toUserLocaleString() + " (Error Report " + timeStamp + ").log";

    // Initialize files
    initializeFilesAndCount();
}
```

`private void initializeFilesAndCount()`
- Initializes log files and creates report objects
```
private void initializeFilesAndCount() throws AppException {

    errorRecordsMap = new HashMap<Long, String>();

    // Control Report
    RMBatchFileKey controlReportKey = new RMBatchFileKey();
    controlReportKey.setCreateImmediately(true);
    controlReportKey.setBatchStartDateTime(batchStartDateTime);
    controlReportKey.setFileName(controlReportFileName);
    controlReportKey.setFileTypeCode("PRG_LOG");
    controlReportKey.setJobName(BATCHPROCESSNAMEEntry.[JAVA IDENTIFIER FROM .CTX].toUserLocaleString());
    controlReportKey.setParameters(batchParameters.toString());
    controlReport = new RMBatchFile(controlReportKey);

    // Error Report
    RMBatchFileKey errorReportKey = new RMBatchFileKey();
    errorReportKey.setCreateImmediately(false);
    errorReportKey.setBatchStartDateTime(batchStartDateTime);
    errorReportKey.setFileName(errorReportFileName);
    errorReportKey.setFileTypeCode("ERR_LOG");
    errorReportKey.setJobName(BATCHPROCESSNAMEEntry.[JAVA IDENTIFIER FROM .CTX].toUserLocaleString());
    errorReportKey.setParameters(batchParameters.toString());
    errorReport = new RMBatchFile(errorReportKey);

}
```

Streamers run -- processing records and generating ResultLists

`public void sendBatchReport(String instanceID, BatchProcessDtls batchProcessDtls, BatchProcessChunkDtlsList processedBatchProcessChunkDtlsList, BatchProcessChunkDtlsList unprocessedBatchProcessChunkDtlsList)`
- Called when streamers finish running to send batch report
- Loops through the processed chunk details list and sends each detail string to `decodeProcessChunkResult()` to translate each string into a meaningful message
- Finally, calls `updateReportWithCounts()` to populate reports with data
```
 @Override
  public void sendBatchReport(String instanceID, BatchProcessDtls batchProcessDtls,
    BatchProcessChunkDtlsList processedBatchProcessChunkDtlsList,
    BatchProcessChunkDtlsList unprocessedBatchProcessChunkDtlsList) throws AppException, InformationalException {

    // loop through all processed chunks and decode the results
    for (BatchProcessChunkDtls tempChunkDtls : processedBatchProcessChunkDtlsList.dtls) {
      try {

        [PROJECT]ResultList tempResultList = decodeProcessChunkResult(tempChunkDtls.resultSummary);

        // loop through the chunk result list to get down to the
        for ([PROJECT]Result tempResultDetails : tempResultList.dtls) {
          if (!RMBatchConstants.EMPTY_STRING.equals(tempResultDetails.errorMessage)) {
            // Error occurred, add the errors to hash map
            errorRecordsMap.put(tempResultDetails.concernRoleID, tempResultDetails.errorMessage);
          }
        }

      } catch (Exception e) {
        StringWriter errors = new StringWriter();
        e.printStackTrace(new PrintWriter(errors));
      }
    }

    updateReportsWithCounts();

}
```
`public [PROJECT NAME]ResultList decodeProcessChunkResult(String key)`
- This method uses the key string generated by the streamers to populate batch reports
- Called from a foreach loop in `sendBatchReport()` for every processed chunk's details
```
@Override
public [PROJECT]ResultList decodeProcessChunkResult(String key) throws AppException, InformationalException {

    [PROJECT]ResultList resultList = new [PROJECT]ResultList();

    // init for 1 item
    String[] items = new String[1];

    if (!StringUtil.isNullOrEmpty(key)) {

      // check str for more than one item
      if (key.contains(RMBatchConstants.kNewItem)) {
        items = key.split(RMBatchConstants.kNewItemDelim);
      }
      else {
        items[0] = key;
      }

      // iterate through items
      for (int i = 0; i < items.length; i++) {
        [PROJECT]Result recordResult = new [PROJECT]Result();

        // split items based on getChunkResult encoding
        String[] elements = items[i].split(RMBatchConstants.kNewElementDelim, -1);

        /* 
        * the recordResult.* values correspond to the values set in the chunkResult 
        * string in the BatchStream class's getChunkResult() method.
        * These values will likely be different for each batch implementation
        */
        recordResult.concernRoleID = Long.valueOf(elements[0]);
        recordResult.errorMessage = elements[1];
        recordResult.casesProcessed = Integer.valueOf(elements[2]);
        recordResult.casesSkippedCount = Integer.valueOf(elements[3]);
        recordResult.casesAdded = Integer.valueOf(elements[4]);
        recordResult.casesUpdated = Integer.valueOf(elements[5]);

        totalRecordsAdded += recordResult.casesAdded;
        totalRecordsUpdated += recordResult.casesUpdated;

        if (recordResult.casesUpdated == 1) {
          updatedConcernRoles.add(elements[0]);
        }

        // add to return list
        resultList.dtls.add(recordResult);
      }
    }

    return resultList;
}
```

`updateReportWIthCounts()`
- Writes data to log files
- Method will look different for each batch implementation to match that implementation's reporting requirements.
- Basically, just a bunch of lines being written to a file
```
private void updateReportsWithCounts() throws AppException {

    DateTime batchFinishDateTime = DateTime.getCurrentDateTime();

    try {

      // add batch end time to control report
      controlReport.writePrintlnToFile(RMBatchConstants.BATCH_FINISH_TIME + batchFinishDateTime);
      controlReport.writePrintlnToFile(CuramConst.gkEmpty);

      // update control report with counts
      controlReport.writePrintlnToFile(RM_AGE_RANGE_BATCH_TOTAL_PROCESSED + (totalRecordsToBeProcessed));

      controlReport.writePrintlnToFile(RM_AGE_RANGE_BATCH_TOTAL_ADDED + (totalRecordsAdded));
      controlReport.writePrintlnToFile(RM_AGE_RANGE_BATCH_TOTAL_UPDATED + (totalRecordsUpdated));
      if (totalRecordsUpdated > 0) {
        controlReport.writePrintlnToFile(CuramConst.gkEmpty);
        controlReport.writePrintlnToFile(RM_AGE_RANGE_BATCH_USERS_UPDATED);
        for (String concernRoleID : updatedConcernRoles) {
          controlReport.writePrintlnToFile(concernRoleID);
        }
      }

      // if erroredRecords found
      if (!errorRecordsMap.isEmpty()) {

        errorReport.writePrintlnToFile(CuramConst.gkEmpty);
        errorReport.writePrintlnToFile(RMBatchConstants.BATCH_FINISH_TIME + batchFinishDateTime);

        controlReport.writePrintlnToFile(CuramConst.gkEmpty);
        controlReport.writePrintlnToFile(RMBatchConstants.NUMBER_FAILED_RECORDS + errorRecordsMap.size());

        errorReport.writePrintlnToFile(RMBatchConstants.FAILURE_COMMENT);
        errorReport.writePrintlnToFile(RMBatchFile.SECTION_SEPARATOR);

        // update the errorReport with failed records from the HashMap
        for (Long batchProcessingID : errorRecordsMap.keySet()) {

          errorReport.writePrintlnToFile("Record processing failed for case: " + batchProcessingID);
          errorReport.writePrintlnToFile(errorRecordsMap.get(batchProcessingID));
          errorReport.writePrintlnToFile(CuramConst.gkEmpty);
        }
      }
      else {
        controlReport.writePrintlnToFile(CuramConst.gkEmpty);
        controlReport.writePrintlnToFile(RMBatchConstants.NO_FAILURE_COMMENT);
      }

    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      controlReport.close();
      errorReport.close();
    }
}
```
