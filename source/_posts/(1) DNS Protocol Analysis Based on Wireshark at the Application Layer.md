---
title: (1) DNS Protocol Analysis Based on Wireshark at the Application Layer
date: 2022-12-31 17:57:38
tags: wireshark 网络 服务器
categories: NetWork
---

<!--more-->


## What is DNS
DNS (Domain Name System) is the English abbreviation of "Domain Name System". It is a computer and network service naming system organized into a domain hierarchy. It is used in TCP/IP networks. The job of converting domain names to IP addresses.
When port 53 of the computer uses TCP/UDP domain name request query, if the data packet is less than 512 bytes, UDP is used; if the returned request size is greater than 512 bytes, the two sides of the interaction will negotiate to use TCP.
## DNS Domain Name Space Structure
The Domain Name System acts as a hierarchical and distributed database that contains various types of data, including hostnames and domain names. The names in the DNS database form a hierarchical tree structure called a domain namespace.
![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/d288b338ff2f4644b59c395fc63e9c94_1720253884434.png?token=ANB4BCKJ752YW2BBC2JN25LGRD67Y)

- Root domain: In the use of DNS domain names, it is stipulated that the trailing period '.' is used to specify the domain hierarchy whose name is at the root or a higher level.

- Top-level domain: used to indicate a country, region or organization. Use three characters, such as com -> commercial companies, edu -> educational institutions, net -> Internet companies, gov -> non-military government agencies, and so on.

- Second-level domain: The registered name used by an individual or organization on the Internet. Use two characters, such as: cn -> for China, jp -> Japan, uk -> United Kingdom, hk -> Hong Kong and so on.

- Host: The host name is at the bottom of the domain name space structure. The combination of the host name and the domain name constitutes the FQDN, and the host name is the leftmost part of the FQDN. For example, FQDN: (Fully Qualified Domain Name) fully qualified domain name: a name with both a host name and a domain name. (via the symbol ".") For example: the host name is "bigserver", the domain name is "mycompany.com", then the FQDN is "bigserver.mycompany.com".


## Three DNS Acquisition Process
DNS is an application layer protocol. In fact, it also works for other application layer protocols, including but not limited to HTTP, SMTP and FTP, and is used to resolve the hostname provided by the user into an IP address.

The specific process is as follows:

1. The DNS client is running on the user host, that is, our PC or mobile client is running the DNS client.

2. The browser extracts the domain name field from the received url, which is the accessed host name, such as http://www.baidu.com/, and transmits the host name to the client of the DNS application.

3. The DNS client sends a query message to the DNS server, which contains the field of the host name to be accessed (including some column cache queries and the work of the distributed DNS cluster).

4. The DNS client will finally receive a reply message, which contains the IP address corresponding to the host name.

5. Once the browser receives the IP address from the DNS, it can initiate a TCP connection to the HTTP server located by the IP address.
## DNS Service Architecture
The role of the DNS service: resolve the domain name to an IP address, and resolve the IP address to a domain name.

Assume that some applications (such as web browsers or mail readers) running on the user's host need to convert the host name to an IP address. These applications will call the client side of DNS and specify the hostnames that need to be translated. (On many UNIX-based machines, the application needs to call the function gethostbyname() in order to perform this conversion). After receiving the DNS client of the user host, it sends a DNS query message to the network. The UDP datagrams used by all DNS request and answer messages are sent through port 53. After a delay of several ms to several s, the DNS client on the user host receives a DNS answer message that provides the desired mapping. The result of this query is passed to the application calling DNS. Therefore, from the perspective of invoking applications on the user's host, DNS is a black box that provides simple and direct conversion services. But in fact, the black box that implements this service is very complex. It consists of a large number of DNS servers distributed around the world and an application layer protocol that defines the communication method between DNS servers and query hosts.

## DNS Adopts Distributed Cluster Work Mode
A simple design pattern of DNS is to use only one DNS server on the Internet, which contains all the mappings. In this centralized design, the client directly sends all query requests to a single DNS server, and the DNS servers respond directly to all querying clients. As tempting as this design approach is, it doesn't work with the current Internet. Because today's Internet has a huge and growing number of hosts, this centralized design has problems with single point of failure, communication capacity, long-distance time delay, and high maintenance overhead.

DNS servers are generally divided into three types, root DNS servers, top-level DNS servers, and authoritative DNS servers.
## DNS Working Process
When a DNS client needs to look up a name used in a program, it queries a local DNS server to resolve the name. Each query message sent by the client includes 3 pieces of information to specify the question the server should answer.

- The specified DNS domain name, expressed as a Fully Qualified Domain Name (FQDN).

- The specified query type, which can specify a resource record by type, or as a specialized type of query operation.

- The specified category of DNS domain names.

For DNS servers, it should always be specified as the Internet category. For example, the specified name can be a computer's fully qualified domain name, such as im.qq.com, and the specified query type is used to search for address resource records by this name.

DNS queries are resolved in a variety of different ways. Clients can also sometimes answer queries in-place by using cached information obtained from previous queries. A DNS server can answer queries using its own cache of resource record information, or it can query or contact other DNS servers on behalf of the requesting client to fully resolve the name, and then return an answer to the client. This process is called recursion.

Alternatively, the client itself may try to contact other DNS servers to resolve the name. If the client does this, it uses independent and additional queries based on the server's answers. This process is called iteration, that is, interactive queries between DNS servers are iterative queries.

The DNS query process is as follows:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/ca97b1c38431427daa33eacc68b2da85_1720253884434.png?token=ANB4BCIVOTM77MXCUXLE7DDGRD674)
1. Enter the www.qq.com domain name in the browser, and the operating system will first check whether its local hosts file has this URL mapping relationship. If so, it will first call this IP address mapping to complete domain name resolution.

2. If there is no mapping of this domain name in hosts, search the cache of the local DNS resolver to see if there is a mapping relationship of this URL. If so, return directly to complete the domain name resolution.

3. If there is no corresponding URL mapping relationship between hosts and the local DNS resolver cache, it will first find the preferred DNS server set in the TCP/IP parameters. Here we call it the local DNS server. When this server receives a query, if you want to The queried domain name is included in the local configuration area resources, and the resolution result is returned to the client to complete the domain name resolution, which is authoritative.

4. If the domain name to be queried is not resolved by the local DNS server area, but the server has cached the URL mapping relationship, then call this IP address mapping to complete the domain name resolution, which is not authoritative.

5. If the local zone file and cache resolution of the local DNS server are invalid, then query according to the settings of the local DNS server (whether to set a forwarder). If the forwarding mode is not used, the local DNS will send the request to 13 root DNS, the root After receiving the request, the DNS server will determine who is authorized to manage the domain name (.com), and will return an IP responsible for the top-level domain name server. After the local DNS server receives the IP information, it will contact the server responsible for the .com domain. After the server responsible for the .com domain receives the request, if it cannot resolve it, it will find a lower-level DNS server address (http://qq.com) that manages the .com domain to the local DNS server. When the local DNS server receives this address, it will look for the http://qq.com domain server, repeat the above actions, and query until it finds the www.qq.com host.

6. If the forwarding mode is used, the DNS server will forward the request to the upper-level DNS server, and the upper-level server will analyze it. If the upper-level server cannot resolve it, it will either find the root DNS or forward the request to the upper-level , and cycle through it. Regardless of whether the local DNS server uses forwarding or root hints, the result is finally returned to the local DNS server, and the DNS server then returns to the client.

From the client to the local DNS server is a recursive query, and the interactive query between DNS servers is an iterative query.
## DNS Message Format
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/2abc7a7358854aa4a87c5a86ae36a78e_1720253884434.png?token=ANB4BCIMFUU3DPPYVFGHHUTGRD676)
- 16-bit identifier: Mark a bunch of DNS queries and responses to distinguish which DNS response corresponds to which DNS query response
- 16 flag bits:
	![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/927e70b564b647baaf2e9217a49420b5_1720253891632.png?token=ANB4BCPQBMQAK47VXQLRWJTGRD7AC)
>- a. QR: query/response flag, 0 means this is a query message, 1 means a response message
>- b. opcode: Define the type of query/response, 0 means standard query, 1 means reverse query (obtain host domain name from IP address), 2 means request server status
>- c. AA: Authorization response flag, only used by the response message. 1 indicates that the DNS service provider is an authorized server
>- d. TC: Truncation flag, only used by response messages. Because UDP datagrams are limited in length, DNS packets that are too long will be truncated. For 1, the DNS message exceeds 512 bytes and is truncated
>- e. RD: Recursive query flag, 1 table performs recursive query: if the target DNS server cannot resolve a certain host name, it will continue to query other DNS servers, so recursively, until the result is obtained and returned to the client; 0 Indicates iterative query: If the target DNS server cannot resolve a certain host name, it will return the IP addresses of other DNS servers it knows to the client, and let the client decide whether to continue sending requests to other DNSs.
>- f. RA: Allow recursion flag, only used by response message. 1 indicates that DNS supports server recursive query
>- g. ZERO: These 3 bits are unused and must be set to 0
>- h. rcode: 4-digit return code, indicating the response status. 0 means no error, 3 means the domain name does not exist


For inquiries:
- 16-digit number of questions: generally includes a query question

- The number of 16-bit response resources, the number of 16-bit authorized resource records, and the data of 16-bit additional resource records: at least 1 in the query report;

For acknowledgments:

- the number of response resource records for the response message: at least 1,

- Authorized resource, additional resource record data: can be 0 or non-0

The query question format is:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/e97d9043c82d459d95828c5fbdec49e1_1720253891632.png?token=ANB4BCKBCDFA4QB4R6P36PDGRD7AE)
The query name encapsulates the host name to be queried in a certain format, and how to perform query operations in the query type table:

>- a. Type A, the value is 1: the table obtains the IP address of the destination host
>- b. Type CNAME, the value is 5: the table gets the alias of the target host
>- c. Type PTR, value 12, table reverse query (obtain domain name by IP address)
>- d. The query type is generally 1, and the table obtains the IP address.

Response, authorization, and additional information fields all use the Resource Record (RR) format:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/7e344ad50b934b28bc5be0bbc864c94e_1720253891632.png?token=ANB4BCNPOMGKRVNGZOH35J3GRD7AO)

>- domain name, type, class: the name corresponding to the resource, the format is the same as that of the query question
>- Survival time: how long the record result of the query can be cached by the local client, the unit is S
>- resource data length, resource data: depends on the type field. For type A, the resource data is a 32-bit IPv4 address, and the resource data length is 4 bytes

## Wireshark Capture Packets
WireShark is mainly divided into these interfaces:

- Display Filter (display filter), used to set filter conditions for packet list filtering. Menu path: Analyze --> Display Filters.
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/b6d6c2f7f0f34b4cbed6ef8698746255_1720253891632.png?token=ANB4BCPG4DUKLNCZKAAFE2LGRD7AS)

- Packet List Pane (data packet list), display the captured data packets, each data packet contains number, time stamp, source address, destination address, protocol, length, and data packet information. Packets of different protocols are displayed in different colors.
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/bb220bc7190744d09b76724ac96f405c_1720253891632.png?token=ANB4BCM26NA6T62SFHHUIITGRD7AU)
- Packet Details Pane (packet details), select the specified packet in the packet list, and all the detailed information of the packet will be displayed in the packet details. The packet details panel is the most important, used to view every field in the protocol. The lines of information are:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/4203da069bf44d7983a1dc80d102e399_1720253902706.png?token=ANB4BCPZYSUTMIMSBMPODEDGRD7AY)

(1) Frame: Overview of data frames at the physical layer
(2) Ethernet II: Data link layer Ethernet frame header information
(3) Internet Protocol Version 4: Internet layer IP packet header information
(4) User Datagram Protocol: The data segment header information of the transport layer T, here is UDP
(5) Domain Name System: Application layer information, here is DNS

### Grab the DPU of DNS first
Since DNS uses port 53 of the computer, we directly write `udp port 53` in the filter:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/682ebdc7e4974464bd9565c54f640245_1720253902706.png?token=ANB4BCPL2MAOMGN3TFDZY6LGRD7A2)
Then we start crawling:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/d614accbbc7a4f24aad0b2a87d0dd44c_1720253902706.png?token=ANB4BCOMHYPOIGPVS3IUTZTGRD7A4)
But we didn't get any results, because our computer doesn't have any DNS requests, so let's try typing `baidu.com` in the browser:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/f36b82c39ef94873b05cb5d60f7f74bd_1720253902706.png?token=ANB4BCPMRW34NFKNFIHUNUDGRD7A6)
It can be found that there is still no result. This is because the local address has been cached. If there is no mapping of this domain name in hosts, then search the local DNS resolver cache to see if there is such a URL mapping relationship. If so, return directly and complete DNS. So we should enter an unfamiliar address, such as `www.nihao.com`:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/19b410989a644b78b173ebc733391332_1720253902706.png?token=ANB4BCLEFM5Q7Y4YY5QVRXTGRD7BC)
At this time, two crawling results were obtained in an instant.
Here is the first result:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/a7ac837b9efe4fc4b69d0c8194525165_1720253909407.png?token=ANB4BCOP2HZNSZAT6ZYCJELGRD7BE)
We only look at the DNS protocol:
### Query:

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/c3aaf18aa86b4026aeb4721202df1c3c_1720253909407.png?token=ANB4BCIAGWVTSLKCYM3F32DGRD7BI)
- `Transaction ID: 0x4a57` indicates that the request is `0x4a57` whose ID is 16 bits;
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/cb373d0ab7a04c889bcce87521147006_1720253909407.png?token=ANB4BCL24EMK7UCWLX535V3GRD7BK)
- For 16-bit Flags: QR is 0, indicating that it is a query; opcode is 0, indicating that it is a standard query; AA is missing because it is only used for response (response); TC is 0, indicating that it is not truncated; RD 1, indicating that a recursive query is to be performed; RA is missing because it is only used for responses.
- For four-digit rcode (range 0-15):
>-0 No errors.
>- 1 Format error - The server cannot understand the requested message.
>- 2 Server failure (Server failure) - The request cannot be processed because of the server.
>- 3 Name Error - only meaningful to authoritative domain name resolution servers, indicating that the resolved domain name does not exist.
>- 4 Not Implemented - The name server does not support the query type.
>- 5 Refused (Refused) - The server refused to respond due to the set policy. For example, the server does not want to respond to some requesters, or the server does not want to perform certain operations (such as zone transfer).
>- 6-15 Reserved, not used yet.

- Questions and Answers:
![](https://raw.githubusercontent.com/Luyoung0001/picBed/main/626ecbeee6ee43ca8f02f3ce291dbf88_1720253909407.png?token=ANB4BCIH5NZUFLVNAN3KZTLGRD7BM)
>- Questions (Questions): Unsigned 16-bit integer indicates the number of question records in the message request segment.
>- Number of resource records (Answer RRs): An unsigned 16-bit integer indicates the number of answer records in the answer section of the message.
>- Number of Authorization Resource Records (Authority RRs): The unsigned 16-bit integer indicates the number of authorization records in the authorization section of the message.
>- Number of additional resource records (Additional RRs): An unsigned 16-bit integer indicating the number of additional records in the additional segment of the message.

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/9e2dbc116a324499b3eb9f5951018e46_1720253909407.png?token=ANB4BCO44AXED7E533KSNOLGRD7BO)
The crawl results show that there is 1 question record, 0 answer records, 0 authorization records, and 0 additional records.

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/4e1a1b0212e84a399ad1c7818fb2cf5a_1720253916289.png?token=ANB4BCJYDQUGL5H4ENPW6I3GRD7BS)

> - The query name is: time.apple.com, which is a FQDN
>- The type is A, indicating that the corresponding IP address is obtained from the domain name
>- class IN, the query class is 1, and the table obtains the IP address (The CLASS of a record is set to IN (for Internet) for common DNS records involving Internet hostnames, servers, or IP addresses. In addition, the classes Chaos (CH) and Hesiod (HS) exist.[34] Each class is an independent name space with potentially different delegations of DNS zones.)
	IN 1 the Internet
	CS 2 the CSNET class (Obsolete - used only for examples in some obsolete RFCs)
	CH 3 the CHAOS class
	HS 4 Hesiod [Dyer 87]
>- and word length ("time.apple.com") is 14, Lable Count is 3.

  ### Response
  We can see that this response belongs to query because they have the same Transaction ID (0x4a57).

![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/da07f9e337fe407e9820348f0f55941d_1720253916289.png?token=ANB4BCOIKZHLYW5WBDWAUUTGRD7BU)

It can also be seen that this time it has AA, rcode; and a status of Answer RRs = 3, indicating that three answers have been obtained.
Let's look at Answer again:
![在这里插入图片描述](https://raw.githubusercontent.com/Luyoung0001/picBed/main/598e7609263844f9ad78ed0a2d5b4cc2_1720253916289.png?token=ANB4BCIGUZCJQR5C52T36CLGRD7BY)
We got a total of three results:
- 17.253.116.253
- 17.253.114.253
- 17.253.114.125

According to the DNS protocol, we know that these three addresses should all point to "time.apple.com", let's check it!
![17.253.116.253](https://raw.githubusercontent.com/Luyoung0001/picBed/main/ac78a30068db45c8a9c2141b76f36fcd_1720253916289.png?token=ANB4BCKFJ2BSYUOLHI2RJ6DGRD7B6)

![17.253.114.253](https://raw.githubusercontent.com/Luyoung0001/picBed/main/104dd84ddc6146dca0e5d7301b6fa6da_1720253916289.png?token=ANB4BCPLZRGVK6NA7QM3WNTGRD7CC)


![17.253.114.125](https://raw.githubusercontent.com/Luyoung0001/picBed/main/942e77c01402481e8a16caf33a69ec74_1720253925484.png?token=ANB4BCOIAJAGGJLCDMJ3R73GRD7CG)



It can be seen that it is indeed so.

But: **Why does the DNS response message captured by Wireshark provide multiple "Answers"?**
**The answer is for load balancing.** DNS provides domain name resolution service. When accessing a site, it is first necessary to provide a DNS server to obtain the IP address corresponding to the domain name. During this process, the DNS server completes the mapping from the domain name to the IP address. Similarly, this mapping can also be One-to-many. That is, the DNS server can calculate and return many IP addresses according to the records and load balancing algorithm, and let the client choose a connection by itself.


*This text is over, thank you for reading.*


**Links for reference:**
*1:https://www.cnblogs.com/longlyseul/p/16251432.html
2:https://blog.csdn.net/qq_45946755/article/details/105301541
3:https://nordvpn.com/zh/ip-lookup/
4:https://www.mdnice.com/writing/cf64bb84044447bfa438f76c669f3ee2
5:http://jaminzhang.github.io/dns/DNS-Message-Format/
6:https://www.cnblogs.com/bonelee/p/12326832.html
7:https://help.aliyun.com/document_detail/102237.html
8:https://blog.csdn.net/JXH_123/article/details/26055215
9:https://www.freebuf.com/sectool/256745.html
10:https://stackoverflow.com/questions/60016607/what-is-in-in-dns-records*
















