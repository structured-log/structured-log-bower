#structured-log [![Build Status](https://travis-ci.org/structured-log/structured-log.svg)](https://travis-ci.org/structured-log/structured-log)

A structured logging framework for Javascript. Based on [Serilog](http://serilog.net/).

**WARNING:** structured-log is a work in progress and will change over time.

## Installation

### Server-side ([Node.js](https://nodejs.org/)) via [npm](https://www.npmjs.com/package/structured-log)

  npm install --save structured-log

### Client-side (Web Browser) via [Bower](http://bower.io/search/?q=structured-log)

  bower install --save structured-log
 
## Basic Setup

This section describes the most basic *structured-log* configuration for server and client.

### Server-side

In your NodeJS script:

	var structuredLog = require('structured-log');
	var consoleSink = require('structured-log/console-sink');

	var log = structuredLog.configure()
    	.writeTo(consoleSink())
	    .create();
    
### Client-side

In your HTML file:

```
  <script type='text/javascript' src='bower_components/structured-log/structured-log.min.js />
```

In your Javascript code:

	var log = structuredLog.configure() 
    	.writeTo(structuredLog.consoleSink())
	    .create();
  
### Multiple sinks

Setup of *structured-log* is via a *fluent API* that configures and creates a logger. One example of this is to specify multilple sinks, eg:

	var log = structuredLog.configure() 
		.writeTo(consoleSink)
		.writeTo(httpSink({ url: 'http://somewhere.com' }))
		.create();

### Writing to another log

A log can easily be piped to another log:

	var someOtherLog = structuredLog.configure()
		// ... setup ...
		.create(); 

	var log = structuredLog.configure() 
		.writeTo(consoleSink)
		.writeTo(someOtherLog)
		.create();


## Basic Usage

Verbose (not output by default):

	log.verbose('My debug message!');
 
Info:

	log.info('Something happened in the application...');

Info alternative:

	log('Something happened in the application...');

Warning:
  
	log.warn('Some not-fatal error happened...');

Error:

	log.error('Something bad happened...');

Error can also accept an exception or error object:

	log.error(exceptionOrErrorObject, 'Something bad happend...');

## Structured Logging

All the logging functions accept a message template and a set of key/value properties. When the template is rendered for display the values of the properties can be included in the message. 

The properties are maintained separately to the template and rendered message which is what makes this a structured logging system.

Here are some examples that have been adapted for Javascript from the [Serilog C# examples](http://serilog.net/):
 
	var position = { Latitude: 25, Longitude: 134 };
	var elapsedMs = 34;
	
	log.info("Processed {@Position} in {Elapsed:000} ms.", {
		Position: position,
		Elapsed: elapsedMs
	});

Properties can also be specified by positional parameters, the same as in Serilog C#: 

	log.info("Processed {@Position} in {Elapsed:000} ms.", position, elapsedMs);

## Included Sinks

A *sink* is a plugin that is invoked for each *log event*. Usualy a sink defines an *output method* for logs, such as the ability to output to the [console]( https://developer.mozilla.org/en/docs/Web/API/console).  

*structured-log* includes a number of built-in sinks.

### Server-side

All sinks are imported using the Nodejs *require* function as follows:

	var someSink = require('<sink-name>');

| Name | Description |
| ---- | ----------- |
| console-sink | Writes formatted log events to the *console* |
| colored-console-sink | Same as above, but with colors |
| json-console-sink | Writes structured json to the console for each log event |
| stream-sink | Writes formatted log events to a Nodejs stream |
| json-stream-sink | Writes structured json tot he console for each log event |
| http-sink | Outputs structured json log events via HTTP post |

### Client-side

| Name | Description |
| ---- | ------------- |
| console-sink | Writes formatted log events to the *console* |
| json-console-sink | Writes structured json to the console for each log event |
| http-sink | Outputs structured json log events via HTTP post |

## Batching

All sinks can be batched, although it only really makes sense for sinks that require batching for increased performance or to reduce timing issues (eg HTTP logs being received out of order). 

### Configuring Batching

All sinks can have batching enabled by calling the *batch* function with an appropriate configuration:

	var httpSink = require('structured-log-http-sink'); 
	
	var log = structuredLog.configure()
		.batch({
			batchSize: 1000,          	// Flush the queue every 1000 logs.
			timeDuration: 3000,         // Milliseconds to wait before flushing the queue.            
		})
		.writeTo(httpSink({
			url: 'http://somelogreceiver',	// Configuration for the custom sink.
		})
		.create();

*batchSize* specifies the amount of logs to include in a batch. When this number of logs are in the queue the queue will be flushed and processed by the sink.

*timeDuration* specifies the amount of time that will pass before the log queue is flushed. This ensure that the queue is periodically flushed even if not enought logs events have been queued to trigger the *batchSize* flush. 

Either of these options can be omitted and will be set to default values.

### Flushing Queued Logs

The queue of batched logs can be flushed at any time by calling the *flush* function.

If it suits your purpose you can simply call flush:
	
	log.flush();

If you need a callback when the flush has completed you have two options.

The first option is the standard Javascript-style callback:

	log.flush(function (err) {
		if (err) {
			// An error occurred while flushing.
		}
		else {
			// The queue was flushed successfully.
		}
	});
 
The second option is to use the promise that is returned by *flush*:

	log.flush()
		.then(function () {
			// The queue was flushed successfully.
		})
		.catch(function (err) {
			// An error occurred while flushing.
		})
		.done(); // Terminate the promise chain.

## 3rd-party Sinks

A number of additional sinks are available as separate packages.

If you release your own custom sink for *structured-log* please let us know and we'll add it to the list!  

### Server-side (via npm)

| Name | Description |
| ---- | ------------- |
| [email-sink](???) | Outputs formattted log messages via SMTP |
| [mongodb-sink](???) | Writes structure json log events to the MongoDB database |

### Client-side (via bower)

| Name | Description |
| ---- | ------------- |
| [websockets-sink](???) | Outputs formattted log messages via websockets |
| [socketio-sink](???) | Outputs formattted log messages via [the Socket.IO library](http://socket.io/) |

## Make your own sink

It is very easy to make your own sink. There are plenty of built-in examples of sinks. So can you can always copy and modify an existing sink.

### Custom sink

All sinks are designed to run in either *batched* or *unbatched* modes. The *emit* function always accepts an array of *log events*. When running unbatched this array will contain only a single item. When running batched the array may contain more than 1 *log event* depending on how many have been queued. 

	var myCustomSink = function (options) {
		return {
			emit: function (logEvents) {
				//
				// ... process array of log events ...
				//
			}
		};
	};
	
	var customSinkOptions = {
		// Whatever custom options you need...
	};
	
	var log = structuredLog.configure()
		.writeTo(myCustomSink(customSinkOptions))
		.create();

**As a Nodejs module:**

	module.exports = function (options) {
		return {
			emit: function (logEvents) {
				//
				// ... process array of log events ...
				//
			}
		};
	};

**Somewhere else:**

	var myCustomSink = require('./MyCustomSink');
	
	var customSinkOptions = {
		// Whatever custom options you need...
	};
	
	var log = structuredLog.configure()
		.writeTo(myCustomSink(customSinkOptions))
		.create();


## Advanced Setup

The *fluent API* has numerous functions to configure your log.

### Log Levels

Set the minimum log level that is output:

	var log = structuredLog.configure()
		.minLevel('WARN')
		.writeTo(consoleSink())
		.create();

*minLevel* applies to subsequent sinks in the configuration, so you can use it to set a different level for each sink: 

	var log = structuredLog.configuration()
		.minLevel('VERBOSE')
		.writeTo(consoleSink())
		.minLevel('INFO')
		.writeTo(httpSink())
		.minLevel('ERROR')
		.writeTo(emailSink())
		.create();

### Filtering

Custom filtering can be applied to include/exclude logging based on a predicate function. 

	var log = structuredLog.configure()
		.filter(function (logEvent) {
			return someCondition(logEvent);
		})
		.writeTo(consoleSink())
		.create();

This kind of filtering affects subsequent sinks in the configuration, you can use it in combination with *clearFilter* to provide different filters for different sinks: 

	var log = structuredLog.configure()
		.filter(function (logEvent) {
			return okForConsole(logEvent);
		}))
		.writeTo(consoleSink())
		.resetFilter()
		.filter(function (logEvent) {
			return okForHttp(logEvent);
		}))
		.create();

Logs can also be filtered after configuration, this effectively creates a new log with the added filter:

	var log2 = log.filter(function (logEvent) {
		// ... some condition ...
	});
	
	log2.info("This log is filtered by the new criteria!");   


### Enrichment

Enrichment can be used to add key/value properties to all logs output via a particular logger.

	var log = structuredLog.configure()
		.enrich({
			UserId: getCurUserId(),
			SessionId: getCurSessionId(),
		})
		.writeTo(consoleSink())
		.create();

A function can also be provided that is evaluated at runtime to attach properties to log events in a more dynamic fashion:

	var log = structuredLog.configure()
		.enrich(function () {
			return {
				UserId: getCurUserId(),
				SessionId: getCurSessionId(),		
			};
		})
		.writeTo(consoleSink())
		.create();

Any number of properties can be attached to log messages in this manner. The properties may then be used in the log messages themselves:

	log.info("Current user {UserId} has done something.");

As with other configuration, the *enrich* only affects subsequent sinks.

Logs can also be enriched after configuration, this effectively creates a new log with additional properties:

	var log2 = log.enrich({ NewProperty: 'just for log2' };
	
	log2.info("I've added a new property: {NewProperty}");   
	
### Tagging

Logs can be tagged with string values. This is useful to filter and categories logs generated by an application:

	var log = structuredLog.configure()
		.tag("authentication-system")
		.writeTo(consoleSink())
		.create();

Logs can also be tagged after configuration, this effectively creates a new log that has the desired tag:
	
	var log2 = log.tag('some-new-tag');
	
	log2.info("This log is tagged with 'some-new-tag'");
	
