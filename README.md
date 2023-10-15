# MEVSEC BLOG

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
