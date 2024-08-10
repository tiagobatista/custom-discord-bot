# dns-resolver

A common interview question for technical roles is: “What happens when you enter a URL into the address bar of your browser and hit enter?” Today we’re going to dig into part of the answer. The part that deals with how your system translates the host part of the URL to an IP address, i.e. from this: dns.google.com to 8.8.8.8 or 8.8.4.4.

In other words, domain name resolution; turning the hostname that your browser has extracted from the URL into an IP address.

To do this your browser contacts a DNS Resolver. The DNS Resolve may have the answer in it’s cache in which case it can return it immediately, if not then it will have to look it up. To look it up it contacts an authoritative name server. To do that it will first consult it’s cache for an authoritative name server and if it doesn’t have one it will contact a root name server to get one.

Step Zero

In this introductory step you’re going to set your environment up ready to begin developing and testing your solution.

I’ll leave you to choose your target platform, setup your editor and programming language of choice. I’d encourage you to pick a tech stack that you’re comfortable doing network programming with.

Step 1

In this step your goal is to build a message to be sent to a name server.

A DNS message has:

A header.
A questions section.
An answer section.
An authority section.
An additional section.
The header is defined in RFC 1035 Section 4.1.1

A question section has three fields: the name that we’re looking up, a record type and the class. The name is a byte string, the other two fields are 1b-bit integers.

The question is defined in the same RFC, Section 4.1.2.

The name field is an encoded version of the host name string, so if we’re looking up dns.google.com it will be encoded as the following (bytes) 3dns6google3com0.

For this initial query your message should have empty answer, authority and additional sections.

To build a query we will need to assemble all of this into a byte string ready send over the network.

All the integers are sent over the network big endian byte order.

The message that you’re going to build will look like this:

Two bytes of the id - you can generate a random number for this. I’ve used 22 in the example below.
Two bytes for the flags - I’m lumping several fields together here for simplicity, what matters is that we set ‘recursion desired’ bit to 1 because we’re asking a DNS resolver first. You can see which bit that is from the RFC Section 4.1.1.
Two bytes each for the number of questions, answer resource records, authority resource records and additional resource records. The last three of these will be 0 and the number of questions will be 1.
The encoded byte string for the question.
Two bytes for the query type: 1 this time (it is defined in the RFC Section 3.2.2)
Two bytes for the query class: 1 this time (it is defined in the RFC Section 3.2.4)
Which would give us the following (hex) 00160100000100000000000003646e7306676f6f676c6503636f6d0000010001

Once you have the data structures to represent the header, question and message and the ability to convert it to a bytes string ready proceed to Step 2.

Step 2

In this step your goal is to send the request to a name server and dump the response out.

To do that we need to create a socket to send UDP and send out message to a name server, then read back the response. As we’re looking for the IP address of Google’s DNS server we can reasonably assume that Google’s DNS server can tell us what it is (we’re skipping to asking the name server for the domain to simplify this step).

So create your socket and send your message to Google’s DNS server using the IP address: 8.8.8.8 and port 53 (the DNS service port).

Read back the response, which will look something like this (hex): 00168080000100020000000003646e7306676f6f676c6503636f6d0000010001c00c0001000100000214000408080808c00c0001000100000214000408080404

For now just work checking that the first two bytes match the id you sent.

Step 3

In this step your goal is to parse the response and extract the answers from it. The response has the same structure as the request we sent but hopefully you have received some data in the answers section. The format of the answers is defined in Section 4.1.3 of the RFC.

You’ll need to parse the header, then parse the question, answer, authorities and additionals sections. You should also check that the QR bit specified in 4.1.1. is 1 to indicate a response.

When parsing the last three sections you’ll need to be aware that DNS supports some compression, you’ll find details in the RFC Section 4.1.4. Don’t forget you’ll need to re-join the sections of the name with .

If you decode the message you get back from the name server you should now have the same id you send, one question, two answers and zero authorities and additionals. The answers should include the name we queried dns.google.com, a type and class of 1, a TTL greater than zero, and the IP addresses 8.8.8.8 and 8.8.4.4.

Step 4

In this step your goal is to be able to query one of the root name servers so you can resolve any host and domain name, not just Google’s.

Firstly you’ll need to set the bit for recursion to zero now instead of one. In other words the flags will now be zero.

Next you’ll need to parse the response and take note of the type. Type 1 is an A record - which is going to give us the IP Address. Type 2 is an NS record - that is an authoritative server (remember these values are defined in the RFC Section 3.2.2).

If the response is an NS record we will find a list of authoritative name servers in the authorities section and in the additionals section we may find a record with it’s IP address. If so you can use that, if not you will need to query a name server to resolve the IP address of the name server you’ve been referred to.

So to complete this step, change the name server you query to be on of the root name servers - you can find a list of them on the wikipedia page or just use 198.41.0.4. Then implement the code to make a series of calls to resolve the names until you get an answer.

That might look something like this if you use 198.41.0.4 to resolve dns.google.com:

Querying 198.41.0.4 for dns.google.com
Querying 192.12.94.30 for dns.google.com
Querying 216.239.34.10 for dns.google.com
'8.8.8.8'
Congratulations you’ve built a very basic DNS Resolver.

As a bonus step, consider extending it to handle CNAME records.
