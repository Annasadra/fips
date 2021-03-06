---
fip: 2
title: Improvements to paging via API
status: Accepted
type: Functionality
author: Pawel Mastalerz <pawel@dapix.io>
created: 2020-04-06
updated: 2020-04-14
---

## Abstract
This FIP implements the following:
* Adds new API end points for fetching FIO Domains with support for paging
* Adds new API end points for fetching FIO Addresses with support for paging

## Motivation
There are currently deficiencies in paging for certain API calls:
* /get_fio_names has no paging at all. If an account has more FIO Domains or FIO Addresses than can be returned before table read time out, only a partial results are returned without warning to the user or ability to retrieve the rest.

## Specification
### Get FIO Domains
#### New end point: *get_fio_domains*
##### Request
|Parameter|Required|Format|Definition|
|---|---|---|---|
|fio_public_key|Yes|FIO Public Key|Valid FIO Public Key|
|limit|No|Positive Int|Number of domains to return. If omitted, all domains will be returned. Due to table read timeout, a value of less than 1,000 is recommended.|
|offset|No|Positive Int|First request from list to return. If omitted, 0 is assumed.|
###### Example
```
{
	"fio_public_key": "FIO8PRe4WRZJj5mkem6qVGKyvNFgPsNnjNN6kPhh6EaCpzCVin5Jj",
	"limit": 100,
	"offset": 0
}
```
##### Processing
* Request is validated per Exception handling
* Return *limit* FIO Domains starting at *offset* owned by *fio_public_key*.
##### Response
##### Exception handling
|Error condition|Trigger|Type|fields:name|fields:value|Error message|
|---|---|---|---|---|---|
|Invalid FIO Public Key|FIO Public Key format is not valid|400|"fio_public_key"|Value sent in, e.g. "notakey"|"Invalid FIO Public Key"|
|Invalid limit|limit format is not valid|400|"limit"|Value sent in, e.g. "-1"|"Invalid limit"|
|Invalid offset|offset format is not valid|400|"offset"|Value sent in, e.g. "-1"|"Invalid offset"|
|No FIO Domains|No FIO Domains were found for provided FIO Public Key or that key does not hash to a known account|404|||"No FIO Domains"|
##### Response
|Group|Parameter|Format|Definition|
|---|---|---|---|
|fio_domains|fio_domain|String|FIO Domain for requested public key.|
|fio_domains|expiration|String|Expiration date.|
|fio_domains|is_public|Int|0 - domain is not public, 1 - domain is public|
||more|Int|Number of remaining results|
###### Example
```
{
	"fio_domains": [
		{
			"fio_domain": "alice",
			"expiration": "2020-09-11T18:30:56",
			"is_public": 0
		}
	],
	"more": 0
}
```

### Get FIO Addresses
#### New end point: *get_fio_addresses*
##### Request
|Parameter|Required|Format|Definition|
|---|---|---|---|
|fio_public_key|Yes|FIO Public Key|Valid FIO Public Key|
|limit|No|Positive Int|Number of addresses to return. If omitted, all addresses will be returned. Due to table read timeout, a value of less than 1,000 is recommended.|
|offset|No|Positive Int|First request from list to return. If omitted, 0 is assumed.|
###### Example
```
{
	"fio_public_key": "FIO8PRe4WRZJj5mkem6qVGKyvNFgPsNnjNN6kPhh6EaCpzCVin5Jj",
	"limit": 100,
	"offset": 0
}
```
##### Processing
* Request is validated per Exception handling
* Return *limit* FIO Addresses starting at *offset* owned by *fio_public_key*.
##### Response
##### Exception handling
|Error condition|Trigger|Type|fields:name|fields:value|Error message|
|---|---|---|---|---|---|
|Invalid FIO Public Key|FIO Public Key format is not valid|400|"fio_public_key"|Value sent in, e.g. "notakey"|"Invalid FIO Public Key"|
|Invalid limit|limit format is not valid|400|"limit"|Value sent in, e.g. "-1"|"Invalid limit"|
|Invalid offset|offset format is not valid|400|"offset"|Value sent in, e.g. "-1"|"Invalid offset"|
|No FIO Addresses|No FIO Addresses were found for provided FIO Public Key or that key does not hash to a known account|404|||"No FIO Addresses"|
##### Response
|Group|Parameter|Format|Definition|
|---|---|---|---|
|fio_addresses|fio_address|String|FIO Address for requested public key.|
|fio_addresses|expiration|String|Expiration date.|
||more|Int|Number of remaining results|
###### Example
```
{
	"fio_addresses": [
		{
			"fio_address": "purse@alice",
			"expiration": "2020-09-11T18:30:56"
		}
	],
	"more": 0
}
```

## Rationale
Custom end points were put in place to make interaction with FIO Protocol easier for developers by hidding the complexity of EOSIO tools. Enhancing the functionality is done for the same reason. Advanced tools such as *get_table* remains unchanged.

## Implementation
* Adds new API end points for fetching FIO Domains with support for paging
  --modify chain_api_plugin to add new endpoint, modify chain_plugin.cpp and hpp to add new params and code for the fetching of domains.  dev test and resolve all issues (1 day)
* Adds new API end points for fetching FIO Addresses with support for paging
  --modify chain_api_plugin to add new endpoint, modify chain_plugin.cpp and hpp to add new params and code for the fetching of addresses.  dev test and resolve all issues (1 day)

### Abandoned changes
An attempt was made to add a new parameter to /get_obt_data, /get_pending_fio_requests and /get_sent_fio_requests to allow return of data based on create time. This was intended to because implementing wallets have expressed desire to cache the data locally and requested ability to query by providing a time stamp and only receiving requests since that time stamp.

However during implementation, this feature was abandoned for the following reason:
There does not seem to be a feasible way to do this. When initially prototyping the by time aspects of requests, we thought we could do a combined index (account and time) and then search on this combined index, but during implementation (and after putting the combined index into place in the data model) we discovered that the 128 bit indexes we are using do not perform reliably when other than exact match searches are performed. This leaves us with no other viable options to implement a by time search. Ideally what must be done is 1) make the range searches on 128 bit indexes to work in EOSIO, then query the range and filter the data returned (which still may not work at scale as we cannot guarantee the sequential ordering of the indexes generated, IE: the range returned could be enormous; 2) store this information off chain, in a history node, so that relational compound indexing can be used to search the data (this is the preferred solution).

## Backwards Compatibility
### New API end points
*/get_fio_domains* and */get_fio_address* are new end points and therefore do not impact existing users. */get_fio_names* end point remains unchanged.
