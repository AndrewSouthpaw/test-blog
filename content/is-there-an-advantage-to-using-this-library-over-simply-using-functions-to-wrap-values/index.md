---
title: "Is there an advantage to using this library over simply using functions to wrap values?"
description: "This seems to provide the same functionality and avoids importing a new library and syntax. Am I missing something?"
date: "2017-12-04T19:21:48.817Z"
categories: []
published: true
canonical_link: https://medium.com/@andrewsouthpaw/is-there-an-advantage-to-using-this-library-over-simply-using-functions-to-wrap-values-e64670c0eebf
redirect_from:
  - /is-there-an-advantage-to-using-this-library-over-simply-using-functions-to-wrap-values-e64670c0eebf
---

Is there an advantage to using this library over simply using functions to wrap values?

```
const user = () => User.create({ role: ‘member’ })
it(...)
```

This seems to provide the same functionality and avoids importing a new library and syntax. Am I missing something?
