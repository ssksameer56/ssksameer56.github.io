# Improving API Visibility with Logs in Go in a Gin API

## Overview

Logging is an important part of any application especially APIs in my opinion. It allows you to see how the system is behaving in real time. There's a limit to how much you can check via unit tests and integration tests and with logging you can check how your application behaves with any request.

Since, the application once deployed is a blackbox, logs are the only way to get a peek. Therefore, logs should be written or added so that they help you debug or see what is happening. They are the first line of defence when your application misbehaves. In my short career, extensive logging has helped me identify an issue quickly than debugging or trying to trace the code locally.

The approach I have defined below is the once I have used to APIs that serve a lot of clients(which API doesn't :P). The logging added is kept with the following ideas in mind

    1. Able to isolate a specific request and its flow in the API
    2. Able to identify issues or errors with any external services/APIs its consuming
    3. Easy to reproduce a request that may be causing issues

Before expanding on these points, I'll specify that we use Zerolog and go-gin to create our APIs so some parts are specific to those frameworks/packages. But it's easy to extend the idea to other loggers or frameworks.

If the API is misbehaving for all, its easy to fix. However, there are times when only one user or one client seems to get the issue. And given the volume of requests per second, its not easy to discern the logs for any specific request. You might then try to reproduce it locally, but setting up the client or running all the dependencies(databases, cache, external APIs etc) might take up a lot of time.

### Using Unique IDs

First idea is to create unique IDs(uuid4) for each request that is served by the API. You can add them in the controller and pass them via contexts in any Go function(its good practice to pass ctx as the first parameter to any Go function). Any function called would log using this ID. Most log browsers are good enough that they can search for this specific keyword and see you logs.

```
traceID := uuid.New().String() // create a unique Id
cctx = context.WithValue(c.Request.Context(), types.TraceIDKey{}, traceID) // add the value in a derived context
```
In the controller, you can derive the context from the original request context so you can keep track of deadlines/cancellations and add your unique ID to be accessed anywhere.

Keys for any context value should be kept any empty structs. Inside any function, you can access them using the following - 

```
traceId := ctx.Value(types.TraceIDKey{}).(string) // application would panic if this value doesn't exist
```

You can then use this unique ID in any logs that would be written in the function. It adds a bit of boilerplate code everywhere but totally worth the pain in my opinion.


### Logging the Complete Flow

I expect you'll be logging the incoming requests sans the sensitive data. Using the above approach we can trace the complete flow from request but GIN doesn't log the unique ID. You can override the default functionality - 

```
app.Use(gin.LoggerWithFormatter(func(param gin.LogFormatterParams) string {
		return fmt.Sprintf("%d %s - %s ",
			param.Latency.Microseconds(),
			param.Path,
			param.Request.Context().Value(types.TraceIDKey{}),
		)
	}))
```
The `LoggerWithFormatter` is an existing functionality where you can specify how you want GIN to log the HTTP logs. It has access to the request data via the context variable by default but not the unique ID we have. Remember, we added it in a context dervied from the request context. What you can do is add the uniqueID at the end where you are about to write the response back to client.

```
c.Request = c.Request.WithContext(context.WithValue(c.Request.Context(), types.TraceIDKey{}, traceID))
```
This adds any value you want to the original request context accessible to the gin `params` object via the request context.You can explore the `params` object to see that data that is available for any request and log it accordingly.

### Keeping an Eye on various components

It can be assumed that your API will be consuming or using various services via network - databases, cache, external APIs, maybe some authentication mechanism. Those can be unreliable and in my opinion, its important that you log them in a specific format or label them to see its behaviour. While using a simple string works as well, I find using `Zerolog's` `Str()` function better.

```
log.Err(err).Str("component", "Database").Str("uniqueID", uniqueID).Msg("error occured during xyz")
```

Using Str makes the log easy to read in the final log and in code as well. It keeps the message and where it originated seperate. We can also add the uniqueID we want as a `Str()` instead of as a formatted parameter in the message.

An advantage of this format is that Loki or other browsers can find the pattern and parse them making viewing them easier. If you use the JSON formatter, the Str() values come as a field in the resulting JSON object.

### Using Logs to Build Simple Metrics

We use Loki and Grafana to view our logs and system behaviour. Grafana allows you to build metrics on top of the messages/logs being generated - API response times, gRPC response times, error rate, etc. An excellent way to use this is to standardize your logs using the above mechanism

```
log.Info().Str("component", "gRPC Service1").Str("uniqueID", uniqueID).Msgf("%d gRPC Call Time in microseconds,time)
```

You can use `unwrap` in Grafana to parse the `%d gRPC Call Time in microseconds` message to extract the time and using the above `Str()` methods, its even easier to isolate the logs.

Using the same format of logging time can help you check across various services at once, and if you want to be granular, using the `component` Str() value can help.