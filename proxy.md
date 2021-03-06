# What is Spacewalk Proxy?

Spacewalk Proxy (and downstream versions like Red Hat Satellite Proxy) provides four major benefits:


1. [Reverse-proxy](http://en.wikipedia.org/wiki/Reverse_proxy) style mirroring of content to speed up downloads or reduce bandwidth between clients and upstream Spacewalk server.
1. Single-point-of-exit to simplify networking / firewall rules between clients and upstream Spacewalk server.
1. Providing a place to store RPM files if they cannot be stored on the upstream server (for example, if connected to Red Hat Network (Note: proxy-to-hosted configurations are being deprecated))
1. Automatically install and configure a jabberd service so that Spacewalk can push osad messages down to the Clients even though there's no direct network connection between them.
## Why do I need Spacewalk Proxy for my Reverse Proxy?

Generic Reverse Proxy servers cache responses to GET requests. This is fine and good for most use cases, however for Spacewalk <-> Client communication you don't necessarily want every Client to be able to get some requests. Clients should only be able to get content that is applicable to them given their subscribed channels or registered organization, anything else would be wrong and mismatched for that client. Spacewalk Proxy wraps an authentication layer around a standard Reverse Proxy (Squid), enabling the custom Spacewalk Proxy pieces to handle authentication and access control such that the Squid's cached resources can be shared with all clients who are allowed to have them. This authentication layer wrapper also allows us to be aware of and respond with files that are on the local Proxy filesystem but not in Squid's cache, such as those that are uploaded through [rhn_package_manager]().
# How does it work?

(This section is primarily aimed at developers). There are two main components to Spacewalk Proxy, one on either side of the squid daemon. The Broker component recieves all incoming requests and verifies that the requesting client actually has access to the thing it's requesting. If so it checks the Spacewalk Proxy lookaside cache (see [rhn_package_manager]() secton, and responds with the file if it finds one. If not it forwards the request on to squid. Squid responds with a cached response if it has one, or if not the request falls out to the Redirect component which sends the request upstream to Spacewalk. Along the way there are two authentication tokens that must be exist and be valid. First the client must have (and Proxy must have cached) a client auth token that is associated with a list of authorized channels on the Proxy. Second the Proxy must have a Proxy auth token that the upstream Spacewalk server recognizes so that the Proxy can call special API methods on Spacewalk. Let's walk through some example flows.



               ------------ Spacewalk Proxy server ------------------
               |                                                     |
               |    ---- (POST or non-cached requests) ------------  |
               |    |                                              | |
    Client -----> Proxy(Broker) ----> Squid ----> Proxy(Redirect) ->---> Spacewalk server
               |      ^                                              |
               |      |                                              |
               |  lookaside cache                                    |
               |                                                     |
               ------------------------------------------------------

[Example 1:](=) Client issues GET request, no (or expired) Client Auth Token (or "Session Token"), no (or expired) Proxy Auth Token, no hit in lookaside cache, no hit in Squid cache. This flow is complicated, common, and shows the entire process, so let's start here.

1. Client issues GET request for something, let's say an RPM.
1. Proxy(Broker) recieves GET request
1. Proxy(Broker) notices that the Client Auth Token is expired (in rhnBroker.py:_local_GET_handler), returns rhnFault(34) to client.
1. Client recieves rhnFault(34), sends POST request to Proxy to get new auth token
1. Proxy(Broker) recieves POST request
 a. Proxy(Broker) copies request and sends to Spacewalk server (rhnShared.py:_proxy2server), including attaching the Proxy Auth Token to the request
  i. Spacewalk notices that Proxy Auth Token is expired, rejects request with rhnFault(1004)
  i. Proxy(Broker) recieves rhnFault(1004), makes new request to get new Proxy Auth Token
  i. Spacewalk recieves Proxy login request, reponds with new Proxy Auth Token
  i. Proxy(Broker) recieves new Proxy Auth Token, caches locally, retries original client login request with new Proxy Auth Token
 a. Spacewalk validates client's login request and responds with new Client Auth Token
 a. Proxy(Broker) recieves new Client Auth Token, caches locally, responds to Client POST request with token
1. Client recieves new auth token, tries original GET request again
1. Proxy(Broker) recieves GET request, verifies valid Client Token, verifies channel access is valid, does not find RPM in lookaside cache, decides to forward request to Squid.
 a. Proxy(Broker) copies request into new one (including Proxy Auth Token), sends to Squid
 a. Squid checks cache, finds no hit, forwards to Proxy(Redirect)
  i. Proxy(Redirect) recieves reqeust, copies into new one, sends new request to Spacewalk
  i. Spacewalk verifies request and responds with RPM
  i. Proxy(Redirect) recieves responds, responds to Squid
 a. Squid receives response, caches it, responds to Proxy(Broker)
1. Proxy(Broker) recieves response from Squid, responds back to Client
1. Client recieves the rpm, proceeds with install

[Example 2:](=) Client issues GET request, valid Client Auth Token and Proxy Auth Token, no hit in lookaside cache, no hit in Squid cache. Let's say that our client from [Example 1]() request another RPM, but this time all the tokens are already current / valid.

1. Client issues GET request for RPM.
1. Proxy(Broker) recieves GET request, verifies valid Client Token, verifies channel access is valid, does not find RPM in lookaside cache, decides to forward request to Squid.
 a. Proxy(Broker) copies request into new one (including Proxy Auth Token), sends to Squid
 a. Squid checks cache, finds no hit, forwards to Proxy(Redirect)
  i. Proxy(Redirect) recieves reqeust, copies into new one, sends new request to Spacewalk
  i. Spacewalk verifies request and responds with RPM
  i. Proxy(Redirect) recieves responds, responds to Squid
 a. Squid receives response, caches it, responds to Proxy(Broker)
1. Proxy(Broker) recieves response from Squid, responds back to Client
1. Client recieves the rpm, proceeds with install

[Example 3:](=) Client issues GET request, valid Client Auth Token and Proxy Auth Token, no hit in lookaside cache, hit in Squid cache. Let's say that another client (that already has valid auth token) requests same rpm as from [Example 1]().

1. Client issues GET request for RPM.
1. Proxy(Broker) recieves GET request, verifies valid Client Token, verifies channel access is valid, does not find RPM in lookaside cache, decides to forward request to Squid.
 a. Proxy(Broker) copies request into new one (including Proxy Auth Token), sends to Squid
 a. Squid checks cache, finds cache hit, responds to Proxy(Broker)
1. Proxy(Broker) recieves response from Squid, responds back to Client
1. Client recieves the rpm, proceeds with install

[Example 4:](=) Client issues GET request, valid Client Auth Token and Proxy Auth Token, hit in lookaside cache. Client is requesting an rpm that we've uploaded through [rhn_package_manager]().

1. Client issues GET request for RPM.
1. Proxy(Broker) recieves GET request, verifies valid Client Token, verifies channel access is valid, finds RPM in lookaside cache, responds back to Client
1. Client recieves the rpm, proceeds with install
## Proxy Headers

Let's take a moment to talk about some the various headers that you may find in Proxy requests.


* X-RHN-Proxy-Auth - the main one that transports the Proxy Auth Token(s). Token(s) may be plural because if there are multiple Proxies in a chained configuration then this will be a comma-separated list of Proxy Auth Tokens.
 * Each auth token will look something like: <proxy_server_id>:<user_id>:<issued_timestamp>:<expiration_lifetime>:<signature_hash>:<proxy_hostname_or_ip_address> (see rhnProxyAuth.py)
* X-RHN-Proxy-Version - it's there but not used. Ignore the value.
* X-RHN-Proxy-Auth-Error - a header that Spacewalk will stick in a response if the Proxy needs to update it's auth token. Valid types are 1003 (invalid token) or 1004 (expired token), anything else is an unhandled failure case.
* X-RHN-Proxy-Auth-Origin - the hostname or IP address of the Proxy that the auth error is intended for. So that for example the Proxy closest to the Spacewalk server doesn't eat all of the "your token is expired" messages intended for a Proxy that was chained and further away.
* X-RHN-IP-Path - contains an ordered list of all Proxy IP addresses that it flowed through, so that we can display the connection path in the Spacewalk UI.
## Client Headers

Headers used in Client <-> Spacewalk communication that we may care about in Proxy:


* X-RHN-Action - what are we doing? (for example, 'login').
* X-RHN-Server-ID - server ID of the client.
* X-RHN-Auth - the client's auth token.
* X-RHN-Auth-Channels - comma-separated list of channels that this client is allowed to access.
* X-RHN-Auth-Server-Time - time on the Spacewalk server when the auth response was sent.
* X-RHN-Auth-Expire-Offset - how long the token will be valid.
## Auth Tokens

Both types of cached auth tokens (client and proxy) are stored in /var/cache/rhn/proxy-auth/. Proxy tokens will have filenames of p<proxy_system_id><hash>, whereas the client auth tokens will have filenames of just the client's system_id.

## rhn_package_manager

rhn_package_manager is the tool that you use to put RPMs in Proxy's lookaside cache. There are two main uses for it:


1. If you cannot or do not want to have the RPMs available on the upstream server (one possible scenario is if you are registered upstream to Red Hat Network then you cannot upload RPMs, however that is being deprecated).
1. To [pre-fill](proxy-precache) a permanent cache with RPMs that you want to be available without ever requiring them to be downloaded over the network from the upstream server.

Either way the tool functions similarly, placing the RPM file in the lookaside cache (/var/spool/rhn-proxy/rhn) so that the Proxy(Broker) can find it if a client is authorized to download it. In the first use case rhn_package_manager also uploads the RPM header (but not the file itself) up to Spacewalk so that Spacewalk can do things like calculate available upgrades for Servers and regenerate the yum metadata files (required for the RPM to be visible to yum).

rhn_package_manager inherits a lot from the rhnpush utility, which is more generic. A common source of problems is that the rhnpush that is installed on the Proxy is not compatible with the rhn_package_manager (rpm name: spacewalk-proxy-package-manager) that is installed.
## Lookaside Cache Files

Let's briefly talk about some of the files that you may find in the lookaside cache (/var/spool/rhn-proxy/). If you have pushed packages with rhn_package_manager there will be an rhn/ directory there that contains the files on a structured directory tree (that happens to look exactly like the directory tree in /var/satellite/rhn/ on the Spacewalk server, in case you wanted to mirror it).


There will also be a list/ directory there (assuming Spacewalk Proxy 2.2 or higher) that contains cached descriptions of rpms in channels or channels associated with kickstarts. This is necessary because Proxy does not have any sort of database by itself, so naturally it would only know information that is in the incoming request or cache token files. However it needs to be able to differentiate between files that have the same name but different checksums in the lookaside cache (example: a signed and an unsigned copy of the same RPM) and respond with the correct file, and the GET request from yum only contains filename and channel label, not file checksum. These cached list files enable Proxy(Broker) to correctly evalutate if the requested file is actually in the lookaside cache. They should be created / updated / maintained automatically.
# Links to related pages

* [DebuggingProxy](DebuggingProxy)

* [HowToInstallProxy](HowToInstallProxy)
* [proxy-precache](proxy-precache)