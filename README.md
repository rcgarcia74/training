# Using HDP and HDF to Harness the Power of Social Media
This lab will walk through the collection and analysis of Twitter data leveraging the Hortonworks Data Platform (HDP) and Hortonworks DataFlow (HDF), combining both data in motion and data at rest. While the initial analytics are simple, this pattern enables the building blocks for more advanced use cases. This demo leverages HDFS, YARN, Hive, Solr, and HDF (NiFi).

## Pre-requisites
Where possible, the following pre-requisites should be completed prior to start of the lab.

1.	Provision the Hortonworks Sandbox from the Azure Marketplace.
2.	Obtain a twitter developer account and application keys.
    - http://www.gabfirethemes.com/create-twitter-api-key/
3.	The commands in this lab should be executed as the root user. Obtaining root can be done via sudo, ensure this works after using ssh to access the Sandbox: 

  `sudo su -`

## Change the Ambari Admin Password
If not already completed, change the Ambari admin password. It is recommended to use the following password: Welcome2lab!

### Run ambari-admin-password-reset:


#### Reset the Ambari admin password
`ambari-admin-password-reset`

#### Restart Ambari agent
`service ambari-agent restart`


Log into the Ambari Web Interface at http://<public ip>:8080/ using the new credentials. Note, it may take several minutes for the Ambari Web Interface to become available after restart. 

## Increase Memory Available for Hive, Tez, and YARN
Sandbox is intended to run on minimal hardware requirements, as a result, the Hive, Tez, and YARN configurations are less than ideal out of the box. Run the following script to increase the memory available to Hive, Tez, and YARN.

Run the optimize_hive_config script:

##### Increase Hive, Tez, and YARN memory
`bash <(curl -s https://raw.githubusercontent.com/sakserv/twitter-nifi-lab-setup/master/optimize_hive_config.sh)`.

##Install HDF (NiFi) via an Ambari Service
###Install HDF (NiFi) via Ambari:
Open a browser and navigate to Ambari via <public ip>:8080. Login to Ambari as admin. On the bottom of the panel to the left, navigate to Actions -> Add Service.

![alt text](https://github.com/rcgarcia74/training/blob/master/fig1.png) 


On the Add Service wizard, ensure the checkbox next to NiFi is selected.

![alt text](https://github.com/rcgarcia74/training/blob/master/fig2.png) 

As this is a single node Sandbox, leave the defaults on the Assign Masters panel.

Finally, select Deploy and wait for Ambari to install and start HDF (NiFi). Once the install completes, select Next and then Complete.

##Configure Solr
Solr enables the ability to search across large corpuses of information through specialized indexing techniques. Solr will be combined with Banana, a visualization tool, to create search driven dashboards of Twitter data.

###Download the Banana Dashboard
Banana is a tool for creating dashboards on Solr data. Download and install the pre-built twitter dashboard.


#### Download and install the Banana dashboard
`wget https://raw.githubusercontent.com/abajwa-hw/ambari-nifi-service/master/demofiles/default.json -O /opt/lucidworks-hdpsearch/solr/server/solr-webapp/webapp/banana/app/dashboards/default.json`

###Download the Solr Configuration
Due to the timestamp used by Twitter, it is necessary to update the Solr configuration to support this additional timestamp format.


#### Download and install the updated Solr configuration
`wget https://raw.githubusercontent.com/sakserv/twitter-nifi-solrconfig/master/solrconfig.xml -O /opt/lucidworks-hdpsearch/solr/server/solr/configsets/data_driven_schema_configs/conf/solrconfig.xml`

### Start Solr
Start Solr in SolrCloud mode.


#### Start Solr
`export JAVA_HOME=/usr/lib/jvm/java-1.7.0-openjdk.x86_64`

`/opt/lucidworks-hdpsearch/solr/bin/solr start -c -z localhost:2181`

###Create the Solr Collection
A Solr collection is the index and configuration associated with the data that will be stored in the search index. Also specified are the number of replicas and shards, as this is a single node cluster, only a single replica and shard are used.


##### Create the Tweets Collection
`/opt/lucidworks-hdpsearch/solr/bin/solr create -c tweets -d data_driven_schema_configs -s 1 -rf 1`

## Import the Twitter HDF (NiFi) Template
HDF (NiFi) has a powerful templating system to allow for easy sharing and consumption of dataflows. The following imports the Twitter dataflow template and starts the collection of Tweets into Solr and HDFS.

### Download the Twitter dataflow template
The Twitter dataflow template should be downloaded to the workstation accessing NiFi at the following link.

- https://raw.githubusercontent.com/abajwa-hw/ambari-nifi-service/master/demofiles/Twitter_Dashboard.xml

### Import the Template
Navigate to the HDF (NiFi) web interface at http://<public ip>:9090/nifi/ via a web browser.

Import the template by selecting the Templates administration panel found in the upper right of the HDF (NiFi) web interface.

![alt text](https://github.com/rcgarcia74/training/blob/master/fig3.png)

Click the Browse button and navigate to where the Twitter_Dashboard.xml file was downloaded. Click the X in the right hand corner to return to the canvas.

![alt text](https://github.com/rcgarcia74/training/blob/master/fig4.png)

### Instantiate the Template
Drag and drop the template artifact onto the canvas.

![alt text](https://github.com/rcgarcia74/training/blob/master/fig5.png) 

Select Twitter Dashboard and click Add

![alt text](https://github.com/rcgarcia74/training/blob/master/fig6.png) 

This will create a HDF (NiFi) Process Group, double click to drill down into the flow.

### Configure the Twitter Grab Garden Hose Processor
Next, configure the Twitter Grab Garden Hose Processer with the Twitter API information. Right click the processor and select Configure.

![alt text](https://github.com/rcgarcia74/training/blob/master/fig7.png)

Fill in the properties:

![alt text](https://github.com/rcgarcia74/training/blob/master/fig8.png) 

### Review the remaining Processors
Review the other processors and modify properties as needed:
- EvaluateJsonPath: Pulls out attributes of tweets
- RouteonAttribute: Ensures only tweets with non-empty messages are processed
- PutSolrContentStream: Writes the selected attributes to Solr. In this case, assuming Solr is running in cloud mode with a collection 'tweets'
- ReplaceText: Formats each tweet as pipe (|) delimited line entry e.g. tweet_id|unixtime|humantime|user_handle|message|full_tweet
- MergeContent: Merges tweets into a single file (either 20 tweets or 120s, whichever comes first) to avoid having a large number of small files in HDFS. These values can be configured.
- PutFile: writes tweets to local disk under /tmp/tweets/
- PutHDFS: writes tweets to HDFS under /tmp/tweets_staging

### Start the Flow
If the processors are correctly configured, all processors should have a red stop symbol in the upper left. If a yellow caution symbol is present, this indicates required configuration is missing.

Start the flow by selecting an empty area of the canvas and clicking the green start icon at the top the interface.

![alt text](https://github.com/rcgarcia74/training/blob/master/fig9.png)  

## Validate the Results
Now that the Twitter dataflow is running, the matching tweets will be persisted to Solr and HDFS. Validate this is successfully occurring.

### Banana Dashboard
Navigate to the banana dashboard and ensue the dashboard is populated.
- http://<public ip>:8983/solr/banana/index.html#/dashboard

![alt text](https://github.com/rcgarcia74/training/blob/master/fig10.png)   

### View Files in HDFS
Tweets appear under /tmp/tweets_staging dir in HDFS. You can see this via Files view in Ambari:

![alt text](https://github.com/rcgarcia74/training/blob/master/fig11.png)   

## Analyze Tweets with Hive
After persisting the tweets in HDFS, the opportunity to perform analytics against the data is nearly unlimited. Leveraging Hive, SQL can be used to analyze the Tweets. The following sections will create the Hive table and run some basic queries against the tweets.

To access Hive, ssh into the server  and execute the hive CLI tool as the hdfs user:

![alt text](https://github.com/rcgarcia74/training/blob/master/fig12.png)   

### Create the Hive Table
Create the hive table to allow querying the tweets.


##### Create the tweets table

`create external table tweets(

  tweet_id bigint, 
  
  created_unixtime bigint, 
  
  created_time string, 
  
  displayname string, 
  
  msg string,
  
  fulltext string
  
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '|'
STORED AS TEXTFILE
LOCATION '/tmp/tweets_staging';`

### Display the top 10 twitter handles by tweet volume
Query the tweets table, grouping tweet count by twitter handle and display only the top 10 in descending order.


#### Display the top 10 twitter handles by tweet volume
`select displayname, count(*) as tweetcount from tweets group by displayname order by tweetcount desc limit 10;`


