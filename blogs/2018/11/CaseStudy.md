# Case study - Intermittent "The server returned an invalid or unrecognized response." error

## Issue
A web application was hosted in Azure App Service. Intermittently the web application threw the following exception:
```
System.Net.Http.HttpRequestException: An error occurred while sending the request. ---> System.IO.IOException: The server returned an invalid or unrecognized response.
   at System.Net.Http.HttpConnection.SendAsyncCore(HttpRequestMessage request, CancellationToken cancellationToken)
   --- End of inner exception stack trace ---
   at System.Net.Http.HttpConnection.SendAsyncCore(HttpRequestMessage request, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.SendWithNtConnectionAuthAsync(HttpConnection connection, HttpRequestMessage request, Boolean doRequestAuth, CancellationToken cancellationToken)
   at System.Net.Http.HttpConnectionPool.SendWithRetryAsync(HttpRequestMessage request, Boolean doRequestAuth, CancellationToken cancellationToken)
   at System.Net.Http.RedirectHandler.SendAsync(HttpRequestMessage request, CancellationToken cancellationToken)
   at 
   ------customer code starts here------
   ...
```

## Troubleshooting
### Clarifications & possible causes
The inner exception: `System.IO.IOException`, with an error message: *"The server returned an invalid or unrecognized response."* didn't give us much details of the context when the error happened. Possible causes could be:
1. The web application didn't submit a valid request, or a bug in the its code.
2. A network issue.
3. The target backend service returned an invalid response.

In order to clarify it, I asked customer a few questions regarding the symptom of the issue firstly:

1. Q: How often did the error happen.

    A: Usually no more than once per hour.

2. Q: To which URL was the request sent, when it encountered the error?

    A: Multiple ones, which were of independent backend services.

3. Q: And any idea with the "Unrecognized Response"?

    A: There wasn't any error response was recorded from the backend service side.

Customer also shared me that he had followed [https://github.com/dotnet/corefx/issues/33594](https://github.com/dotnet/corefx/issues/33594) and set environment variable *DOTNET_SYSTEM_NET_HTTP_USESOCKETSHTTPHANDLER* to *false*. That altered the call stack of the error, but the error still happened and was the same.

With the clarifications above, I suspected network could be mostly the cause, due to
1. Altering the code couldn't fix the issue.
2. Also the issue happened to mutiple independent backend services.

### Code review
Customer asked if we needed to capture a memory dump to the exception as the next troubleshooting step. I thought about it. Whether a dump would be useful, depended on where and how the exception was thrown from the code. So I decided to review the code first for where and how. I only needed to review the code of function `System.Net.Http.HttpConnection.SendAsyncCore()`, which was on the top of the above call stack and threw the exception.

Based on the code listed at [https://github.com/dotnet/corefx/blob/master/src/System.Net.Http/src/System/Net/Http/SocketsHttpHandler/HttpConnection.cs#L499](https://github.com/dotnet/corefx/blob/master/src/System.Net.Http/src/System/Net/Http/SocketsHttpHandler/HttpConnection.cs#L499) The only where that threw an `IOException` in `SendAsyncCore()` was:
```csharp
                // When the connection was taken out of the pool, a pre-emptive read was performed
                // into the read buffer. We need to consume that read prior to issuing another read.
                ValueTask<int>? t = ConsumeReadAheadTask();
                if (t != null)
                {
                    int bytesRead = await t.GetValueOrDefault().ConfigureAwait(false);
                    if (NetEventSource.IsEnabled) Trace($"Received {bytesRead} bytes.");

                    if (bytesRead == 0)
                    {
                        throw new IOException(SR.net_http_invalid_response);
                    }

                    _readOffset = 0;
                    _readLength = bytesRead;
                }
```
This was the first read attempt to the response, after sending and flushing the request. And if total bytes read was 0, an `IOException` would be thrown, with an error code of `net_http_invalid_response`.

This `IOExceptinon` would then be caught by the following code at the bottom of the same function, which wrapped the `IOException` with an HttpRequestException and re-threw.
```csharp
            catch (Exception error)
            {
...
                else if (error is IOException ioe)
                {
                    // For consistency with other handlers we wrap the exception in an HttpRequestException.
                    // If the request is retryable, indicate that on the exception.
                    throw new HttpRequestException(SR.net_http_client_execution_error, ioe, _canRetry);
                }
                else
                {
                    // Otherwise, just allow the original exception to propagate.
                    throw;
                }
            }
```

I could read that:
1. This code matched the exception structure in the error message.
2. Dump wouldn't help in this case.

    The exception was thrown as the **result** of the 0 byte issue. A dump triggered by that exception wouldn't tell me  the reason for the 0 byte response.

### Conclusion
With all the clarifications and code review above, possible causes of the 0 byte response issue should be:
1.	Request timed out, nothing was returned from the backend service before it timed out.
2.	The connection was forcefully closed before the web application received any payload.

    It could be closed from either the source side, the destination or a network device between them.

I discussed the findings with customer and he confirmed that there wasn't any long running request recorded in the backend servers' logs.

After elimiting all impossible causes, the only remaining one was due to the Internet connectoin between the web application and the backend services.

## Suggestions
An http call over Internet could fail potentially. Internet connections were not stable all the time by its nature. Our code needed to be prepared for it and a retry could mitigate a temporarily Internet connectivity issue mostly. 

Actually **Retry** is an important design pattern. [https://docs.microsoft.com/en-us/azure/architecture/patterns/retry](https://docs.microsoft.com/en-us/azure/architecture/patterns/retry)
>An application that communicates with elements running in the cloud has to be sensitive to the transient faults that can occur in this environment. Faults include the momentary loss of network connectivity to components and services, the temporary unavailability of a service, or timeouts that occur when a service is busy.

Similarily, applications that connect to a cloud service such as Azure SQL Database should expect periodic reconfiguration events and implement retry logic to handle these errors instead of surfacing these as application errors to users.
 
[https://docs.microsoft.com/en-us/azure/sql-database/sql-database-troubleshoot-common-connection-issues#steps-to-resolve-transient-connectivity-issues](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-troubleshoot-common-connection-issues#steps-to-resolve-transient-connectivity-issues)

[https://docs.microsoft.com/en-us/azure/sql-database/sql-database-connectivity-issues](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-connectivity-issues)
 
## More information
There are several open issues in GitHub, regarding this `System.Net.Http.HttpRequestException`. For instances:
[https://github.com/dotnet/corefx/issues/33594](https://github.com/dotnet/corefx/issues/33594)

[https://github.com/dotnet/corefx/issues/30691](https://github.com/dotnet/corefx/issues/30691)

It is common to have different ideas and solutions for the same error message or exception type in different cases. Hence it is important to clarify how the error happens, like what we did in this case, before making any assumption to the root cause.
