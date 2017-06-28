# Bug Overview:
Firefox does not correctly follow redirects when the connections is http2 and from a domain resolved
to multiple ips to a another domain which is resolved to a single(?) ip, which is one of the
multiple of ips of the first domain, on an http2 connection.

## Example: 
Let say `common.example.com` is resolved to two ips `172.129.0.5` and `172.129.0.102` 
and `specific1.example.com` is resolved to `172.129.0.5` (the first ip) 
and `specific2.example.com` to `172.129.0.102` (the second ip)

All servers support http2 and are well configured. 

When firefox makes request to `https://common.example.com/file` it makes it to `172.129.0.5`, which 
responds with 302 redirect to `https://specific2.example.com/file`. But instead of making new 
connection to `172.129.0.102` it reuses the one to `172.129.0.5`

This repo makes exactly this example with 2 nginxs and a bind9 for dns. 
The two nginxs always redirect common.example.com to the other nginx, respond with 200 for their
"speicific" domain and with 404 for all other domains 

## How to reproduce:

You will require docker and docker-compose(fairly recent ones), openssl and bash :). 
NOTICE: The following has only been tested under fedora 25 some options might have to be tweaked on
your machine.

1. clone this repo
2. run ./generate_ca.sh to generate a new CA certificate and key
3. run ./generate_certs.sh to generate new certificates to be used for ssl
4. run docker-compose up
5. set 172.129.0.128 as your dns (this is a docker container)
6. open firefox and add ca.crt as a trusted certificate authority
7. open https://common.example.com/file - see it change to https://specific[12].example.com/file and
   a 404. You can also open the developer tools and see which servers the requests are made to.

## Specifics found so far: 

- This only happens when the connection is http2, with simple http or https(HTTP/1.1) connections
  the issue is not reproducible.
- It is required that the first domain is resolved to multiple ips and one of those ip is the ip of
  the domain to which it is redirected
- The certificates of the domain have to be from a trusted CA
- The bug is reproducible both under firefox 54 and 55b3, no other versions are tested at this time
- With wireshark I can confirm that an http2 connection is established, the first get with the
  redirect is received and then a dns request for the redirected domain is made. After the dns
  response is received the http2 connection is reused even though the server it's made to is not the
  one in the dns response. Sometimes a second http2 connection, to the correct server, is made and
  closed, without any requests made, around the same time. This points to some kind of a race
  condition.
- Because firefox caches a lot of things and reuses connections it's advisable to always restart
  firefox between tries to reproduce the issue
- As an example of the above if first we open https://specific1.example.com and
  https://specific2.example.com and then https://common.example.com, the issue is not reproduced
