# kubernetes

*kubernetes* enables reading zone data from a kubernetes cluster.
It implements the [spec](https://github.com/kubernetes/dns/blob/master/docs/specification.md)
defined for kubernetes DNS-Based service discovery:

Service `A` records are constructed as "myservice.mynamespace.svc.coredns.local" where:

* "myservice" is the name of the k8s service
* "mynamespace" is the k8s namespace for the service, and
* "svc" indicates this is a service
* "coredns.local" is the zone

Pod `A` records are constructed as "1-2-3-4.mynamespace.pod.coredns.local" where:

* "1-2-3-4" is derived from the ip address of the pod (1.2.3.4 in this example)
* "mynamespace" is the k8s namespace for the service, and
* "pod" indicates this is a pod
* "coredns.local" is the zone

Endpoint `A` records are constructed as "epname.myservice.mynamespace.svc.coredns.local" where:

* "epname" is the hostname (or name constructed from IP) of the endpoint
* "myservice" is the name of the k8s service that the endpoint serves
* "mynamespace" is the k8s namespace for the service, and
* "svc" indicates this is a service
* "coredns.local" is the zone

Also supported are PTR and SRV records for services/endpoints.

## Syntax

This is an example kubernetes configuration block, with all options described:

```
# kubernetes <zone> [<zone>] ...
#
# Use kubernetes middleware for domain "coredns.local"
# Reverse domain zones can be defined here (e.g. 0.0.10.in-addr.arpa),
# or instead with the "cidrs" option.
#
kubernetes coredns.local {

	# resyncperiod <period>
	#
	# Kubernetes data API resync period. Default is 5m
	# Example values: 60s, 5m, 1h
	#
	resyncperiod 5m

	# endpoint <url>
	#
	# Use url for a remote k8s API endpoint.  If omitted, it will connect to
	# k8s in-cluster using the cluster service account.
	#
	endpoint https://k8s-endpoint:8080

	# tls <cert-filename> <key-filename> <cacert-filename>
	#
	# The tls cert, key and the CA cert filenanames for remote k8s connection.
	# This option is ignored if connecting in-cluster (i.e. endpoint is not
	# specified).
	#
	tls cert key cacert

	# namespaces <namespace> [<namespace>] ...
	#
	# Only expose the k8s namespaces listed.  If this option is omitted
	# all namespaces are exposed
	#
	namespaces demo

	# lables <expression> [,<expression>] ...
	#
	# Only expose the records for kubernetes objects
	# that match this label selector. The label
	# selector syntax is described in the kubernetes
	# API documentation: http://kubernetes.io/docs/user-guide/labels/
	# Example selector below only exposes objects tagged as
	# "application=nginx" in the staging or qa environments.
	#
	labels environment in (staging, qa),application=nginx

	# pods <disabled|insecure|verified>
	#
	# Set the mode of responding to pod A record requests.
	# e.g 1-2-3-4.ns.pod.zone.  This option is provided to allow use of
	# SSL certs when connecting directly to pods.
	# Valid values: disabled, verified, insecure
	#  disabled: Do not process pod requests, always returning NXDOMAIN
	#  insecure: Always return an A record with IP from request (without
	#            checking k8s).  This option is is vulnerable to abuse if
	#            used maliciously in conjuction with wildcard SSL certs.
	#  verified: Return an A record if there exists a pod in same
	#            namespace with matching IP.  This option requires
	#            substantially more memory than in insecure mode, since it
	#            will maintain a watch on all pods.
	# Default value is "disabled".
	#
	pods disabled

	# cidrs <cidr> [<cidr>] ...
	#
	# Expose cidr ranges to reverse lookups.  Include any number of space
	# delimited cidrs, and or multiple cidrs options on separate lines.
	# kubernetes middleware will respond to PTR requests for ip addresses
	# that fall within these ranges.
	#
	cidrs 10.0.0.0/24 10.0.10.0/25

	# upstream <address> [<address>] ...
	#
	# Defines upstream resolvers used for resolving services that point to
	# external hosts (External Services).  <address> can be an ip, an ip:port, or
	# a path to a file structured like resolv.conf.
	upstream 12.34.56.78:53
	
	# federation <federation-name> <federation-domain>
	#
	# Defines federation membership.  One line for each federation membership.
	# Each line consists of the name of the federation, and the domain.
	federation myfed foo.example.com
	
	# autopath [NDOTS [RESPONSE [RESOLV-CONF]]
	#
	# Enables server side search path lookups for pods.  When enabled, coredns
	# will identify search path queries from pods and perform the remaining
	# lookups in the path on the pod's behalf.  The search path used mimics the
	# resolv.conf search path deployed to pods. E.g.
	#
	#  search ns1.svc.cluster.local svc.cluster.local cluster.local foo.com
	#
	# If no domains in the path produce an answer, a lookup on the bare question
	# will be attempted.	
	#
	# A successful response will contain a question section with the original
	# question, and an answer section containing the record for the question that
	# actually had an answer.  This means that the question and answer will not
	# match. For example:
	#
	#    # host -v -t a google.com
	#    Trying "google.com.default.svc.cluster.local"
	#    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50957
	#    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
	#
	#    ;; QUESTION SECTION:
	#    ;google.com.default.svc.cluster.local. IN A
	#
	#    ;; ANSWER SECTION:
	#    google.com.		175	IN	A	216.58.194.206
	#
	#
	# NDOTS (default: 0) This provides an adjustable threshold to
	# prevent server side lookups from triggering. If the number of dots before
	# the first search domain is less than this number, then the search path will
	# not executed on the server side.
	#
	# RESPONSE (default: SERVFAIL) RESPONSE can be either NXDOMAIN, SERVFAIL or
	# NOERROR. This option causes coredns to return the given response instead of
	# NXDOMAIN when the all searches in the path produce no results. Setting this
	# to SERVFAIL or NOERROR should prevent the client from fruitlessly continuing
	# the client side searches in the path after the server already checked them.
	#
	# RESOLV-CONF (default: /etc/resolv.conf) If specified, coredns uses this
	# file to get the host's search domains. CoreDNS performs a lookup on these
	# domains if the in-cluster search domains in the path fail to produce an
	# answer. If not specified, the values will be read from the local resolv.conf
	# file (i.e the resolv.conf file in the pod containing coredns).
	#
	# Enabling autopath causes coredns to use more memory since it needs to
	# maintain a watch on all pods. If autopath and "pods verified" mode are
	# both enabled, they will share the same watch. I.e. enabling both options
	# should have an equivalent memory impact of just one.
	autopath 0 SERVFAIL /etc/resolv.conf

	# fallthrough
	#
	# If a query for a record in the cluster zone results in NXDOMAIN,
	# normally that is what the response will be. However, if you specify
	# this option, the query will instead be passed on down the middleware
	# chain, which can include another middleware to handle the query.
	fallthrough
}

```

## Wildcards

Some query labels accept a wildcard value to match any value.
If a label is a valid wildcard (\*, or the word "any"), then that label will match
all values.  The labels that accept wildcards are:
* _service_ in an `A` record request: _service_.namespace.svc.zone.
   * e.g. `*.ns.svc.myzone.local`
* _namespace_ in an `A` record request: service._namespace_.svc.zone.
   * e.g. `nginx.*.svc.myzone.local`
* _port and/or protocol_ in an `SRV` request: __port_.__protocol_.service.namespace.svc.zone.
   * e.g. `_http.*.service.ns.svc.`
* multiple wild cards are allowed in a single query.
   * e.g. `A` Request `*.*.svc.zone.` or `SRV` request `*.*.*.*.svc.zone.`

## Deployment in Kubernetes

See the [deployment](https://github.com/coredns/deployment) repository for details on how
to deploy CoreDNS in Kubernetes.
