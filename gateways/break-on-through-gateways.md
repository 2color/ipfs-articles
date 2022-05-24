# Break on Through with IPFS HTTP Gateways

<!-- ## Outline

- What is IPFS?
  - Definition
  - Concepts
    - CID
    - Peer to peer
    - Resilience - can’t be shut down
    - Easier caching with CID
  - Use-cases
    - Reading public data, e.g. NFTs
  - IPFS and the WWW
    - client-Server vs peer to peer
    - Trust
      - TLS: Certificates & encryption
      - Self verifiability
- Fetching data from IPFS
  - Low level vs high-level cloud services
    - Server, linux, nginx vs AWS S3/Cloudfront
    - IPFS Node vs Gateway
  - Fetching data from the network
    - How to use IPFS gateways
    - Public gateways
  - Resolution styles -->


The Interplanetary File System (IPFS) is a peer-to-peer protocol for storing and accessing files and websites. As distributed **peer-to-peer** protocol, it's fundamentally different from the HTTP protocol that forms the foundation for the internet.

<!-- Because files are at the heart of the internet and the internet is at the heart of everything, the potential use cases for IPFS are endless. -->

IPFS is a relatively new protocol compared to the time-honored HTTP protocol and isn't feasible for use in every scenario. The good news is that with the help of IPFS HTTP gateways, you can tap into the IPFS network directly from any browser.

This blog post will give an overview of the core concepts of the IPFS protocol, discuss the relationship between IPFS and HTTP(S), the role of IPFS gateways, and demonstrate how you can consume data from the IPFS network using HTTP without running any IPFS infrastructure.

If you're already familiar with the concepts of IPFS and would like to learn how to use IPFS gateways feel free to skip ahead to the [practical example](#TODO) section.

> Note: The blog post uses HTTP to refer to both HTTP and HTTPS for brevity and assumes that HTTPS should be used in every production application.

## The challenges with the client-server model

Typically, when you access a website, your browser uses several protocols to load the website:

- First, DNS is used by your browser to find the IP address of the server.
- Second, HTTP is used to request the website from the server.

Such interactions are characterized by the **client-server** model whereby your browser is a client interacting with an HTTP and DNS server.

While the client-server model has been the predominant model for the internet, it is fundamentally centralized and comes at the cost of resilience, reliance on gatekeepers, and single points of failure.

Practically speaking, a common challenge with the client-server model is that it puts all responsibility to ensure the availability of content on the server operator.

For example, when you open the following [URL of an image of Astronaut Jessica Watkins](https://www.nasa.gov/sites/default/files/thumbnails/image/04_iss067e033423.jpg) from the NASA website, you rely on the NASA server(s) being up.

When the server is down or unreachable, you won’t be able to access the image.

Moreover, the HTTP protocol does not specify a way to **ask other servers** for the image so the file is only available as long as the origin server hosts it.

> Note: In reality most websites rely on multiple servers that are load balanced to ensure high availability of content. But solutions to ensure high availability are not standardised as part of the HTTP protocol and are typically opaque to clients.

<!-- 
Would be good to also note that these solutionsremain under the control of a single entity, be that ISP,  CDN or other 'cloud' provider, which means if that single entity goes down, website goes down too. 

We should back this claim with multiple examples of Amazon's technical issues taking down huge chunks of the internet
I think this problem deserves own section, with a decade of examples:

2021: https://www.reuters.com/markets/commodities/amazons-prime-ring-other-apps-down-thousands-users-2021-12-07/

2017:  https://web.archive.org/web/20220523115642/https://www.vox.com/2017/3/2/14792636/amazon-aws-internet-outage-cause-human-error-incorrect-command

2011: https://web.archive.org/web/20210418175303/https://www.datacenterknowledge.com/archives/2011/04/21/major-amazon-outage-ripples-across-web/
2011: https://netflixtechblog.com/lessons-netflix-learned-from-the-aws-outage-deefe5fd0c04
2009: https://www.datacenterknowledge.com/archives/2009/06/11/lightning-strike-triggers-amazon-ec2-outage

2008: https://www.wired.com/2008/06/amazon-down/
Would be good to show it is not only Amazon, e.g. microsoft Azure:

2021: https://www.zdnet.com/article/global-azure-outage-knocked-out-virtual-machines-other-vm-dependent-services/

2018: https://www.datacenterknowledge.com/microsoft/azure-outage-proves-hard-way-availability-zones-are-good-idea

2014: https://www.datacenterdynamics.com/en/news/microsoft-confirms-azure-outage-was-human-error/

2012: https://www.datacenterdynamics.com/en/news/microsoft-misconfigured-network-device-led-to-azure-ou
Google Cloud:

2019: https://www.datacenterdynamics.com/en/opinions/google-cloud-platform-outage-analysis/

2018: https://www.datacenterknowledge.com/google-alphabet/google-cloud-has-disruption-bringing-snapchat-spotify-down
For best impact, extract names of household names like Spotify, netflix, reddit, etc  
-->

## From client-server to peer-to-peer with IPFS

One of the core characteristics of the IPFS is that it is a peer-to-peer network. In contrast to the client-server model where you typically have many clients consuming from a single server, with the peer-to-peer model, every computer (typically referred to as a _peer_) in the IPFS network can wear both the hat of a server and a client. This means that every IPFS peer can become a productive member of the network.

![client-server compared with peer-to-peer](https://cloudflare-ipfs.com/ipfs/bafybeiho4jont2vresctmrlec4zi7lawxfehbaut2b3kwsgiez7wug6lr4/http-vs-ipfs.png)


> Note: The article uses the terms **peer** and **node** interchangeably to refer to computers running the IPFS software.

As illustrated in the diagram, instead of relying on a single server at the center of the network that clients connect to, each peer connects to multiple peers. Since the `jpg` file is stored on three of the peers, two of those three nodes can be down and the file will still be accessible to the network. What's more, any number of peers become a provider for the `jpg` file, once they download it from the network.

In summary, with IPFS, nodes pool their resources, e.g., internet connection and disk space, and ensure that the availability of files is **resilient** and **decentralized**.

## Location addressing vs. content addressing

In IPFS, data is **content-addressed** rather than _location-addressed_ as is common in the client-server model of the web. To understand the difference between the two approaches, let's go back to the example with the image from NASA.

In the example with the image loaded from NASA, we used location addressing to fetch the image in the form of a URL. The URL contained all the location information to find and fetch the image:

- _scheme_: the protocol `https`.
- _hostname_: DNS name `www.nasa.gov` mapped to an IP address of the server.
- _path_ to the location on the server: `/sites/default/files/thumbnails/image/04_iss067e033423.jpg`

![location addressing](https://i.imgur.com/YKE6bRS.png)

The challenges with location addressing are numerous. We've all had the experience of going down an internet rabbit hole only to be abrupted by dead links because the link changed or the server is no longer hosting the files.

In a peer-to-peer network like IPFS, a given file might be hosted on a number of the IPFS nodes.

This is where _content addressing_ comes in handy. With IPFS, every single file stored in the system is addressed by a [cryptographic hash](https://en.wikipedia.org/wiki/Cryptographic_hash_function) of its contents known as a [**Content Identifier** or **CID**](https://docs.ipfs.io/concepts/glossary/#cid). The CID is a long string of letters and numbers that is unique to that file.

There are three crucial things to remember with regards to CIDs:

- Any difference (even a single bit) to the file will produce a different CID. This property is known as immutability.
- The same content added to two different IPFS nodes will produce the same CID (given the same parameters)
- A single CID can represent a single file or a folder of files, e.g. a static website. This property is known as "turtles all the way down".

![Content addressing](https://cloudflare-ipfs.com/ipfs/bafybeiaiuy36ffuutlf32bn64zmwwmhgpi2z33ambckfuzcjclpyowgvte/content-addressing.png)


The diagram illustrates what two different files look like on the network. The red jpeg represents one CID hosted on two nodes, while the purple jpeg represents a different CID hosted on two other nodes.

> **Note:** Depending on the size, IPFS may chunk (split) a single file into multiple blocks each with a CID of their own for efficiency reasons. Even so, the file will also have root CID. You can explore what this looks like for the NASA image using the [IPLD explorer](https://explore.ipld.io/#/explore/QmRKs2ZfuwvmZA3QAWmCqrGUjV9pxtBUDP3wuc6iVGnjA2).

### CIDs are reachable as long as there's at least one node pinning them

One of the benefits of content addressing is that you can retrieve a CID from any IPFS node as long as there's at least one node providing it to the network. This means that if you ask an IPFS node for a CID and the node doesn't have it, it can ask look on IPFS network and retrieve it.


### Content addressing caches better

The web as we know it today relies heavily on **caching** to ensure fast page loads and near instant interactivity. 

However, implementing caching correctly can be a challenging task due to the fact that websites contain a mix of mutable and immutable content. For example, an image loaded 

### Mutable vs. immutable
The content of modern websites can be divded into two category:
- Mutable
<!-- ## Finding files in IPFS using content addressing -->


## Speaking IPFS

Now that we’ve covered the core concepts of IPFS, it's important to note that IPFS is a set of open-source protocols, specifications, and software implementations.

So how do you use IPFS to access files in real-world applications?

There are two prominent ways to fetch files stored in the IPFS network:

- Running an IPFS node by installing one of the IPFS implementations as a daemon (long-running process) on your computer w
hich becomes a member of the IPFS peer-to-peer network and announces what data it’s holding and responding to requests for data.
- Using an **IPFS Gateway** which allows fetching CIDs using the HTTP protocol.

The first option allows you to _speak the IPFS protocol_ while the latter serves as a bridge in situations where you might be constrained to using HTTP. Choosing the right approach depends on your use case.

## What are IPFS gateways?

IPFS gateways are public services that translate between _Web2_ and _Web3_ thereby providing a bridge between HTTP and IPFS.

They allow you to use the HTTP protocol –which almost every programming language is capable of– to request a CID from the IPFS network, fetch it, and use HTTP to send the data back.

In its simplest form, a gateway is an IPFS node that also accepts HTTP connections in addition to speaking the IPFS protocol to participate in the peer-to-peer network. In fact, most IPFS implementations can also work as a gateway.

![Any IPFS node can also be a gateway](https://cloudflare-ipfs.com/ipfs/bafybeibezzvuhqwzz47yv4cg2hvyevuekcjr4uv7txp3frb5e44cr2bl2i/browser-gateway.png)

To use an IPFS gateway, you need to know two things:

- The address of the gateway, e.g. `https://ipfs.io/ipfs/[CID]`
- The CID (Content Identifier), e.g. `bafybeibml5uieyxa5tufngvg7fgwbkwvlsuntwbxgtskoqynbt7wlchmfm`

You can find public gateway operators in the [public gateway checker](https://ipfs.github.io/public-gateway-checker/) and check whether they are online and the latency from your location. Beware that many of the public gateways are provided on a best effort basis without any SLA. Follow along on how to ensure the reliable availability of your content.

<!-- needed because it's common for many internet access is common on a spectrum of devices ranging from resource-constrained IoT devices to powerful servers. -->

<!-- Some browsers such as Brave and [Opera](https://blogs.opera.com/tips-and-tricks/2021/02/opera-crypto-files-for-keeps-ipfs-unstoppable-domains/) and introducing new protocols to browsers can be a lengthy process. This is where IPFS Gateways come in handy. -->

## How to use IPFS gateways?

### Example fetching an image from a gateway

Let's take a look at what using an IPFS gateway looks like in practice, drawing on the example with the image of Astronaut Jessica Watkins. 

The image which was originally hosted on the NASA servers has been uploaded to the IPFS network, the corresponding CID for the image is `bafybeibml5uieyxa5tufngvg7fgwbkwvlsuntwbxgtskoqynbt7wlchmfm`

To fetch the image try one of the following links:
- https://ipfs.io/ipfs/bafybeibml5uieyxa5tufngvg7fgwbkwvlsuntwbxgtskoqynbt7wlchmfm
- https://cloudflare-ipfs.com/ipfs/bafybeibml5uieyxa5tufngvg7fgwbkwvlsuntwbxgtskoqynbt7wlchmfm
- https://nftstorage.link/ipfs/bafybeibml5uieyxa5tufngvg7fgwbkwvlsuntwbxgtskoqynbt7wlchmfm


### Resolution style

One of the key things to understand when it comes to using IPFS gateways is the two resolution styles that can be used to fetch content using gateway URLs:

- Path resolution where the CID is in the path portion of the gateway URL, e.g. `https://ipfs.io/ipfs/bafybeibml5uieyxa5tufngvg7fgwbkwvlsuntwbxgtskoqynbt7wlchmfm`. 
- Subdomain resolution where the cid a subdomain of the 


#### Path resolution 

A "path style" URL puts the IPFS CID into the path portion of the gateway URL, like this:

https://nftstorage.link/ipfs/bafkreied5tvfci25k5td56w4zgj3owxypjgvmpwj5n7cvzgp5t4ittatfy

If the CID points to a directory listing, you can append the name of a file within the directory to fetch the file:

https://nftstorage.link/ipfs/bafybeid4gwmvbza257a7rx52bheeplwlaogshu4rgse3eaudfkfm7tx2my/hi-gateway.txt

### Subdomain style URLs
A "subdomain style" gateway URL puts the CID into the host portion of the URL, as a subdomain of the gateway host, like this:

https://bafkreied5tvfci25k5td56w4zgj3owxypjgvmpwj5n7cvzgp5t4ittatfy.ipfs.nftstorage.link

If the CID points to a directory listing, you can use the path portion of the URL to specify the filename:

https://bafybeid4gwmvbza257a7rx52bheeplwlaogshu4rgse3eaudfkfm7tx2my.ipfs.nftstorage.link/hi-gateway.txt

This is the preferred style for serving web assets over HTTP gateways, because web browsers provide security isolation on a per-domain basis. Using the subdomain style, every CID gets its own "namespace" for things like cookies and local storage, which isolates things from other web content stored on IPFS.




### How do I ensure that CIDs I care about are available?

- Difference between a gateway and pinning service

https://github.com/ipfs/pinning-services-api-spec#readme

### How can I improve fetching/loading time


- Running a node
- Pinning with multiple pinning services
- (?) Periodically fetching CIDs from big public gateways

> it is kinda the opposite these days. we should note that this does not work on big public gateways because many of them run in "ephemeral, read-only mode",  meaning they do not announce CIDs that were read via them. so "prewarming" their cache does not add additional provider on DHT. When  performance matters, use a dedicated gateway. Avoid community-provided ones from public list, run your own or pay someone to run one for you, and put HTTP CDN in front of it.


<!-- Caching plays a big role in gateway performance. I think some of the pinning service providers offer caching as a paid service, but I can’t recall the exact latest details.


Example: NFT.storage has a new gateway that improves retrieval performance: nftstorage.link. This gateway “races” 3 public gateways (Pinata, Cloudflare, and ipfs.io) and also caches the majority of NFTs (>70% of them).
Individually, each gateway takes ~500ms-1.5s. Combining the racing + caching, overall performance on the NFTStorage gateway ends up being ~200ms.
More info:
https://nft.storage/blog/post/2022-03-08-gateway-release/
https://nft.storage/docs/concepts/gateways/#the-nftstorage-gateway
This is not helpful for the specific pathway you’re investigating (local node > public gateway), alas. But it is probably helpful in thinking about overall tradeoffs. (edited) 
 -->

### considerations for apps
- Loading time can vary depending on availability

> Lidel: where performance matters, add logic that "prewarms" HTTP CDNs/cache, and make sure etag and cache-control headers returned by http gateway are not lost. when go-ipfs 0.13 or later is used, leverage `X-Ipfs-Roots` header for even smarter HTTP cache invalidation.


## Post gateway future

<!-- 
 


Cloudflare
IPFS.io
NFTStorage.link

## Optimizing Load Time in Application Design

Efficient and extremely portable, IPFS is quickly becoming a lingua franca of Web3. 

Ultimately, users just want stuff to work. Images showing when they request them, datasets syncing seamlessly, etc.

(Why is IPFS more brittle?)
Gateway loading indicator, IPFS gateway waiting behaviors/status messages

NFTStorage.link – racing gateways, caching data

Do some performance testing and audits, include results here

## A post-gateway future?

Gateways are a necessary and useful compromise in today's web. Today, the majority of legacy Web2 browsers don't support IPFS -- they only speak HTTP(S), which [downsides]. To harness the full range of capabilities and benefits of a decentralized web, the vehicles we use must be upgraded to speak Web3.
 
If you're building a native desktop or mobile app, integrating IPFS capabilities from the get-go will help advance

Go-ipfs
Fleek SDK
Others?

Examples of apps that do this
Berty messenger
Valist package management
Audius
A js-ipfs example?
Brave Browser 

-->


## Ideas from call with Lidel

- IPFS is great for immutable content 
- IPFS content is fetched best when cached.
- Use relative links 
- Pin on multiple services
- HTTP Headers
    - HTTP Cache control plays well with  https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control
    - 
- Run your own node, or at the very least 
- use your own domain name (CNAME/proxying)
- Service worker hijacking ipfs requests that uses multiple gateways

## Summary

TODO