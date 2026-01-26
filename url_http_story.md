# The story of URLs & HTTP

## The beginning: file transfer

It is common to want to transfer a file from one computer from another. The simple problem is:

- there are two computers, client C, and server S
- the client C would like to get the file from the server S. C knows the path to the file, ex: '/Users/atotic/story.txt'

The client and the server need to talk. The way the computers can talk to each other is by opening a connection and sending messages to each other over that connection. Kind of like a human chat, but between the machines.

How do you get a file from client to server? 

1. the server gets ready: it runs an app that listens for connections on a port. Port is just a number that each server gets when it starts listening.

2. Second, the client gets ready: it has the name of the server (ex: restcountries.com), the port (ex: 80), and the path to the file (ex: '/Users/atotic/robots.txt')

3. Third, the client opens the connection to the server restcountries.com, port 80. When connection is ready, it sends a command to server (ex: GIVE ME FILE /Users/atotic/robots.txt). Server says "SURE, here is the file: <contents of robots.txt..............>"

Since client and server can't talk English to each other, they have to agree on the computer language they both understand: what commands and responses are allowed, and what do they mean. We call this _protocol_

Long time ago, people designed multiple protocols for file transfer: FTP (file transfer protocol), Gopher, HTTP (hypertext transfer protocol). Nowdays, it is almost all HTTP, HTTP won.

## URLs

In the ancient times, when we still had many protocols, telling someone how to get a file was complex. You'd say something like: "use ftp, cd to this directory, and get this file".

URLs (Universal Resource Locators) were designed to simplify these instructions on how to get a file. A URL would contain all the information you needed to get it: 
&lt;protocol>://&lt;hostname>:&lt;port>/&lt;file_path>

http://restcountries.com/robots.txt

This tells you to use protocol http, go to server restcountries.com, and get file called robots.txt

And that is all URLs were in the beginning.

Almost all the other protocols have become obsolete, https has become the world's default.

## HTTP

URLs were invented by Tim Berners Lee when he invented World Wide Web. 
At the same time Tim Berners Lee invented URLs, he also invented the 
HTML (hypertext markup language used to make web pages), and 
HTTP (hypertext transfer protocol, used for file transfer.

The HTTP was the simplest possible protocol. It had one command server understood:

```GET <path to the file>```

Server would respond by sending the file, or closing the connection in error if file was not found.

URLs, HTML, and HTTP were the standards that made World Wide Web possible. 
The browsers were clients that knew how to display HTML pages, and follow URL links.
And the servers were just servers, they'd just send files in response to GET requests.

## Complications: Forms 

Plain hypertext pages were ok, but how can we do something more interesting, for example
type some text in a seach box, have server do a search, and return the result? Or what about
more complicated forms, with names, addresses, etc.

The initial solution was to extend the URL with **query parameters**. Query parameters were put
after the file path, with '?' used to separate file path from query params.

The URL now looked like this:

&lt;protocol>://&lt;hostname>:&lt;port>/&lt;file_path>?&lt;query_params>

And the query params could have multiple key value pairs, separated by '&'. For example, { search: Sports, language: German } would be encoded as:

http://restcountries.com/?search=Sports&language=German

The client would sent the whole path+params to the server, the server would see the '?' and go "I know this is not just a file request, I might need to do a search for Sports instead, and return the search results instead of a file"

## Complications: HTTP APIs

People liked HTTP a lot because it was so simple, but wanted to do more than just serve files. They wanted to expose APIs to the network. You could thing of an HTTP request with a query string as a function call:

http://https://restcountries.com/v3.1/capital?cityName=tallinn

looks a little bit like a call to:

getCountryByCapital(cityName)

People would define their APIs a set of URLs, where each URL did something (not necessarily returning a file).

To accomodate these APIs, lots of new things got added to HTTP.

#### HTTP headers

Client wanted some more information from the server. Client wanted
to know how big the file was, so it could display progress bar.
Client wanted to know what type of file it was, HTML, or plain text,
or an image, so it could display the file correctly. Also, cookies if user was logged in!

To accomodate this new data client and server wanted to communicate, we came up with **headers**. The headers were sent at the beginning of each request/response. Over the network, the communication looked somethings like this:

```
Client request:

GET <path to the file>
User-Agent: Mosaic
Referer: http://restcountries.com/
```
```
Server response:

Status: 200
Content-Length: 33108
Content-Type: text/html
Date: Thu, 22 Jan 2026 07:56:03 GMT

file contents sent here after the headers
```

With headers, client and server could now pass a lot more information to each other.

Status was also an important improvement. Before status, any error would just close the connection, and client would have no idea what happened. With status, server could tell client that there was an error, what type of error it was, so that client would handle it properly (authentication required, retry, redirect, etc...)

#### new HTTP commands: POST, etc

People wanted to upload files with forms. Sticking a file into the URL
query parameters did not work well, because URLs were limited in size,
no one wanted to work with megabyte long URLs.

So the protocol was extended with a new word, POST. With POST request,
client would send form data not as part of file path, but after 
the headers. We call this part of request **the payload**.

You will encounter POST when working with APIs.

To make APIs a little easier to write, other commands (called methods in HTTP standard), were also added: PUT/DELETE/HEAD. You'll rarely encounter these, but you should know they are there if APIs specify them.

## Complications: HTTP security

Initial HTTP protocol was not encrypted, so anyone could see the data
going between client and the server. No privacy was bad.
To fix this, HTTP was extended to
HTTPS, which does the same thing as HTTP, but encrypts all communication
between client and server. That is why most of the URLs you see today start with `https://`, and not `http://`

## References
[HTTP standard](https://datatracker.ietf.org/doc/html/rfc7231)

[URL standard](https://url.spec.whatwg.org/)

[curl](https://curl.se/), a great command tool that lets you do everything with HTTP from the command line.


