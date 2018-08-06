# Microsoft Graph Kusto Guide
## Getting access
- Join `graphagspartners` for access to Graph Kusto logs
- All Graph deployments have their own Kusto DB except PROD and Canary share "public" Kusto
- Public cloud: [Playing with Kusto and MS Graph](https://microsoft.sharepoint.com/teams/microsoftgraph59/_layouts/OneNote.aspx?id=%2Fteams%2Fmicrosoftgraph59%2FSiteAssets%2FMicrosoft%20Graph%20Notebook&wd=target%28AGS%20Engineering%2FLivesite%2FLivesite%20tools.one%7CC7685BD6-D3B7-453E-A149-02276CEACDD5%2FPlaying%20with%20Kusto%20and%20MS%20Graph%7C27E4D16B-0969-4E79-A033-B22884799917%2F%29)
- Sovereigns: [Kusto in Mooncake and Blackforest](https://microsoft.sharepoint.com/teams/microsoftgraph59/_layouts/OneNote.aspx?id=%2Fteams%2Fmicrosoftgraph59%2FSiteAssets%2FMicrosoft%20Graph%20Notebook&wd=target%28AGS%20Engineering%2FLivesite%2FLivesite%20tools.one%7CC7685BD6-D3B7-453E-A149-02276CEACDD5%2FKusto%20in%20Mooncake%20%28Gallatin%2C%20China%5C%29%20and%20Blackforest%7C0EDDCF50-2E0F-41EE-BDEC-19B2133DA932%2F%29) and [Kusto in Fairfax](https://microsoft.sharepoint.com/teams/microsoftgraph59/_layouts/OneNote.aspx?id=%2Fteams%2Fmicrosoftgraph59%2FSiteAssets%2FMicrosoft%20Graph%20Notebook&wd=target%28AGS%20Engineering%2FLivesite%2FLivesite%20tools.one%7CC7685BD6-D3B7-453E-A149-02276CEACDD5%2FKusto%20in%20Fairfax%7CDFF4E973-798A-4CCC-B4FB-E226A3E2CA8D%2F%29)
- Download [Kusto.Explorer](http://aka.ms/Kusto.Explorer) or try the [Kusto web app](https://kustoweb.azurewebsites.net/msgraphkus/msgraphdb)
- Kusto docs at [http://aka.ms/kusto](http://aka.ms/kusto)
## Metrics pipeline
- Hot path vs warm path
- Kusto data is part of the warm path pipeline. Not used for alerts/monitors
- Expect data to be there in 3-5 minutes
## Looking up failed requests
Lookup requests with the request ID from either the error response body or the response headers.
```js
AggregatorServiceLogEvent
| where env_time > ago(1d)
| where correlationId == "3d83fa9a-5b9b-49ca-aa37-4c67e485c539"
```
MS Graph custom helper method:
```js
GetRequestLogs(
datetime(2018-07-25 10:31:57.8669192),
"3d83fa9a-5b9b-49ca-aa37-4c67e485c539") 
```
Datetime can be +/- 3 minutes.
## Determining impact
- Is it just me? My app? Just my tenant?
- Helps determine severity of ICM incident
Example error:
```
[Microsoft.Online.AggregatorService.Common.ServiceException: Connection did not succeed. Try again later.; at Microsoft.Online.AggregatorService.Controller.SimpleRequestWithAutodiscover.<Execute>d__3.MoveNext() in D:\bt\932496\repo\src\dev\Controller\Requests\SimpleRequestWithAutodiscover.cs:line 100
--- End of stack trace from previous location where exception was thrown ---
at System.Runtime.ExceptionServices.ExceptionDispatchInfo.Throw()
at System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(Task task)
at Microsoft.Online.AggregatorService.Controller.RequestController.<HandleRequest>d__0.MoveNext() in D:\bt\932496\repo\src\dev\Controller\Controllers\RequestController.cs:line 123;]
```
How many apps have hit this in the last day?
```js
AggregatorServiceLogEvent
| where env_time > ago(1d)
| where message contains "Microsoft.Online.AggregatorService.Common.ServiceException: Connection did not succeed. Try again later." 
| summarize count() by appId
```
Which tenants hit this error?
```js
AggregatorServiceLogEvent
| where env_time > ago(1d)
| where message contains "Microsoft.Online.AggregatorService.Common.ServiceException: Connection did not succeed. Try again later." 
| summarize count() by tenantId
```
How many failed requests?
```js
AggregatorServiceLogEvent
| where env_time > ago(1d)
| where message contains "Microsoft.Online.AggregatorService.Common.ServiceException: Connection did not succeed. Try again later." 
| count
```
Wrong! Use a tag to avoid double counting.
```js
AggregatorServiceLogEvent
| where env_time > ago(1d)
| where tagId == "306D7875" 
| where message contains "Microsoft.Online.AggregatorService.Common.ServiceException: Connection did not succeed. Try again later." 
| count 
```
How to find tags?
- ```TagInfo()``` Kusto function in public cloud Kusto
- [Searching AGS logs](https://microsoft.sharepoint.com/teams/microsoftgraph59/_layouts/OneNote.aspx?id=%2Fteams%2Fmicrosoftgraph59%2FSiteAssets%2FMicrosoft%20Graph%20Notebook&wd=target%28AGS%20Engineering%2FLivesite%2FLivesite%20tools.one%7CC7685BD6-D3B7-453E-A149-02276CEACDD5%2FSearching%20AGS%20logs%20-%20IFX%7CFF2082E5-60C5-458A-A539-C1473D959A12%2F%29)

## Latency Analysis
- Use the `['operations']` metric to examine how long AGS spent in each module
- `['operations']` is a JSON object so use a JSON viewer for easy analysis
```js
AggregatorServiceLogEvent
| where env_time > ago(1d)
| where correlationId == "3d83fa9a-5b9b-49ca-aa37-4c67e485c539"
| project ['operations'] 
| where operations != ""
```
## To be aware of
### Query costs
Only the last week of logs are stored on SSD and only 30 days of logs are kept in total. Queries for logs older than a week old may timeout when not optimized.
### `TrafficType`
```js
AggregatorServiceLogEvent
| where env_time > ago(15m)
| where trafficType == "ForkedCustomer"
| project correlationId 
| take 1 
```
Seeing ForkedCustomer traffic? Filter out traffic types other than customer.
### `ring`
Only see an issue in early rings? Might be a build reggresion.
```js
AggregatorServiceLogEvent
| where env_time > ago(1d)
| where message contains "Microsoft.Online.AggregatorService.Common.ServiceException: Connection did not succeed. Try again later." 
| summarize count() by appId, tenantId, ring
```
### `IIS logs`
Don't recommend digging around those.
- Can't join on correlationid
- Must use timestamp, region, role instance to narrow down and find request
```js
IISEventLogs
| where TIMESTAMP > ago(10m)
| take 1
```
### `PII scrubbing`
See a URL like `/lllll/llddldld-ldld-dddd-ddll-llddddddlddl?$select=llll.llll@llllllllllllllll.lllLlllll Lllllllll/dddd LLL/`? That means Graph couldn't parse it so PII could be anywhere. Check for escape characters in the URL.
```js
AggregatorServiceLogEvent
| where env_time > ago(1h)
| where incomingUri contains "/lllll/llddldld-" 
| project incomingUri, correlationId 
| take 1
```

## Graph builds
- Check a machines build number and ring with /ping. Ie. https://graph.microsoft.com/ping or https://graph.microsoft.us/ping

