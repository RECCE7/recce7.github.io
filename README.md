# Purpose of Project
Create an easy-install honey pot that any Security Engineer or Systems Administrator can write plug-ins that mimic applications or services running on their network.  The honey pot needs to display critical data to the user in an easy to read and navigate format.  This tool is useful to show what kind of attacks are going on and verify that the actual services the plug-ins represent are guarded appropriately.


# Features and Libraries Used

### Designed to Run on [RaspberryPi Zero](https://www.raspberrypi.org/products/pi-zero/)
Application built around running on a RaspberryPi Zero to make it easy to deploy in any physical environment as well as being cost efficient.  Packages for both [Debian](https://www.debian.org/) and [CentOS](https://www.centos.org/) available to increase flexibility for users.

### Displaying Reporting Data with [Grafana](http://grafana.org/)
HoneyPotR will come packaged with some basic graphs and output configured by default. This allows any user to get up and running with HoneyPotR quickly by providing pre-built graphs so that information can be displayed for real-time attacks immediately.  Grafana also makes it very easy to build your own graphs in order to display any metrics that fit your needs.
![](recce7.github.io/images/grafana1.png)
![](recce7.github.io/images/grafana2.png)

### Using [Docker](https://www.docker.com/) for Builds and CI Management
Docker makes it easy to develop, test, build and manage a Continuous Integration development cycle.  By using Docker we have given our clients the ability to run HoneyPotR on any platform.  If running a Raspberry Pi does no fit your needs, Docker allows you to run our Debian and CentOS packages on any system such as Microsoft and Apple OS X.  Docker can also be used to deploy Grafana on a machine other than the Raspberry Pi.

### Passive OS Fingerprinting with [P0f](http://lcamtuf.coredump.cx/p0f3/)
By passively listening to the TCP Handshake between our honey pot and the attacker we are able to pull the following data:
* Time that the attacker connected and disconnected
* Amount of connections the attacker currently has on HoneyPotR
* Detected remote Operating System, version and accuracy estimated system
* HTTP Client if available, i.e. browser, version and user agent; curl version; wget version
* Network Link
* Distance, amount of hops made to reach HoneyPotR
* Language
* Uptime

### Geoip
Using a simple cURL call to [ipinfo.io](http://ipinfo.io) we are able to gather useful information regarding the location of the attacker and the type of service they may be using.  We are then mapping the information from this call to a world map that is able to display the location of all attacks made on HoneyPotR.  This world map is accessible via an endpoint in HoneyPotR's REST API so that a user is able to pass a days GET parameter to narrow down the amount of points showing on the map.

### REST API
Users will have access to reporting data through a RESTful API.  Attack data resources are represented as JSON Arrays which makes it easy to narrow down and display exactly what information a user may want to access using HTTP GET calls.  This API allows flexibility for our clients to get reporting data however fits best for their environment.  For example if a client already had a GUI for reporting traffic on their network it would be easy for them to make calls to HoneyPotR via the REST API without the need to use Grafana.



# How It Works

## Framework
The framework class is responsible for managing the flow of information throughout the system.  Once the application is started the framework will initiate a _NetworkListener_ that will attempt to complete the TCP handshake of any SYN packets that are addressed to the machine HoneyPotR is running on.  Once this handshake is complete, the framework is able to determine which plug-in is being called based on the port number set in the configuration.  As the plug-in is running and recording data, it is being passed back down to the framework which then passes it on to the _DataManager_.

The framework is also responsible for handling the multi-threaded nature of our application, the safe and graceful shutdown of the application and all current connections.  Multiple threads are needed to ensure that each running plug-in is able to handle connections from multiple attackers at once while being able to simultaneously write the data that is being collected from the malicious actor.  It is also imperative that HoneyPotR is able to gracefully close all running connections so that no data is lost before it is written.  In the case that the application is compromised we need to shutdown gracefully so that we can end the threat without losing data collected up to that point.

## Plug-ins
Each plug-in should be developed to match services running in production as closely as possible.  HoneyPotR comes with Telnet and HTTP servers by default.  Both include interfaces in which the attacker can interact with that will record all of the attackers commands.  By including these plug-ins we give our users an example of how to develop their own services, as well as the ability to collect data while developing their own.

The default plug-ins are also able to handle scans from popular network scanners such as [Nmap](https://nmap.org).  It is common for attacks to start with scans so that the hacker will have an understanding of what services are running on the network.  The ability to capture and record these scans will give us an idea of what kind of information the attacker is looking to gain before going active with his attack.

## Data Management
HoneyPotR uses [Sqlite3](https://www.sqlite.org/) which comes embedded in [Python3](https://www.python.org/download/releases/3.0/).  This made creating a persistant database, that we store locally, easy and lightweight.  All of the data management logic resides in the ```database``` directory which contains modules and classes used by the framework to create, modify and insert data into the database. The resulting database is then used by the report server to query data for reporting.

There are two types of tables defined within the system. User defined tables are those tables that users may create columns in for the storing of specific plug-in data. Non-User defined tables are created by various SQL scripts and should not to be altered by plug-in authors. These Non-User defined tables are the ipInfo, p0f and sessions tables.

The **DataManager** is instantiated by the Framework and has access to the global configuration defined in ```common/config/globals.cfg```. The DataManager makes use of this information to both create non-existent tables and to update table definitions in the database by checking the difference between the current schema and the database configuration defined by the plug-in author.  The DataManager also has access to a **DataValidator** class which checks that the data to be inserted is well formed.  The **DataQueue** class contains a Python queue (FIFO) along with some other methods for standard queue operations.  The DataManager gets items from this queue and inserts them to the database.  This method of writing data allows us the ability to write data from the plug-ins without the need to keep those connections open waiting for the database to unlock from other write operations.  To ensure that we do not lose in information in the queue the Framework ensures the queue is empty before shutting down the data management system.

### DataManager Class
The DataManager Class is called by the framework within its start method. The DataManager creates an instance of **Database** so that it can call the  ```create_default_database``` method and create the initial database or modify the existing database with updates to the user defined configurations between this run and the previous run of the program. The DataManager then creates the DataQueue object that will be used to store database transactions until the thread can process them by acquiring a condition variable.

When the Framework calls the DataManager's ```start``` method the DataManager's ```run``` method will be invoked. This run method will check to see if the queue is empty and then the thread will wait, this will release the lock on the condition variable and block until it is called by notify on that same condition variable.

When the Framework calls DataManager ```insert_data method``` this method will acquire the condition variable and call the Queue's ```insert_into_data_queue``` method and will then ```notify``` the condition variable telling the run method to continue checking the queue for data to insert. Once the queue is empty the run method will again wait for another notification from the ```insert_data``` method. The DataManager also contains a ```shutdown``` method that will shut down the thread.

### Database Class and Table Init Module
The **Database** class will create the directory for storing the database itself as well as creating the database file. It will also execute any scripts for creating Non-User defined tables within that database. Finally it will execute the ```update_schema``` method that will in turn create and/or update any user defined definitions based on plug-in configuration.  This class interacts with the **Table_Init** module to: 
* Create tables and indexes
* Modify existing table definitions
* Verify that defined columns are valid sqlite3 data types

Currently when a user-defined plug-in table is redefined in the plug-in configuration that table will be effectively truncated in order to add or remove the columns defined. In the future it would be good to implement the redefinition in a way that keeps what data is possible to persist between redefinitions.

### DataValidator Class
The DataValidator class is responsible for checking upon insert into the DataQueue that the data that is being inserted is well formed for the database. The DataValidator has access to the schema currently stored in the database. By pulling this schema out of the database it can be used to assist with data validation.  Currently the DataValidator has check methods for the following:
* **check_value_len:**  This method checks that there is exactly 1 table being written to per insert into the data queue
* **check_value_is_dict:** This method checks that the value for insertion is of type dictionary.
* **check_key_in_dict_string:** This method checks that the first and only key in the insert dictionary is a string (this key signifies the table name for insertion)
* **check_key_is_valid_table_name:** This method checks that the key for the insertion dictionary is actually a table name in the database.
* **check_row_value_is_dict:** This method checks that the row values (column: values) of the dictionary provided for insertion is itself a dictionary. For example: {'table1':{'column1':'value1'}}
* **check_all_col_names_strings:** This method checks that all the column names to be inserted are strings.
* **check_all_col_exist:** This method checks that all the column names provided exist in the target table.
* **check_data_types:** _Not implemented yet_ Intended to examine all column values for insertion and to validate against legal regular expressions for the data types we intend to store in the table.

If all checks in the validator do not pass then the value will not be inserted into the DataQueue. Any check errors will be logged to the ```recce7.log``` file explaining why that data was not allowed into the data queue. These errors can be useful for plug-in authors when designing a table for their plug-ins.

### DataQueue Class
The DataQueue class simply creates a python Queue and instantiates the DataValidator class. The method ```insert_into_data_queue``` is called by the DataManager to insert data into the python Queue. This insert_into_data_queue method runs all the checks provided in the data validator prior to inserting the dictionary into the Queue.  The DataQueue also provides helper methods for ```get_next_item``` and ```check_emtpy```.

### Table Insert Module
The Table_Insert module provides methods to insert the data from the DataQueue into the database itself.  Because the name-value pairs in the insert dictionary represent the column names and their values it is necessary to prepare the data for insertion.
The following methods are available in the Table Insert Module:
* ```prepare_data_for_insertion``` orders the values prior to insertion so that the appropriate value goes with the appropriate field in its table
* ```prepare_data_for_insertion``` also pulls the session information from the plug-in table insert request for storage in the sessions table (if provided) for session grouping visibility
* ```prepare_data_for_insertion``` also decides whether or not to insert or update data in the p0f (finger printing table) and whether or not to insert a new row in the ipInfo table
* ```insert_data``` joins the ordered values provided together into a SQL insert statement and inserts data into the sessions table if provided
* ```update_data``` method constructs a SQL update statement based on a where clause provided. This is currently only used in the p0f preparation because we need the most recent fingerprinting stored with the session in that table instead of a new row if the information changes.

### Util Module
The util module contains some common methods used throughout the database code and the configuration for the default columns that are standard for all plug-in tables.  These default columns are pre-pended to all plug-in tables regardless of the extended definitions that the plug-in authors provide therefore are not defined in the plug-ins' configurations file.
The default columns:
* **ID :** A sequential primary key and is managed (not passed in by the plug-in) by the database
* **session :** A field to associate multiple calls or commands to a given plug-in with each other, if this value is not provided then no rows will be inserted into the sessions table
* **eventDateTime :** A field to store the time stamp when the plug-in creates its insert dictionary
* **peerAddress :** A field to store the attackers ip address, provided by the plug-in author
* **localAddress :** A field to store the local address of the machine the plug-in is running on


## Data Reporting
As mentioned in the features section we provide a REST API to allow users to integrate data from our reporting server to their own user interface systems.

### Setup RESTful Report Server
* Navigate to the directory where HoneyPotR was installed
* Edit the `global.cfg` file located in `./config/` so that the Database `path` points to where the database is installed, note the example below:
```
[Database]
path = honeyDB/honeyDB.sqlite
datetime.name = eventDateTime
peerAddress.name = peerAddress
localAddress.name = localAddress
```
* Locate the ReportServer section and edit properties to fit your needs, note the example below:
```
[ReportServer]
reportserver.logName = reportserver.log
reportserver.logLevel = DEBUG
reportserver.host = 192.168.1.16
reportserver.port = 8080
```
* To start the report server navigate back to the installed directory and run ```sudo ./startReportingServer.sh```

### Using REST API
All responses will return content-type: application/json
All examples below assume report server configured to respond on ```localhost``` and port ```8080```

```GET http://<REPORT SERVER HOST>/v1/analytics```
| **Response Headers** | Content-Type --application/json |
| **Status Codes** | 200 OK - Request completed successfully |
| **Request** | http://localhost:8080/v1/analytics |
| **Response** |  |

## Setup Grafana Reporting Graphs and Boards
Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.


# How to Install and Run HunnyPotR
**_Installing by RPM or are we just using git clone and manually setting environment up, then running authbind.sh_**


# Instructions for Developing plug-ins

### Config File Entries
Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.

## API
plug-ins will need to interact with our _Baseplug-in_ so that the data can be inserted properly into the database.  There are two methods that plug-ins must implement.

### do_track(self):
Do track will hold all of the logic in the plug-in that tracks what a user is doing and stores that information in local data structures that follow the data formatting that is described above.  This will differ for each service that is made into a plug-in.  For example we would not expect to see the same kind of attacks hitting a MySQL server as we would for an Apache Web Server.

### get_session(self):
This method allows HoneyPotR to group attacks into sessions based on when an attacker connects until they disconnect.  The reason that this is not handled in the Baseplug-in is that some services natively allow session tracking.  An HTTP server may use cookies or HTTP Headers for tracking sessions while for Telnet we needed to implement a UUID for this.


## Authors and Contributors
Matt @howard-roark, Dina @dsteeve, Charlie @belmontrevenge, Ben @benphillips989, Jesse @jnels124, Randy  @rsoren514, Zach
@zkuhns

## Support or Contact
Have questions or need support? Please contact any of the above contributors at their GitHub pages.