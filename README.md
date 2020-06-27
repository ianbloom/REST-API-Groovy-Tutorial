# So you want to query an API?
## Description
This repository is intended to function as a guide for writing DataSources that query the API.
You will find information on how queries are performed, coding best practices, and even lots of boilerplate code to get you started.
All code blocks in the README will be in Groovy for usage within DataSource development.

## What is an API?
API stands for *Application Programmatic Inteface*.  It's a way of requesting/creating/modifying data in systems that typically will allow these requests/modifications to be performed in the UI as well.
What this often means is that it is far easier to make sense of API documentation if you have used the tool and understand how calls are being made in the back end.
Here is an example within LogicMonitor of an API call occcuring behind the scenes:

INSERT SCREENSHOT HERE

API calls are made via HTTP requests which are very structurally similar.  There are many different models i.e. REST vs. SOAP, that represent different types of data in requests and responses.
As a matter of personal preference, REST is what you're hoping for :P

## Anatomy of an API request
### HTTP client
API requests are made via an HTTP client.

```groovy
// instantiate an http client object for the target system
httpClient = HTTP.open(hostname, port)
```

In >90% of situations, requests are made over port 443.  In the case of a web based API (ours for example), the hostname will take the form of **foobarbaz.com** (or subdomain.logicmonitor.com to continue using LogicMonitor as an example).

### HTTP vs. HTTPS
Both are possible.  Ensure that you make a note of which protocol to use when you build your query URL.  Beyond that, we will treat them basically the same.

### Base URL
In many cases, the hostname will be suffixed by an API path (in LogicMonitor **/santaba/rest**).

### API endpoint (or Resource Path depending on your preferred terminology)
This is a path to the resource you'd like to query/create/modify which will follow your base URL (in LogicMonitor, think **/device/devices** for... you guessed it... a list of devices)

### HTTP verb
This determines exactly how you will be interacting with the resource in question.  Because DataSources are primarily obtainers of data, **GET** will be by far the most common.  However, **POST** is often required to do things like authenticate with a /login endpoint.  Here is some useful documentation on [what verbs are available](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods).

### Query Parameters
While making a GET request, it may be useful to pass parameters that the API uses to shape its return data.  These are seperated from the end of the URL by **?**, separated from each other by **&**.
In LogicMonitor, we might make GET request to http://SUBDOMAIN.logicmonitor.com/santaba/rest/device/devices?size=1000&filter=name~"SQL" to ask for the first 1000 devices whose hostnames contain **SQL**.

### Headers
Many APIs require the passing of headers during calls to do things like pass credentials or cookies, API version, type of content requested i.e. "application/json" among other things.  In LogicMonitor, we require an **Authentication** header that takes the form "Authorization: LMv1 AccessId:Signature:Timestamp".  LogicMonitor will use this to verify that an approved request has occured, and if this value doesn't line up, you will likely recieve an HTTP 401: Unauthorized.

### Data
When making a POST, PUT, or PATCH, you need to send data to the API server to tell it exactly what you would like to create or modify.  For example, a **POST** to "/device/devices" in LogicMonitor must be accompanied by data that contains at the very least a display name and a hostname or IP address.

## How do I take all of this information and turn it into a query?

Luckily for us, LogicMonitor has its own [Groovy library](https://www.logicmonitor.com/support/terminology-syntax/scripting-support/access-a-website-from-groovy/) that simplifies this whole process.  Furthermore, LogicMonitor's Github has a [Monitoring Recipes](https://github.com/logicmonitor/monitoring-recipes) filled with boilerplate code using these libraries to make this process easy.  I have included relevant examples here in this repository under **boilerplate** to make things quick and easy.

Let's go through an example of querying a REST API, then parsing the output.

So first let's import LogicMonitor's HTTP library, as well as a JSON slurper which will help us interact with the API response.

```groovy
import com.santaba.agent.groovyapi.http.HTTP
import groovy.json.JsonSlurper
```

Next, let's import all the device properties we need to perform this query.  Perhaps we are querying the host itself which is acting as its own API server.  Perhaps we have stored API credentials as devic properties:

```groovy
hostname = hostProps.get("system.hostname")
user     = hostProps.get("api.user")
pass     = hostProps.get("api.pass")
```

Let's open up a connection to the API server with an HTTP client:

```groovy
httpClient = HTTP.open(hostname, 443)
```

Now we'll build our query URL using the components we spoke about above:

```groovy
url = "https://" + hostname + "/<INSERT-API-ENDPOINT>" + "?<INSERT-QUERY-PARAMETERS>" + "&<ONE-MORE-PARAMETER>"
```

Let's build some headers into a map as well.  Maps are data types that contain a number of key-value pairs.  They are incredibly powerful because you can use the key to "look-up" the value, which is exactly what we will do with our JSON output:

```groovy
headers                 = [:]
headers['user']         = user
headers['pass']         = pass
headers['Content-Type'] = 'application/json'
```

Now let's send the query that we've built!

```groovy
getResponse = httpClient.get(url, headers)
```

Nice! Now we ask the httpClient for the response body it received from this query:

```groovy
body = httpClient.getResponseBody()
```

Cool!  So the API server will be returning JSON because we are assuming this API is a REST API (and also because we requested this content type in the headers).  Technically, the API server will be returning a JSON string (just a bunch of text), so we will need to use our JSON slurper to turn this string into a map (just like our headers!)

```groovy
response_json = new JsonSlurper().parseText(body)
```

Let's assume that we've made a query to **/device/devices/<SOME-DEVICE-ID>** and we are trying to extract the time when it was created.  Here is the [associated model in our documentation](https://www.logicmonitor.com/swagger-ui-master/dist/#/Devices/getDeviceById).  See if you can figure out the JSON path to get the created time from our response_json that we just built!

```groovy
created_on_epoch = response_json['createdOn']
```

I promise this stuff ain't rocket surgery, it's mostly copy-pasting stuff directly out of documentation.  The last thing we need to do is print out the datapoint value so LogicMonitor can record it:

```groovy
println("created_on_epoch=${created_on_epoch}")
```

The **${}** business is [Groovy string interpolation](https://groovy-lang.org/syntax.html#_string_interpolation) and is an amazing shortcut to building datapoints for printing.  For more information on how LogicMonitor expects your output to be formatted, see [our documentation](https://www.logicmonitor.com/support/logicmodules/datasources/data-collection-methods/scripted-data-collection-overview).  It's also worth reading how [active discovery](https://www.logicmonitor.com/support/logicmodules/datasources/active-discovery/script-active-discovery) is expected to be formatted.

## Okay cool Ian how do I get started, how do I test my scripts, etc.

Incoming personal preference: download [Visual Studio Code](https://code.visualstudio.com/) and never look back.  It has an excellent UI based extension system, and it is a good idea to download Alignment (because it lines up your code.  That's how I made the headers look the way they do) as well as code-groovy which is Groovy linting and formatting, code coloring, and more helpful tools!  Shoot it'll even debug your code for you or tell you if you've instantiated variables that you never called again.

To test within LogicMonitor, I prefer using the [LogicMonitor Script Debug Helper](https://chrome.google.com/webstore/detail/logicmonitor-script-debug/ijojgoccfeggejpbhdahmeijjhpdjklf?authuser=1&_ga=2.34302374.602529484.1593289148-1947610191.1591034734) which is a helpful Chrome extension that allows you to copy paste your code in, supply a hostname to query, and away you go!

After installing, this extension will activate when the **!groovy** command is called within the LogicMonitor Collector Debug Facility.  All of this is [documented on LogicMonitor's website](https://www.logicmonitor.com/support/terminology-syntax/scripting-support/script-troubleshooting)!

## This is all well and good, but you forgot XYZ

Let me know if there's anything else you'd like me to include and I'd be happy to oblige!