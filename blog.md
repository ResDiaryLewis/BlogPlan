My team lead, Adam, [wrote a blog](https://medium.com/resdiary-product-team/resdiary-au-azure-migration-5c1bc13b201d) recently concerning how we migrated our AU servers from RackSpace to Azure. In that post (underneath him calling me out), he mentioned how important it was for us to gauge how our potential new infrastructure would compare to that which it would replace. This post will elaborate slightly on our load testing strategy and experience using GoReplay, as well as some details on using GoReplay middleware.

The ultimate goal of our loadtesting was to migrate to a set of infrastructure (virtual machines, database, and caches) which could handle our expected traffic without over-provisioning and incurring more expense than necessary. Over-provisioning, or under-provisioning, migration infrastructure is typically a symptom of a poor loadtesting strategy. If you don't feel confident that your loadtest results are representative of your real traffic, you will often end up overshooting the capacity of the new infrastructure. You'll then either tolerate the excess expense incurred or have to plan to scale down, which you're liable to get wrong again without a good loadtesting strategy. So, it's important to plan appropriately for your use case, and to pick the right tool.

We also wanted to use our loadtests as an opportunity to test our new monitoring stack. If we could adequately analyse the loadtests using the new monitoring tools, then we'd know that they were suitable for the job.

There are a few possible approaches to loadtesting your new servers:
- Manually:
  Manual loadtesting has the benefit of an intelligent (YMMV) user making requests in a manner that they know will cover different branches of the application. The tester can make realistic requests that will highlight any glaring errors with the new architecture. However, the obvious problem with this approach is that your throughput will be limited to the input speed of the testers (who may experience extreme boredom depending on the length of your test).

- Scripting:
  Scripted tests can typically provide as high a throughput as you desire, allowing you to stress test your infrastructure with many requests. However, the diversity of these requests relies upon well designed tests that cover different branches of the application. Otherwise, you could end up repeating a request that your application can cache, or that isn't very resource intensive. You would need to consider how these repeated requests are handled by your database and/or cache too.

You likely won't ever be fully confident that your manual or scripted load tests are _really_ representative of real traffic. What we really want to test with is:

- Live Traffic:
  If we were to test with live traffic, we would know that our tests throughput and request diversity were exactly as they would really be for that period. GoReplay allows us to capture and replay traffic that was intended for our production infrastructure to our testing infrastructure. You can even modify our throughput with GoReplay by capturing live requests to a file, then replaying them later at a modified speed, or rate limit the replayed traffic (unfortunately this wasn't applicable for us as our application is time sensitive).
  As mentioned by Adam in his post, it's imperative to ensure that your test servers won't produce damaging side effects as a result of them receiving live traffic. Our application ties in with many external services, payment providers for example, that could have disastrous side effects if triggered in two different instances. 

## Installing and running GoReplay
As mentioned in Adam's post, we installed GoReplay on two HAProxy instances in order to intercept and replay the traffic. Note that these are Linux machines. As far as I can tell, [the Windows flavour](https://github.com/buger/goreplay/wiki/Running-on-Windows) of GoReplay appears to suffer as a result of having to support Windows. 
Installing GoReplay simply involves grabbing the latest binary from the [releases page on GitHub](https://github.com/buger/goreplay/releases):
```shell
$ wget https://github.com/buger/goreplay/releases/download/v0.16.1/gor_0.16.1_x64.tar.gz
$ tar -xzf gor_0.16.1_x64.tar.gz
$ ./goreplay -h
```
The [GoReplay documentation](https://github.com/buger/goreplay/wiki) covers most use cases of the tool. For example, here's how we ran it to listen to HTTP traffic and replay it:
```shell
sudo nohup ./goreplay --input-raw :80 --output-http http://test.server < /dev/null > http.gor.log 2>&1 &
```
Where:
- `sudo` runs as root, which is necessary to listen to the network traffic [(unless you configure it otherwise)](https://github.com/buger/goreplay/wiki/Running-as-non-root-user). 
- `nohup` allows the command to ignore the _hangup signal_, which is triggered when you log out of the machine. We found this useful for long load tests.
- `--input-raw` listens to the port specified (`80`).
- `--output-http` replays traffic from any inputs to the given address (`http://test.server`).
- `< /dev/null` has the GoReplay process listen for input from `/dev/null` which never gives input but keeps the process open. Otherwise, the GoReplay process will expect input from `stdin` which won't be open while the process is in the background.
- `> http.gor.log` redirects any process from the output to a log file called `http.gor.log` so that we can refer to it if we need to, either during or after the load test. 
- `2>&1` redirects the `stderr` output to `stdout`, allowing us to collect error output in the same log file as specified above.
- `&` runs the process in the background. So that we can continue to use the shell while we run GoReplay.

We also wanted to replicate HTTPS traffic, but this proved to be a bit more problematic. [GoReplay can't simply replay encrypted requests](https://github.com/buger/goreplay/issues/529), so we had to modify our HAProxy config with this in mind. Our HAProxy config previously looked like this:
![HAProxy HTTP Setup](HAProxy_HTTP.png "HAProxy HTTP Setup")
We modified the config to look like this:
![HAProxy HTTPS Setup](HAProxy_HTTPS.png "HAProxy HTTPS Setup")
By decrypting the traffic in the SSL frontend, then outputting the decrypted traffic to an intermediate port, we can configure GoReplay to listen to the intermediate port and replay the traffic as HTTPS to the test servers. The intermediate frontend then routes the traffic to the SSL backend. So we've effectively looped the traffic back into HAProxy to allow GoReplay to listen unencrypted traffic.

Here's an example of this in code:
```
sample_haproxy.cfg
```
Once we've setup our HAProxy config this way, we can listen to the intermediate port:
```shell
sudo nohup ./goreplay --input-raw :2802 --output-http https://test.server < /dev/null > https.gor.log 2>&1 &
```
The only differences we've made to the command above are:
- We're now listening to a different port (our intermediate port - `2802`)
- We're outputting to `https://`
- We're logging to a different file

~~After running both of these commands, we started to monitor the results of our loadtesting using a mixture of our new and old monitoring tools. Previously we used Stackify for performance monitoring and aggregating logs, but we're in the process of phasing it out; so we used Grafana for performance monitoring and Stackify to analyse any logs until we implement an alternative tool.~~

Initially, we performed a dry run of our load testing process for a short period of time during off-peak hours. The first problem that we encountered with replaying live traffic to the test servers concerned ASPX session cookies. For some requests made to the live server, the application will often return a [`Set-Cookie` header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie), which instructs the users browser to assign a `<cookie-name>=<cookie-value>` pair. So, if we replicate the same HTTP request between the live and test servers where they both generate distinct session IDs, we will encounter `401` response codes.

_Enter GoReplay middleware_. [Middleware](https://github.com/buger/goreplay/wiki/Middleware) allows you to modify the requests that you replay to your test servers based on the original request and replayed response. By designing your own middleware, you can fit GoReplay to more effectively fulfil your needs. It can take the form of any executable In this case, we can map session IDs from the live server to those from the test server, then modify replayed requests with the mapped ID. We used [this example](https://github.com/buger/goreplay/blob/master/examples/middleware/token_modifier.go) from the GoReplay repository to build upon.
The basic algorithm involved looks like this:
![Middleware Flowchart](middleware_flow.png "Middleware Flowchart")
Since middleware has no guarantee that any payload type (request, original response, replayed response) will arrive in any specific order, we map original session IDs to replayed sessions IDs in two maps. The mapping function is structured like this:
```go
// sessionID   - ASPX sessionID returned in this payload
// reqID       - Identifies membership of the same request triple, so
//               so any original response or replayed response responding 
//               to the same request will have the same reqID
// payloadType - One of {request, originalResponse, replayedResponse}, 
//               though requests shouldn't reach this function.
func sessionIDMapping(sessionID, reqID string, payloadType byte) {
	// If this request has already logged an opposite session ID
	if otherSessionID, ok := asyncRequestIDQueue[reqID]; ok {
		switch payloadType { // map original -> replayed
		case payloadOriginalResponse: // if it's the original response
			Debug(fmt.Sprintf("Mapping %s to %s", sessionID, otherSessionID))
			sessionIDToReplaySessionID[sessionID] = otherSessionID
		case payloadReplayedResponse: // if it's the replayed response
			Debug(fmt.Sprintf("Mapping %s to %s", otherSessionID, sessionID))
			sessionIDToReplaySessionID[otherSessionID] = sessionID
		}
		delete(asyncRequestIDQueue, reqID) // delete entry from the async map for clarity
	} else { // if this request ID hasn't been logged
		Debug(fmt.Sprintf("Mapping request ID %s to %s", reqID, sessionID))
		asyncRequestIDQueue[reqID] = sessionID
	}
}
```
This function asynchronously handles responses or replayed responses setting session ID cookies. Since GoReplay has each triple (Request, Response, ReplayedResponse) share a request ID, this function has the first response to reach the middleware map its session ID. When the second response arrives we then have access to both session IDs and we can map the original to the replayed session ID with the knowledge of which one is which (since the second response type is available). An unlikely edge case can occur where the request will arrive before either response, this is unhandled and the request will proceed to be replayed with the wrong session ID.

