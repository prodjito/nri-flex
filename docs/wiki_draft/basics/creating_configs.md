# Creating Flex Configs

Flex configurations are all defined by a yaml file.
The easiest way to kick start learning is to check the existing configs under [/examples](https://github.com/newrelic/nri-flex/tree/master/examples).

As this is a typical yaml file, wherever defining an array of multiples is possible is most definitely useable. This allows you to easily run multiple apis, commands, prometheus exporters, or database / jmx queries etc. You can even do a mix of all of those things in a single config file.

Viewing the internal/load/load.go file, can be a useful reference to some, to see all the available config options which have inline comments.

### Options
- [commands](#commands) Run any standard commands
- [net dial](#net-dial) Can be used for port testing or sending messages and processing the response
- [http](#http) General http requests
- [database queries](#database-queries)
- [jmx queries](#jmx-queries) Uses nrjmx to send JMX requests which will be processed after
- [using prometheus exporters](https://github.com/newrelic/nri-flex/wiki/Prometheus-Integrations-(Exporters))

### Further Configuration

#### [Functions available for things like pagination, manipulating the output, secret mgmt etc.](https://github.com/newrelic/nri-flex/wiki/Functions)
#### [Metric Parser for Rate & Delta Support](https://github.com/newrelic/nri-flex/wiki/Functions#metric_parser)
#### [Global Config](#global-config-that-is-passed-down)
#### [Setting Custom Attributes](#custom-attributes)
#### Environment variables can be used throughout any Flex config files by simply using a double dollar sign eg. $$MY_ENVIRONMENT_VAR.

***


### Commands

With the below example, we can create a redis integration in 6 lines, by simply running a command and parsing it.

```
---
name: redisFlex
apis: 
  - name: redis
    commands: 
      - run: (printf "info\r\n"; sleep 1) | nc -q0 127.0.0.1 6379 ### remove -q0 if testing on mac
        split_by: ":"
```


#### Run Command Specific Options
```
"shell"             // command shell
"run"               // command to run
"split"             // default "vertical", can be "horizontal" useful for outputs that look like a table
"split_by"          // character to split by
"set_header"        // manually set header column names (used when split is is set to horizontal)
"group_by"          // group by character
"regex"             // process SplitBy as regex (true/false)
"line_limit"        // stop processing at this line number
"row_header"        // start the row header at a different line (integer, used when split is horizontal)
"row_start"         // start creating samples from this line number, to be used with SplitBy
"ignore_output"     // ignore command output - useful chaining commands together
"custom_attributes" // set additional custom attributes
"line_end"          // stop processing at this line number
"timeout"           // when to timeout command in milliseconds (default 10s)
"dial"              // address to dial
"network"           // network to use (default tcp) (currently only used for dial)

```
See the redis example for a typical split, and look at the "df" command example for a horizontal split by example.

***

### Net Dial

Dial is a parameter used under commands.

port test eg.
```
name: portTestFlex
apis: 
  - timeout: 1000 ### default 1000 ms increase if you'd like
    commands:
    - dial: "google.com:80"
```

sending a message and processing the output eg.
```
---
name: redisFlex
apis: 
  - name: redis
    commands: 
      - dial: 127.0.0.1:6379
        run: "info\r\n"
        split_by: ":"
```



### HTTP

A simple example, we can easily set multiple URLs, or endpoints, and use the global setting to set the base URL. Alternatively, you could just have full URLs defined for each.
Supports GET, POST & PUT methods.

```
---
name: httpExample
global: ### can set global parameters or nested beneath each API set under APIs, nested will override the global setting for that API endpoint, 
        ### useful for different auth mechanisms per endpoint
    base_url: https://jsonplaceholder.typicode.com/ ### if used, the URLs built under APIs will be, base_url (from global) + url (from API)
    headers:                       
      myHeader: myValue
    user: hi
    pass: bye
    tls_config: # can be set globally to apply to all calls beneath
      insecure_skip_verify: true
      # ca: "/path/to/ca/file.crt"
      # min_version: 0
      # max_version: 0
custom_attributes: # applies to all apis
  myCustAttr: myCustVal
apis: 
  - event_type: httpSample
    url: todos/1
    custom_attributes:
      nestedCustAttr: nestedCustVal # nested custom attributes specific to each api
    headers:
      anotherHeader: myValue
    user: someOther
    pass: myPass
  - event_type: httpSample
    url: todos/2
    tls_config:
      enable: true # enable needs to be used if wanting to use the nested option
      insecure_skip_verify: true
      # ca: "/path/to/ca/file.crt"
      # min_version: 0
      # max_version: 0

```

POST / PUT
```
---
name: httpPostExample 
apis: 
  - name: httpPost
    url: https://jsonplaceholder.typicode.com/posts
    method: POST
    payload: > 
      {"title": "foo","body": "bar","userId": 1}
```

TLS Config

```
---
name: httpExample
global: ### can set global parameters or nested beneath each API set under APIs, nested will override the global setting for that api endpoint, 
        ### useful for different auth mechanisms per endpoint
    base_url: https://jsonplaceholder.typicode.com/ ### if used, the URLs built under APIs will be, base_url (from global) + url (from API)
    tls_config: # all subsequent calls will inherit this unless overridden
      insecure_skip_verify: true 
      # ca: "/path/to/ca/file.crt"
      # min_version: 0
      # max_version: 0
apis: 
  - event_type: httpSample
    url: todos/1
    custom_attributes:
      nestedCustAttr: nestedCustVal # nested custom attributes specific to each api
    tls_config:
      enable: true # <- this flag required if overriding global
      insecure_skip_verify: true
      # ca: "/path/to/ca/file.crt"
      # min_version: 0
      # max_version: 0
```

***


### Database Queries

Flex has several database drivers available, to help you run any arbitrary/custom queries against those databases.

* https://github.com/denisenkom/go-mssqldb //mssql | sql-server
* https://github.com/go-sql-driver/mysql   //mysql
* https://github.com/lib/pq                //postgres

The below example shows us being able to run multiple queries against one database. 
But also being able to define another database to send queries too. 
It is perfectly fine to use multiple database types in a single config file if you wish to do so.

```
name: postgresDbFlex
apis: 
  - database: postgres
    db_conn: user=postgres host=postgres-db.com sslmode=disable password=flex port=5432
    logging:
      open: true
    custom_attributes: # applies to all queries
      host: myDbServer
    db_queries: 
      - name: pgStatActivitySample
        run: select * FROM pg_stat_activity
        custom_attributes: # can apply additional at a nested level
          nestedAttr: nestedVal
      - name: pgStatAnotherSample
        run: select * FROM some_otherTable
  - database: postgres
    db_conn: user=abc host=myhost.ap-southeast-2.rds.amazonaws.com sslmode=disable password=mypass port=5432 # could be another DB
    queries: 
      - name: pgStatDbSample
        run: select * FROM pg_stat_database LIMIT 1
```

***

### JMX Queries

- Flex is able to utilise [nrjmx](https://github.com/newrelic/nrjmx) to run any JMX queries you need.
- A copy of the nrjmx jar file is kept [here](https://github.com/newrelic/nri-flex/tree/master/nrjmx)

#### Install JMX On Linux

Below are the steps to install the nrjmx file and .jar manually to enable support for JMX. It is important to substitute the `{desired version}` below with the version of `nri-flex` you want to use.

##### Steps
1. Download the package manually
   * Find the [release](https://github.com/newrelic/nri-flex/releases) needed
   * `curl -L https://github.com/newrelic/nri-flex/releases/download/{desired version}/nri-flex-linux-{desired version}.tar.gz -o nri-flex-linux-{desired version}.tar.gz`
2. Extract the compressed file
   * `tar -xvf nri-flex-linux-{desired version}.tar.gz`
3. Use the extracted directory as the working directory
   * `cd nri-flex-linux-{desired version}`
4. Run `install_linux.sh --jmx` with privileges, typically `sudo`, to create files in `/var/db/newrelic-infra`
5. (optional) Copy over a JMX configuration to begin monitoring
   * In the `nri-flex-linux-{desired version}/examples/flexConfigs/` folder there are different JMX examples which can be used as a reference
   * After copying an example config to `/var/db/newrelic-infra/custom-integrations/flexConfigs/`, edit the file according to your monitoring needs and save the file
   * Restart the infrastructure agent, `sudo systemctl restart newrelic-infra`

#### JMX options available are:
```
domain
user
pass
host
port
key_store
key_store_pass
trust_store
trust_store_pass
```

Below is a simple tomcat example.

```
name: tomcatFlex
global:
  jmx:
      host: "127.0.0.1"
      port: "9001"
apis: 
  - name: tomcatThreads
    event_type: tomcatThreadSample
    ### note "keep_keys" will do the inverse, if you want all metrics remove the keep keys blocks completely
    ### otherwise tailor specific keys you would like to keep, this uses regex for filtering
    ### this is useful for keeping key metrics
    keep_keys: ###
      - bean
      - maxThreads
      - connectionCount
    commands: 
      - run: Catalina:type=ThreadPool,name=*
        output: jmx
  - name: tomcatRequest
    event_type: tomcatRequestSample
    keep_keys:
      - bean
      - bytesSent
      - bytesReceived
      - errorCount
      - requestCount
    commands: 
      - run: Catalina:type=GlobalRequestProcessor,name=*
        output: jmx
  - name: manager
    event_type: tomcatManagerSample
    keep_keys:
      - bean
      - errorCount
      - requestCount
    commands: 
      - run: Catalina:type=GlobalRequestProcessor,name=*
        output: jmx
  - name: datasource
    event_type: tomcatDatasourceSample
    keep_keys:
      - bean
      - numActive
      - numIdle
```

***


#### Global Config that is passed down
```
base_url
user
pass
proxy
timeout
headers:
 headerX: valueX
jmx:
* domain
* user
* pass
* host
* port
* key_store
* key_store_pass
* trust_store
* trust_store_pass
```

### Custom Attributes
Custom attributes can be defined nearly anywhere in your configuration.
eg. under Global, or API, or further nested under each command. The lowest level defined attribute will take precedence.

A standard key:pair structure is used.
```
custom_attributes:
 greeting: hello
```