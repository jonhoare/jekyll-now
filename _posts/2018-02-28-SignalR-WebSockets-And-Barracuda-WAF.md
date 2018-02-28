I have been tearing my hair out at work for the last day trying to resolve an issue with a web application that uses SignalR over WebSockets where traffic is directed through a Barracuda Web Application Firewall (WAF). Every attempt to connect to the SignalR/connect endpoint using websockets would fail with a 400 Bad Request. Here is how I eventually resolved the issue.

# tl; dr;
It looks like the barracuda WAF doesn't forward the `Connection` header on to the IIS Servers, even with `Enable Websocket` set to `Yes`. I had to configure a rewrite header rule to send this header onto my IIS Servers.

# Background
Traffic is directed first through a Barracuda Web Application Firewall and then sent on to a LoadBalanced set of Windows Server 2012R2 Virtual Machines. We are using IIS 8.5 which hosts a .Net Web Application with SignalR configured.

The entire application appears to run perfectly fine until I inspect the browsers network requests and I can see that I am getting a failed request to `/signalr/connect transport=webSockets`. As this connection failed, SignalR then reverted back to using `Long Polling` which is why my application appeared to continue to work.

I should point out that this application is already running perfectly fine on another set of VM's which don't have a Barracuda WAF infront of, so this leads me to think something is not configured correctly on the WAF.

# Investigation
## WAF and WebSockets
I spent many hours trying to debug the cause of the error, the first of which was a quick search of `barracuda waf websockets` which lead me to [this barracuda article](https://campus.barracuda.com/product/webapplicationfirewall/doc/49054741/how-to-enable-websocket) about how to enable websockets on the WAF.

Great! This must be what I am missing, so I go ahead and check my WAF service and sure enough `Enable WebSocket` is disabled! This must be why! I set this to `Yes`, save the changes and try again. Does this fix my issue? No!

## IIS Configuration
Next I move onto the IIS Server, have I configured this correctly?

I open up the `Server Manager` and `Roles and Features` to check whether the `WebSockets` feature has been enabled on the `IIS Role`. Yup!

## IIS Logging
I try to get some detailed errors from the IIS Server so I look at the IIS Logs and I can see GET requests being made to the `/signalr/connect` endpoint, indeed returning a Status code of `400`. Not very helpful, but does tell me that at least the traffic is making its way from the WAF to the IIS Server.

## Enable SignalR Debugging/Tracing
I decide to try and [enable debugging/tracing](https://docs.microsoft.com/en-us/aspnet/signalr/overview/testing-and-debugging/enabling-signalr-tracing) to see if I can get any information from the Server as to why it is returning a `400` status code.

I get some very useful information and find that during the SignalR initialization, a scan of assemblies occurs and it turns out we are missing a dependency in our application. I go ahead and fix up that dependency issue, the error disappears from the SignalR debugging logs, but I still get a `400` error with WebSockets connecting to `/signalr/connect`.

## Back to IIS Logs
After some more searching online I find some examples of how to get WebSockets working with `nginx`. Now we don't use `nginx` ourselves, but thought maybe this would give some insight into what I need to do with our WAF to get this to work.

To get this to work with `nginx` you need to configure your nginx server as below.

```conf
server {
    listen 80;
    server_name myhost;

    location / {
        proxy_set_header   X-Forwarded-For $remote_addr;
        proxy_set_header   Host $http_host;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_pass         "http://127.0.0.1:1234";
        proxy_http_version 1.1;
    }
}
```

With this information, I inspected my browsers request using the Network tab in Chrome and could see that I was sending up the following Headers.

```
Connection: Upgrade
Host: myhost
Upgrade: websocket
```

I decided to configure my IIS Server to output the Headers `Host`, `Upgrade`, `Connection` in the Logs to see if the Headers I was sending were making their way to the IIS Server.

After making a few failing requests I checked the logs and took a look at the headers that the IIS Server was receiving...

```
Host: myhost
Upgrade: websocket
Connection: -
```

It looks like the Barracuda WAF is stripping out the `Connection` header when forwarding the traffic onto my IIS Servers.

# Solution
So after some searching online I found out how to go ahead and Add a rewrite Header rule for the `Connection` header to ensure the WAF sent that on properly.

1. Open up the Barracuda WAF Management Site.
2. Go to Website -> Website Translations
3. Under `HTTP Request Rewrite`, Add a new Rewrite Rule...
   
   1. Specify a Rule Name.
   2. Set a Sequence Number.
   3. Select `Rewrite Header` as the Action.
   4. Enter `Connection` as the Header Name.
   5. Use `*` as the Old Value, which will replace any value.
   6. Enter `upgrade` as the Rewrite value, so that this is sent instead.
   7. Use `*` as the Rewrite condition.
   8. Click Add.

I try and load the application again and boom, connection successful to `/signalr/connect` with webSockets transport!

# Final notes
For websockets to work successfully, a handshake is initiated using the HTTP protocol and sends an `Upgrade` Header with value `websocket`. This tells the handling server that the connection should be upgraded, when a `Connection` header is sent, to this value `websocket`. When you send the `Connection` header with a value of `Upgrade`, this tells the server to Upgrade the Connection as per the `Upgrade` header.

Without passing through the `Connection` Header, the Servers would not upgrade the connection to `websocket` and so the Server responds with a `400` status code as it doesn't understand the request. As soon as I send the `Connection` Header, I get a `101 Switching Protocol` status code back as expected.

---

Hopefully this helps someone else out who is in a similar situation, tearing their hair out with WebSockets and Barracuda Web Application Firewalls... :)