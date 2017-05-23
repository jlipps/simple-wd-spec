# The W3C WebDriver Spec: A Simplified Guide

This document is essentially a cheat sheet for the official [WebDriver spec](https://www.w3.org/TR/webdriver/). The official spec is designed for implementers to have very detailed information about processing algorithms and so on. Much of the information in the spec is not targeted towards those who are simply writing client libraries, or even users who want a closer look at the API. It also uses language which is so exact it can sometimes obfuscate an intuitive understanding of a section.

The approach used here is to simply look at the supported endpoints along with their inputs and outputs, without worrying how the implementation is supposed to work. This should be beneficial to client library implementers as well as remote end implementers looking for some quick highlights. And in most cases there are examples to illustrate

**DISCLAIMER**: This is _not_ the official spec. It is my interpretation of it and an attempt to present the most salient bits of it in a more digestible fashion. You should always consult the official spec before beginning work on a client or server implementation!

## Introduction

What is WebDriver? From the spec:

> WebDriver is a remote control interface that enables introspection and control of user agents. It provides a platform- and language-neutral wire protocol as a way for out-of-process programs to remotely instruct the behavior of web browsers.

Essentially, it's a client-server protocol that allows you to automate web browsers. Clients send well-formed requests, the server interprets them according to the protocol, and then performs the automation behaviors as defined by the implementation steps in the spec. The most common use of this technology is for automated testing.

The WebDriver spec grew out of the [Selenium](https://seleniumhq.org) project, and that is still the community of users pushing forward the associated browser automation technology and using it every day to write and run automated tests. Browser vendors now also support the WebDriver spec natively.

WebDriver has gone beyond the web, with implementations for mobile and desktop app automation. The [Appium](https://appium.io) project is a set of WebDriver-compliant servers that allow automation of these non-web-browser platforms.

### Basic Architecture

Automation is organized around WebDriver _sessions_, whose state is maintained across requests via a 'session id' token shared by the server and client. Creating a new session involves sending parameters in the form of _capabilities_, which tell the server what you want to automate and under what conditions. The server prepares the appropriate browser with any modifications as specified in the capabilities, and the session is then underway. Automation commands and responses are sent back and forth (keyed on the session id), until the client sends a request to delete the session, at which point the browser and other resources are quit or cleaned up, and the session id is discarded.

## Requests and Responses

### Overview

When the client (called a `local end`) sends a request to the server (called a `remote end`), this is known in the spec as a 'command'. Since this is an HTTP protocol, commands have several components:

* An HTTP verb
* A path

The remote end at this point looks up the command based on the HTTP verb and path. The spec defines [a list of endpoints](https://www.w3.org/TR/webdriver/#list-of-endpoints) that map verb + path to a command name. The path portion is actually a list of "URI Templates" that show how path components should be extracted as parameters for the command. For example, in:

```
/session/{session id}/element
```

The `{session id}` bit is saying that this component of the path is a "url variable" called `session id` whose value will be sent to the command (in this case `Find Element`). Once a command is matched to the request, other data is potentially parsed from the request body (these are the "parameters"), the command is executed (having been passed any url variables and request parameters), and a response is returned.

### Request format and handling

A request from the local end to the remote end is a valid HTTP request, with a verb, path, and potentially a body. As mentioned above, the remote end validates the request and attempts to map it to a command. If the request can't be mapped to a command, an `unknown method` error is returned (see below for what it means to return an error).

There is one command (`New Session`) which does not require a `session id` url variable. Every other command requires this variable, since every other command is executed in the context of an existing session. If we are not requesting a new session, the remote end immediately validates the session id against the list of active sessions. If it's not found, an `invalid session id` error is returned.

In the case of a `POST` request, the local end might have sent data in the request body. This data must always be JSON data. The remote end first parses it as JSON (if this fails, an `invalid argument` error is returned). If the result of the parse is not a JSON object (i.e., if it's a string or array or number or what have you), an `invalid argument` is likewise returned. Otherwise, the result of parsing is the set of "parameters" which is passed to the command.

(In the case of a `POST` request without a request body, the "parameters" value is `null`.)

### Response format

When a remote end sends an HTTP response, it first of all uses an appropriate HTTP status code and message (for example, `404` and `no such element`), based on the command that was attempted and the result. The spec defines status codes and messages for various responses, including success and error conditions (see below).

It then sets the following headers:

* `Content-Type`: `"application/json; charset=utf-8"`
* `Cache-Control`: `"no-cache"`

If any data needs to be returned with the response, it is serialized into a JSON object with the key `value`, e.g.:

```json
{"value": null}
```

And this becomes the body of the HTTP response.

### Normal responses

When an error has not occurred, the HTTP status is `200`, and the response body is the appropriate JSON object with the response data in the `value` property of the JSON object.

### Error handling

When an error occurs, the remote end first of all determines the appropriate error code and corresponding HTTP status code (see below for the full list). For example, if an element could not be found, the error code is `no such element` and the corresponding HTTP status code is `404`. The remote end then constructs a data JSON object with the properties `error`, `message`, and `stacktrace`. Here `error` is just the JSON code for the error (see table below; usually the same as the error code itself). `message` is whatever implementation-specific error message is appropriate. And likewise `stacktrace` is an implementation-specific stacktrace useful to implementation maintainers in diagnosing any issues.

An example error JSON object could look like:

```json
{
  "error": "no such element",
  "message": "My fake implementation couldn't find your element",
  "stacktrace": "Fake:21> Not a real stacktrace"
}
```

Since this JSON object becomes the data for the response, the full response from the remote end would be an HTTP status code of `404`, the headers listed above, and finally the following JSON string as the response body:

```json
{
  "value": {
    "error": "no such element",
    "message": "My fake implementation couldn't find your element",
    "stacktrace": "Fake:21> Not a real stacktrace"
   }
}
```

### Error codes

The following is a list of all the possible errors, their HTTP status codes, and their JSON error codes:

|Error code|HTTP Status|JSON code|Description|
|---|---|---|---
|element click intercepted|400|element click intercepted|The Element Click command could not be completed because the element receiving the events is obscuring the element that was requested clicked.|
|element not selectable|400|element not selectable|An attempt was made to select an element that cannot be selected.|
|element not interactable|400|element not interactable|A command could not be completed because the element is not pointer- or keyboard interactable.|
|insecure certificate|400|insecure certificate|caused the user agent to hit a certificate warning, which is usually the result of an expired or invalid TLS certificate.|
|invalid argument|400|invalid argument|The arguments passed to a command are either invalid or malformed.|
|invalid cookie domain|400|invalid cookie domain|An illegal attempt was made to set a cookie under a different domain than the current page.|
|invalid coordinates|400|invalid coordinates|The coordinates provided to an interactions operation are invalid.|
|invalid element state|400|invalid element state|A command could not be completed because the element is in an invalid state, e.g. attempting to click an element that is no longer attached to the document.|
|invalid selector|400|invalid selector|Argument was an invalid selector.|
|invalid session id|404|invalid session id|Occurs if the given session id is not in the list of active sessions, meaning the session either does not exist or that it’s not active.|
|javascript error|500|javascript error|An error occurred while executing JavaScript supplied by the user.|
|move target out of bounds|500|move target out of bounds|The target for mouse interaction is not in the browser’s viewport and cannot be brought into that viewport.|
|no such alert|400|no such alert|An attempt was made to operate on a modal dialog when one was not open.|
|no such cookie|404|no such cookie|No cookie matching the given path name was found amongst the associated cookies of the current browsing context’s active document.|
|no such element|404|no such element|An element could not be located on the page using the given search parameters.|
|no such frame|400|no such frame|A command to switch to a frame could not be satisfied because the frame could not be found.|
|no such window|400|no such window|A command to switch to a window could not be satisfied because the window could not be found.|
|script timeout|408|script timeout|A script did not complete before its timeout expired.|
|session not created|500|session not created|A new session could not be created.|
|stale element reference|400|stale element reference|A command failed because the referenced element is no longer attached to the DOM.|
|timeout|408|timeout|An operation did not complete before its timeout expired.|
|unable to set cookie|500|unable to set cookie|A command to set a cookie’s value could not be satisfied.|
|unable to capture screen|500|unable to capture screen|A screen capture was made impossible.|
|unexpected alert open|500|unexpected alert open|A modal dialog was open, blocking this operation.|
|unknown command|404|unknown command|A command could not be executed because the remote end is not aware of it.|
|unknown error|500|unknown error|An unknown error occurred in the remote end while processing the command.|
|unknown method|405|unknown method|The requested command matched a known URL but did not match an method for that URL.|
|unsupported operation|500|unsupported operation|Indicates that a command that should have executed properly cannot be supported for some reason.|

## The Endpoints

In this section, we go through each endpoint and examine its inputs and outputs and potential errors. The conventions I use are:

* "URL variables": variable strings slotted into URI templates
* "Request parameters": properties of the JSON object in the request body. Could be "None", which means no body
* "Response properties": properties of the JSON object which is the value of the `value` property of the response body. Could be "None", which means that `value` is null. For example, in this JSON response body:
    
    ```json
    {"value": {"foo": "bar"}}
    ```
    
    I'm calling `foo` a "response property" with a value of `"bar"`.
* "Possible errors": errors and codes it's possible for the command to return in case of an error specific to that command. Note that regardless of what's in this list, it's always possible for some errors to occur (e.g., `invalid session id` or `unknown error`). (A value of "None" here means "no particularly relevant errors", not that it's not possible for an error to occur!)

### List of all endpoints

|Method|URI Template|Command|
|------|------------|-------|
|POST|/session|[New Session](#new-session)|
|DELETE|/session/{session id}|[Delete Session](#delete-session)|
|GET|/status|[Status](#status)|
|GET|/session/{session id}/timeouts|[Get Timeouts](#get-timeouts)|
|POST|/session/{session id}/timeouts|[Set Timeouts](#set-timeouts)|
|POST|/session/{session id}/url|[Go](#go)|
|GET|/session/{session id}/url|[Get Current URL](#get-current-url)|
|POST|/session/{session id}/back|Back|
|POST|/session/{session id}/forward|Forward|
|POST|/session/{session id}/refresh|Refresh|
|GET|/session/{session id}/title|Get Title|
|GET|/session/{session id}/window|Get Window Handle|
|DELETE|/session/{session id}/window|Close Window|
|POST|/session/{session id}/window|Switch To Window|
|GET|/session/{session id}/window/handles|Get Window Handles|
|POST|/session/{session id}/frame|Switch To Frame|
|POST|/session/{session id}/frame/parent|Switch To Parent Frame|
|GET|/session/{session id}/window/rect|Get Window Rect|
|POST|/session/{session id}/window/rect|Set Window Rect|
|POST|/session/{session id}/window/maximize|Maximize Window|
|POST|/session/{session id}/window/minimize|Minimize Window|
|POST|/session/{session id}/window/fullscreen|Fullscreen Window|
|GET|/session/{session id}/element/active|Get Active Element|
|POST|/session/{session id}/element|Find Element|
|POST|/session/{session id}/elements|Find Elements|
|POST|/session/{session id}/element/{element id}/element|Find Element From Element|
|POST|/session/{session id}/element/{element id}/elements|Find Elements From Element|
|GET|/session/{session id}/element/{element id}/selected|Is Element Selected|
|GET|/session/{session id}/element/{element id}/attribute/{name}|Get Element Attribute|
|GET|/session/{session id}/element/{element id}/property/{name}|Get Element Property|
|GET|/session/{session id}/element/{element id}/css/{property name}|Get Element CSS Value|
|GET|/session/{session id}/element/{element id}/text|Get Element Text|
|GET|/session/{session id}/element/{element id}/name|Get Element Tag Name|
|GET|/session/{session id}/element/{element id}/rect|Get Element Rect|
|GET|/session/{session id}/element/{element id}/enabled|Is Element Enabled|
|POST|/session/{session id}/element/{element id}/click|Element Click|
|POST|/session/{session id}/element/{element id}/clear|Element Clear|
|POST|/session/{session id}/element/{element id}/value|Element Send Keys|
|GET|/session/{session id}/source|Get Page Source|
|POST|/session/{session id}/execute/sync|Execute Script|
|POST|/session/{session id}/execute/async|Execute Async Script|
|GET|/session/{session id}/cookie|Get All Cookies|
|GET|/session/{session id}/cookie/{name}|Get Named Cookie|
|POST|/session/{session id}/cookie|Add Cookie|
|DELETE|/session/{session id}/cookie/{name}|Delete Cookie|
|DELETE|/session/{session id)/cookie|Delete All Cookies|
|POST|/session/{session id}/actions|Perform Actions|
|DELETE|/session/{session id}/actions|Release Actions|
|POST|/session/{session id}/alert/dismiss|Dismiss Alert|
|POST|/session/{session id}/alert/accept|Accept Alert|
|GET|/session/{session id}/alert/text|Get Alert Text|
|POST|/session/{session id}/alert/text|Send Alert Text|
|GET|/session/{session id}/screenshot|Take Screenshot|
|GET|/session/{session id}/element/{element id}/screenshot|Take Element Screenshot|

### New Session

|HTTP Method|Path Template|
|-----------|-------------|
|POST|/session|

[Spec description](https://www.w3.org/TR/webdriver/#new-session):
> The `New Session` command creates a new WebDriver session with the endpoint node. If the creation fails, a `session not created` error is returned.

* **URL variables:**
	* None
* **Request parameters:**
	* `capabilities`: a JSON object with a special structure that's so complex it deserves its own section. See "Capabilities" under "Data Structures" below.
	* Example:

		```json
		{"capabilities": {...}}
		```
		
* **Response properties:**
	* `sessionId`: a string, the UUID reference of the session, to be used in subsequent requests
	* `capabilities`: a JSON object, the same object as was sent in via the request
	* Example:

		```json
		{
		  "value": {
		    "sessionId": "1234567890",
		    "capabilities": {...}
		  }
		}
		```
		 
* **Possible errors:**
	* `session not created` (`500`): if the session could not be started for a variety of reasons:
		* maximum session count exceeded
		* capabilities could not be processed
		* any other problem in session creation
	* `invalid argument` (`400`): if the capabilities object was malformed in some way (see section on capabilities for examples)

### Delete Session

|HTTP Method|Path Template|
|-----------|-------------|
|DELETE|/session/{session id}|

[Spec description](https://www.w3.org/TR/webdriver/#delete-session):
> The `Delete Session` command closes any top-level browsing contexts associated with the current session, terminates the connection, and finally closes the current session.

* **URL variables:**
	* `session id`: the id of a currently active session
* **Request parameters:**
	* None
* **Response properties:**
	* None
* **Possible errors:**
	* None

### Status

|HTTP Method|Path Template|
|-----------|-------------|
|GET|/status|

[Spec description](https://www.w3.org/TR/webdriver/#status):
> The `Status` command returns information about whether a remote end is in a state in which it can create new sessions and can additionally include arbitrary meta information that is specific to the implementation.

* **URL variables:**
	* None
* **Request parameters:**
	* None
* **Response properties:**
	* `ready`: boolean value; whether the server has the capability to start more sessions
	* `message`: implementation-specific string describing readiness state
	* arbitrary other properties denoting metadata returned by the remote end
	* Example:
	
		```json
		{
		  "value": {
		    "ready": true,
		    "message": "server ready",
		    "uptime": 123457890
		  }
		}
		```
		
* **Possible errors:**
	* None

### Get Timeouts


|HTTP Method|Path Template|
|-----------|-------------|
|GET|/session/{session id}/timeouts|

[Spec description](https://www.w3.org/TR/webdriver/#get-timeouts):
> The `Get Timeouts` command gets timeout durations associated with the current session.

* **URL variables:**
	* `session id`
* **Request parameters:**
	* None
* **Response properties:**
	* `script`: value (in ms) of the session script timeout
		
		> A session has an associated _session script timeout_ that specifies a time to wait for scripts to run. If equal to null then session script timeout will be indefinite. Unless stated otherwise it is 30,000 milliseconds.
		
	* `pageLoad`: value (in ms) of the session page load timeout
	
		> A session has an associated _session page load timeout_ that specifies a time to wait for the page loading to complete. Unless stated otherwise it is 300,000 milliseconds.
	
	* `implicit`: value (in ms) of the session implicit wait timeout
	
		> A session has an associated _session implicit wait timeout_ that specifies a time to wait in milliseconds for the element location strategy when retreiving elements and when waiting for an element to become interactable when performing element interaction. Unless stated otherwise it is zero milliseconds.
	
	* Example:
	
		```json
		{
		  "value": {
		    "script": 30000,
		    "pageLoad": 300000,
		    "implicit": 0
		  }
		}
		```
		
* **Possible errors:**
	* None
	
### Set Timeouts

|HTTP Method|Path Template|
|-----------|-------------|
|POST|/session/{session id}/timeouts|

[Spec description](https://www.w3.org/TR/webdriver/#set-timeouts):
> The `Set Timeouts` command sets timeout durations associated with the current session. The timeouts that can be controlled are listed in the table of session timeouts below.

* **URL variables:**
	* `session id`
* **Request parameters:** Send one or more of the following parameters. For definition of what each of these timeouts means, see [Get Timeouts](#get-timeouts) above.
	* `script`: integer in ms for session script timeout
	* `pageLoad`: integer in ms for session page load timeout
	* `implicit`: integer in ms for session implicit wait timeout
* **Response properties:**
	* None		
* **Possible errors:**
	* `invalid argument` (`400`) if a parameter property was not a valid timeout, or was not an integer in the range [0, 2<sup>64</sup> - 1]

### Go

|HTTP Method|Path Template|
|-----------|-------------|
|POST|/session/{session id}/url|

[Spec description](https://www.w3.org/TR/webdriver/#go):
> The `Go` command is used to cause the user agent to navigate the current top-level browsing context a new location.

* **URL variables:**
	* `session id`
* **Request parameters:** 
	* `url`: string representing an absolute URL (beginning with `http(s)`), possibly including a fragment (`#...`). Could also be a local scheme (`about:` etc).
* **Response properties:**
	* None		
* **Possible errors:**
	* `invalid argument` (`400`) if:
	    * `url` parameter is missing
	    * `url` parameter doesn't conform to above spec 		
	* `timeout` (`408`) if `url` is different from the current URL, and the new page does not load within the page load timeout.

### Get Current URL

|HTTP Method|Path Template|
|-----------|-------------|
|GET|/session/{session id}/url|

[Spec description](https://www.w3.org/TR/webdriver/#get-current-url):
> The `Get Current URL` command returns the URL of the current top-level browsing context.

* **URL variables:**
	* `session id`
* **Request parameters:** 
	* None
* **Response properties:**
	* `url`: current document URL of the top-level browsing context
* **Possible errors:**
	* `no such window` (`400`) if the current top-level browsing context is no longer open

### Back
### Forward
### Refresh
### Get Title
### Get Window Handle
### Switch To Frame
### Switch To Parent Frame
### Get Window Rect
### Set Window Rect
### Maximize Window
### Minimize Window
### Fullscreen Window
### Get Active Element
### Find Element
### Find Elements
### Find Element From Element
### Find Elements From Element
### Is Element Selected
### Get Element Attribute
### Get Element Property
### Get Element CSS Value
### Get Element Text
### Get Element Tag Name
### Get Element Rect
### Is Element Enabled
### Element Click
### Element Clear
### Element Send Keys
### Get Page Source
### Execute Script
### Execute Async Script
### Get All Cookies
### Get Named Cookie
### Add Cookie
### Delete Cookie
### Delete All Cookies
### Perform Actions
### Release Actions
### Dismiss Alert
### Accept Alert
### Get Alert Text
### Send Alert Text
### Take Screenshot
### Take Element Screenshot

## Data Structures

### Capabilities
