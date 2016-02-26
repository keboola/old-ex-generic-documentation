# Extractor configuration
---

# Basics

- The extractor configuration must be stored in attributes of a **sys.c-ex-api-{apiName}** bucket
- All tables within such bucket are configurations for that extractor

## baseUrl

The most important part of configuration, the API url (should end with a `/`)
- **Must** be either a string or user function (allows custom domains, see examples)

Example:

    `https://connection.keboola.com/v2/`

-- OR --

    {
        "function": "concat",
        "args": [
            "https://",
            { "attr": "domain" },
            ".example.com/api/v2/"
        ]
    }

- for *https://yourDomain.example.com/api/v2/*
- uses configuration table attribute, where attribute **domain** would contain `yourDomain`

## http.requiredHeaders

- Headers required to be set in all configuraiton tables
- Should be a comma separated list, eg: `App-Key,X-User-Email`
- **header.{Header-Name}** attribute in configuration table (eg: `header.App-Key`)

## http.header.{Header-Name}

- Headers to be sent with all requests from all configurations
- eg: **http.header.Accept-Encoding**: `gzip`

# Authentication
## Methods
### basic

- **auth.type**: `basic`
- use **username** and **password** attributes in the configuration table

### url.query

- Supports signature function as a value
- Values should be described in configuration bucket attributes
- Example bucket attributes:

- **auth.type**: `url.query`
- **query.apiKey**: `{"attr": "apiKey"}`
    - this will look for the *apiKey* query parameter value in the configuration table attribute named *apiKey*
- **query.sig**: `{"function":"md5","args":[{"function":"concat","args":[{"attr":"apiKey"},{"attr":"secret"},{"function":"time"}]}]}`
    - this will generate a *sig* parameter value from MD5 of merged configuration table attributes *apiKey* and *secret*, followed by current *time()* at the time of the request (time() being the PHP function)
        - Allowed functions are listed below in the *User functions* section

# Pagination
## Methods
Configured in bucket attr. `pagination.method`

### offset

- **pagination.method**: `offset`
- **pagination.limit**: integer
    - If a *limit* is set in configuration's **params** field, it will be overriden by its value
- **pagination.limitParam**(optional)
    - sets which query parameter should contain the limit value (default to `limit`)
- **pagination.offsetParam**(optional)
    - sets which query parameter should contain the offset value (default to `offset`)

### response.param

- **TODO**
- Can be either
    - a value of a query parameter
- set where to search for the paging string (getDataFromPath?)

### response.url

- **pagination.method**: `response.url`
- **pagination.urlKey**: `next_page`
    - path within response object that points to the URL
    - if value of that key is empty, pagination ends
- **pagination.includeParams**: `false`
	- whether or not to add "params" from the configuration to the URL's query from response
	- if enabled and the next page URL has the same query parameters as the "params" field, values from the "params" are used

### pagenum
simple page number increasing 1 by 1

- **pagination.method**: `pagenum`
- **pagination.pageParam**:(optional) `page` by default
- **pagination.limit**:(optional) integer
    - define the page size
    - if limit is omitted, the pagination will end once an empty page is received. Otherwise it stops once the reply contains less entries than the limit.
- **pagination.limitParam**:(optional)
    - query parameter name to use for *limit*

# Config table
## Attributes
Attributes must be configured accordingly to the bucket configuration (eg *auth*, *pagination*, *http.requiredHeaders*)

There are system attributes created automatically when each job (row of the configuration table) is executed, and when it's finished:

- **job.{rowId}.start**: last time the job was started
- **job.{rowId}.success**: last time the job finished successfully
- **job.{rowId}.success_startTime**: start time of last successful run
- **job.{rowId}.error**: last time the job failed

Where `{rowId}` matches the **rowId** column for each row within the table

## Data
- Columns:
    - **endpoint**(required): The API endpoint
    - **params**: Query parameters of the api call, JSON encoded
        - Each parameter in the JSON encoded object may either contain a string, eg: `{""start_date"": ""2014-12-26""}`
        - OR contain an user function as described below, for example to load value from parameters:
        - ```
            {""start_date"":{""function"":""date"",""args"":[""Y-m-d+H:i"",{""function"":""strtotime"",""args"":[{""attr"":""job.1.success""}]}]}}
            ```
    - **rowId**(required): An unique identificator of the configuration row
    - **dataType**: Type of data returned by the endpoint. It also describes a table name, where the results will be stored
    - **dataField**: Allows to override which field of the response will be exported.
        - If there's multiple arrays in the response "root" the extractor may not know which array to export and fail
        - If the response is an array, the whole response is used by default
        - If there's no array within the root, the path to response data **must** be specified in *dataField*
        - Can contain a path to nested value, dot separater (eg `result.results.products`)
    - **recursionParams**: A Json encoded additional configuration for recursion.
        - Currently only "filter" key is supported, which should contain a value consisting of a name of a field from the parent's response, logical operator and a value to compare against. Supported operators are "**==**", "**<**", "**>**", "**<=**", "**>=**", "**!=**"
        - Example: `type!=employee` or `product.value>150`
        - The filter is whitespace sensitive, therefore `value == 100` will look into `value ` for a ` 100` value, instead of `value` and `100` as likely desired.
        - **TODO** functions

# User functions
Can currently be used in query type authentication or endpoint parameters

## Allowed functions

- `md5`: Generate a md5 key from its argument value
- `sha1`: Generate a sha1 key from its argument value
- `time`: Return time from the beginning of the unix epoch in seconds (1.1.1970)
- `date`: Return date in a specified format
- `strtotime`: Convert a date string to number of seconds from the beginning of the unix epoch
- `base64_encode`
- `hash_hmac`: [See PHP documentation](http://php.net/manual/en/function.hash-hmac.php)
- `sprintf`: [See PHP documentation](http://php.net/manual/en/function.sprintf.php)
- `concat`: Concatenate its arguments into a single string
- `implode`: Concatenate an array from the second argument, using glue string from the first arg

## Syntax
The function must be specified in a JSON format, which may contain one of the following 4 objects:

- **String**: `{ "something" }`
- **Function**: One of the allowed functions above
    - Example (this will return current date in this format: `2014-12-08+09:38`:

        ```
        {
            "function": "date",
            "args": [
                "Y-m-d+H:i"
            ]
        }
        ```

    - Example with a nested function (will return a date in the same format from 3 days ago):

        ```
        {
            "function": "date",
            "args": [
                "Y-m-d+H:i",
                {
                    "function": "strtotime",
                    "args": ["3 days ago"]
                }
            ]
        }
        ```

- **Configuration table Attribute**: `{ "attr": "attributeName" }` or `{ "attr": "nested.attribute.name" }`
- **Query parameter**: **TODO**

# Group API

## Extractor run [/run]

### Run extraction [POST]

JSON Parameters:

- **config** (required) ... configuration id (name of configuration table), OR an *{apiName}.{configName}* string
- **api** (optional) ... the *{apiName}* as set in the configuration bucket name. Required if **config** doesn't contain the API information

+ Request (application/json)

    + Headers

            Accept: application/json
            X-StorageApi-Token: Your-Sapi-Token

    + Body

            {
                "config": "keboola.main"
            }

    + Schema

            {
                "type": "object",
                "required": true,
                "properties": {
                    "config": {
                        "type": "string",
                        "required": true
                    },
                    "api": {
                        "type": "string",
                        "required": false
                    }
                }
            }

+ Response 201 (application/json)

        {
            "id": "48419532",
            "url": "https://syrup.keboola.com/queue/job/48419532",
            "status": "waiting"
        }


## Definitions parse [/definitions{?url,type}]

### Get definitions from URL [GET]

+ Parameters
    + url = `` (required, string, `http://example.com/blueprint.apib`) ... URL of the API schema (blueprint)

    + type = `` (required, string, `apib`) ... Document schema type

        + Values
            + `apib`

+ Response 201 (application/json)

        {
            "TODO"
        }
