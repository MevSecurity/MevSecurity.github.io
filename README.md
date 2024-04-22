# MevSec Blog

## Create a DRAFT post

To create a **draft** you only need to add a simple parameter in the yaml config the `draft` parameter like below:

```yaml
title: "FLASHBOT SUPER ARTICLE"
date: 1970-01-01
url: /my draft-post/
draft: true
```

This will create the post in the DRAFT mode so this one will be not visible by the others people.

```bash
hugo server --buildDrafts
```

Then you can use this command to run the server **LOCALLY** and perform some checks before pushing to the Github.

The server will run in localhost and you can perform the check if the article is correct like this:

![CleanShot 2023-10-15 at 19 47 28](https://github.com/MevSecurity/MevSecurity.github.io/assets/23560242/b18a8e3a-f6e1-40a7-a62a-e582303fe743)

Then you can check the result on your browser cf here:

![CleanShot 2023-10-15 at 19 49 59](https://github.com/MevSecurity/MevSecurity.github.io/assets/23560242/deb1cb6d-34af-428e-bc75-dee614ffbc0c)

To ensure this is working we can check on the **Posts** on the up corner right: ![CleanShot 2023-10-15 at 19 50 18](https://github.com/MevSecurity/MevSecurity.github.io/assets/23560242/da72a45b-368f-4182-8fa2-d16a73654c74)

## Tips

You need to add the `<!--more-->`

```markdown
---
weight: 1
title: "Taking down Wormhole's CCQ service through REST API"
date: 2023-10-24
tags: ["Wormhole", "CCQ", "DoS", "immunefi"]
draft: true
author: "fadam"
authorLink: ""
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
```

## Markdown Templates

```markdown
## 4. Socials.

| Discord (Join us!)            | Github                            | Twitter                          |
| ----------------------------- | --------------------------------- | -------------------------------- |
| https://discord.gg/54Q9pnpQcV | https://github.com/Ethnical/Swek3 | https://twitter.com/EthnicalInfo |
```
