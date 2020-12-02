# Sparkify Apache Cassandra Data Engineering Project

Sparkify has asked its data engineering team to develop a NoSQL database to analyze its user songplay data.
The NoSQL database management system chosen for this project is Apache Cassandra. The data engineering
team's task is to take queries that are of interest to management and design the keyspace and tables required
to answer these queries.

## Overview
Sparkify has data about its users' songplay sessions contained in multiple CSV files. The data engineering team
must extract the data from the CSV files and move them into the appropriate tables of an Apache Cassandra NoSQL
database. Each table must be designed to help the Sparkify analysts answer the following queries:

1. Give me the artist, song title and song's length in the music app history that was heard during  sessionId = 338, and itemInSession  = 4
2. Give me only the following: name of artist, song (sorted by itemInSession) and user (first and last name) for userid = 10, sessionid = 182
3. Give me every user name (first and last) in my music app history who listened to the song 'All Hands Against His Own'

## Installation
In order to follow along with the procedures in this project, you will need to install the following on your local computer:

1. Apache Cassandra
2. Python
3. The Python-Cassandra driver

### Install Apache Cassandra
**Linux**
This link contains the directions to install Cassandra on a Debian-based Linux machine.
https://www.tecmint.com/install-apache-cassandra-on-ubuntu/

**MacOS**
The easiest way to install Cassandra on a Mac is using Homebrew. The first link contains instructions on how to install Homebrew
and then use it to install Cassandra.
https://gist.github.com/hkhamm/a9a2b45dd749e5d3b3ae

The second link contains much of the same information but also has an alternative way to start the Cassandra server, which is easier to type
than the method shown in the first link.
https://medium.com/@manishyadavv/how-to-install-cassandra-on-mac-os-d9338fcfcba4

On my Mac running macOS Catalina, I have found that if I install Cassandra using Homebrew then I cannot connect to the Casssandra server using either `cqlsh` or through the Python Cassandra Driver. Rather, I can avoid the connection issue if I install Casssandra from source instead. Instructions for
installing from source can be found at this link:
https://medium.com/@areeves9/cassandras-gossip-on-os-x-single-node-installation-of-apache-cassandra-on-mac-634e6729fad6

The author refers to an older version of Cassandra. You can update the commands from the article with the latest version, which as of this writing is
Cassandra 3.11.9. The latest version can be found from the Cassandra website.
https://cassandra.apache.org/download/

**Windows**
You can find instructions for installing Cassandra on a Windows 10 machine here: 
https://medium.com/informatics/how-to-install-apache-cassandra-on-windows-10-210045e50f57

Another option for installing Cassandra on Windows is to first set up WSL2 on your machine. Instructions for how to do so are here:
https://docs.microsoft.com/en-us/windows/wsl/install-win10

Once you have WSL2 on you machine, you can follow the instructions for installing Cassandra on Linux above.

### Installing Cassandra Driver for Python
If you have not alreay done so, we can now install the python cassandra driver. From a command line terminal, activate your python environment and install the driver using this command:

`pip install cassandra-driver`.

If the above command does not work, then try:

`pip3 install cassandra-driver`.

### Test it Out
If you were able to successfully install Cassandra and the python driver, then you can now open a terminal and start the cassandra server. The links I provided above contains the directions
for how to start the server on each type of OS. Once the server starts, open another terminal window and issue the command `cqlsh`. If there are no issues, then you should be able to see this appear
on your terminal:

```
Connected to Sparkify Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.9 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh>
```

You will see a different Cluster name. The default cluster name is `Test Cluster`. You can modify the cluster name, along with other settings, through the
cassandra.yaml file. For a typical Linux install, you can find this file in the `/etc/cassandra/` directory. You will have to check the location of this file
if you are using another OS.

## Creating the Database
We will be connecting to our Cassandra server through python. To connect to the server on our local machine, we run the following python script:

```python
from cassandra.cluster import Cluster

try:
    cluster = Cluster(['127.0.0.1'])
    # To establish connection and begin executing queries, need a session
    session = cluster.connect()
```

If there are no errors, then we are successfully connected to our local Cassandra server.

### Creating the Keyspace
The keyspace is analogous to a Postgresql database. The keyspace contains the tables that are designed around the queries. 
Run the following script to create the keyspace:

```python
session.execute("""
    CREATE KEYSPACE IF NOT EXISTS sparkify
    WITH REPLICATION = 
    {'class': 'SimpleStrategy',  'replication_factor': 1}
""")
```

The string in the code above is an example of CQL, the query language used to communicate with the server. Since we are using a python
driver to interact with the server, we have to wrap the CQL in a `session.execute()` function. For brevity, I will only be
showing the CQL from now on, omitting the python portion unless the python code is of particular importance.

### Creating the Tables
The tables in a Cassandra keycluster must be designed around the query. Since there are no `JOIN` or aggregate options, we must make sure
that all the data columns are available in our table in order for us to answer the queries we have in mind. We must also carefully design
the primary key for each table in order to optimize data retrieval for our queries. 

#### First Query Table - Session Table
The first query is: *Give me the artist, song title and song's length in the music app history that was heard during sessionId = 338, and itemInSession = 4*.
We create the table for this query using:

```SQL
CREATE TABLE IF NOT EXISTS session_table 
(session_id int, item_in_session int, artist_name text, song_title text, song_length float,
PRIMARY KEY ((session_id, item_in_session), artist_name, song_title, song_length))
```

There is a field for each piece of information we need to return. Also, since we are conditioning the results based on `sessionId` and `itemInSession`,
we place those two first as they both make up the partition key.

#### Second Query Table - User_Session Table
The second query is: *Give me only the following: name of artist, song (sorted by itemInSession) and user (first and last name) for userid = 10, sessionid = 182*.
The conditioning clause for this query is different from the first query. Therefore, we have a different partition key and will need to create another table.

```SQL
CREATE TABLE IF NOT EXISTS user_session_table 
(user_id int, session_id int, artist_name text, item_in_session int, song_title text,
first_name text, last_name text,
PRIMARY KEY ((user_id, session_id), item_in_session, artist_name,  
             song_title, first_name, last_name))
```

In this table, the partition key consists of `user_id` and `session_id` because we condition the query on these two fields. We must also sort `song` by `itemInSession`
but we are not asked to return `itemInSession`. Also, we are asked to return `artist` data. In order to meet these conditions, we have to carefully design the
clustering key, which is the portion of the primary key that follows the partition key. By placing `item_in_session` as the first component of the clustering key,
we will be sorting the results based on this field. The results of the query must be in the same order as how the fields are arranged in the primary key, so
we place artist before song, and the rest of the user information at the end.

#### Third Query Table - User Table
The third query is: *Give me every user name (first and last) in my music app history who listened to the song 'All Hands Against His Own'*
The conditioning field is the song title. We must include that in the partioning key.

```SQL
CREATE TABLE IF NOT EXISTS user_table
(first_name text, last_name text, song_title text,
PRIMARY KEY(song_title, first_name, last_name))
```

The other fields we have to return are the names of the user listening to the song.

### Inserting Data into the Tables
The data is found in the `event_data` directory, spread across several CSV files. A python function uses the `csv` library to access the CSV data by iterating
through each file and then extracting them into a single `event_datafile_new.csv` file. For each table, we have extract the data from the relevant columns from
this new CSV file and insert them into the table using CQL. Here is an example of this process for the first query table:

```python
# We have provided part of the code to set up the CSV file. Please complete the Apache Cassandra code below#
file = 'event_datafile_new.csv'

with open(file, encoding = 'utf8') as f:
    csvreader = csv.reader(f)
    next(csvreader) # skip header
    for line in csvreader:
## TO-DO: Assign the INSERT statements into the `query` variable
        query = "INSERT INTO session_table (session_id, item_in_session, artist_name, song_title, song_length)"
        query = query + "VALUES (%s, %s, %s, %s, %s)"
        ## TO-DO: Assign which column element should be assigned for each column in the INSERT statement.
        ## For e.g., to INSERT artist_name and user first_name, you would change the code below to `line[0], line[1]`
        try:
            session.execute(query, (int(line[8]), int(line[3]), line[0], line[9], float(line[5])))
        except Exception as e:
            print(e)
```

Note that we have to specify the column number for each field of the table. Note that we have to recast the data from the CSV to match the data type of the table.
Inserting data into the other two tables follows the same process, with different data column numbers to match the table fields.

## Running the Queries
Now, we have created the tables and inserted data. We can run the queries that the Sparkify analysts had requested.
The first query is: *Give me the artist, song title and song's length in the music app history that was heard during sessionId = 338, and itemInSession = 4*.
We run the query with the following command:

```SQL
SELECT artist_name, song_title, song_length FROM session_table 
WHERE session_id = 338 and item_in_session = 4
```

This returns:
```
('Faithless', 'Music Matters (Mark Knight Dub)', 495.30731201171875)
```

The second query is: *Give me only the following: name of artist, song (sorted by itemInSession) and user (first and last name) for userid = 10, sessionid = 182*.
We run the query with the following command:

```SQL
SELECT artist_name, song_title, first_name, last_name 
FROM user_session_table
WHERE user_id = 10 and session_id = 182
```

This returns:
```
('Down To The Bone', "Keep On Keepin' On", 'Sylvie', 'Cruz')
('Three Drives', 'Greece 2000', 'Sylvie', 'Cruz')
('Sebastien Tellier', 'Kilometer', 'Sylvie', 'Cruz')
('Lonnie Gordon', 'Catch You Baby (Steve Pitron & Max Sanna Radio Edit)', 'Sylvie', 'Cruz')
```

The third query is: *Give me every user name (first and last) in my music app history who listened to the song 'All Hands Against His Own*.
We run the query with the following command:

```SQL
SELECT first_name, last_name 
FROM user_table
WHERE song_title = 'All Hands Against His Own'
```

This returns:
```
('Jacqueline', 'Lynch')
('Sara', 'Johnson')
('Tegan', 'Levine')
```

## Shutting Down
After we have successfully built the tables and run the queries, we should disconnect from the server by running the following python script:

```python
session.shutdown()
cluster.shutdown()
```