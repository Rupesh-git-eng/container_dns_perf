## How ndots (ndots=5) works within container?

### Sample of /etc/resolv.conf
~~~
sh-4.4$ cat /etc/resolv.conf 
search git.svc.cluster.local svc.cluster.local cluster.local git_host.redhat.com
nameserver 172.30.0.10
options ndots:5
~~~
 
### Let's looks at how internal service domain name to ip is discovered. 
#### (Example -1 ) Query to the domain name.
~~~
dig httpd.git.svc.cluster.local +showsearch +search | grep -A1 'QUESTION'

;; QUESTION SECTION:
;httpd.git.svc.cluster.local.rupesh.svc.cluster.local. IN A
--
;; QUESTION SECTION:
;httpd.git.svc.cluster.local.svc.cluster.local. IN A
--
;; QUESTION SECTION:
;httpd.git.svc.cluster.local.cluster.local. IN A
--
;; QUESTION SECTION:
;httpd.git.svc.cluster.local.example.redhat.com. IN A
--
;; QUESTION SECTION:
;httpd.git.svc.cluster.local.	IN	A

~~~
#### Impact 
> - As the domain is having less than 5 dots i.e four (httpd-git.svc.cluster.local), the search is executed in sequence.
> - After an attempts with all search order final query is made which is expected to be resolved here.
> - time it takes in first four query adds delay here.
> - The **fourth query** is very expensive as it goes out of coredns to upstream dns, return to coredns and then to the client.

#### (Example -2) Query to the fqdn.
~~~
$ dig httpd.git.svc.cluster.local. +showsearch +search | grep -A1 'QUESTION'

;; QUESTION SECTION:
;httpd.git.svc.cluster.local.	IN	A
~~~
#### Impact 
> - Remember **dot** at the end i.e local.
> - Calling fqdn never goes through search order as fqdn doesn't need that. 
> - Dot at the end makes it fqdn.
> - Significant performance improvement as compare to Exe-1.

## (Example -3)  Can't use dot (.), go though search order but make it faster than Example -1, close to Example Exe -2? 

~~~
dig httpd.git +showsearch +search | grep -A1 'QUESTION'

;; QUESTION SECTION:
;httpd.git.rupesh.svc.cluster.local. IN	A
--
;; QUESTION SECTION:
;httpd.git.svc.cluster.local.	IN	A
~~~
#### Impact 
> - Calling short names calls more appropriate queries with search domain.
> - Lesser iteractive queries.
> - Simple configuration
> - Much faster resolution.
