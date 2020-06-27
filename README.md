###### So you want to query an API?
### Description
This repository is intended to function as a guide for writing DataSources that query the API.
You will find information on how queries are performed, coding best practices, and even lots of boilerplate code to get you started.
All code blocks in the README will be in Groovy for usage within DataSource development.

### What is an API?
API stands for *Application Programmatic Inteface*.  It's a way of requesting/creating/modifying data in systems that typically will allow these requests/modifications to be performed in the UI as well.
What this often means is that it is far easier to make sense of API documentation if you have used the tool and understand how calls are being made in the back end.
Here is an example within LogicMonitor of an API call occcuring behind the scenes:

INSERT SCREENSHOT HERE

API calls are made via HTTP requests which are very structurally similar.  There are many different models i.e. REST vs. SOAP, that represent different types of data in requests and responses.
As a matter of personal preference, REST is what you're hoping for :P

### Anatomy of an API request
# HTTP client
API requests are made via an HTTP client.

```
// instantiate an http client object for the target system
httpClient = HTTP.open(hostname, port)
```

In >90% of situations, requests are made over port 443.  In the case of a web based API (ours for example), the hostname will take the form of **foobarbaz.com** (or subdomain.logicmonitor.com to continue using LogicMonitor as an example).

# HTTP vs. HTTPS
Both are possible.  Ensure that you make a note of which protocol to use when you build your query URL.  Beyond that, we will treat them basically the same.

# Base URL
In many cases, the hostname will be suffixed by an API path (in LogicMonitor **/santaba/rest**).

# API endpoint (or Resource Path depending on your preferred terminology)
This is a path to the resource you'd like to query/create/modify which will follow your base URL (in LogicMonitor, think **/device/devices** for... you guessed it... a list of devices)

# HTTP verb
This determines exactly how you will be interacting with the resource in question.  Because DataSources are primarily obtainers of data, **GET** will be by far the most common.  However, **POST** is often required to do things like authenticate with a /login endpoint.  Here is some useful documentation on [what verbs are available](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods).

# Query Parameters
While making a GET request, it may be useful to pass parameters that the API uses to shape its return data.  These are seperated from the end of the URL by **?**, separated from each other by **&**.
In LogicMonitor, we might make GET request to http://SUBDOMAIN.logicmonitor.com/santaba/rest/device/devices?size=1000&filter=name~"SQL" to ask for the first 1000 devices whose hostnames contain **SQL**.

# Headers
Many APIs require the passing of headers during calls to do things like pass credentials or cookies, API version, type of content requested i.e. "application/json" among other things.  In LogicMonitor, we require an **Authentication** header that takes the form "Authorization: LMv1 AccessId:Signature:Timestamp".  LogicMonitor will use this to verify that an approved request has occured, and if this value doesn't line up, you will likely recieve an HTTP 401: Unauthorized.

# Data
When making a POST, PUT, or PATCH, you need to send data to the API server to tell it exactly what you would like to create or modify.  For example, a **POST** to "/device/devices" in LogicMonitor must be accompanied by data that contains at the very least a display name and a hostname or IP address.

### How do I take all of this information and turn it into a query?

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

