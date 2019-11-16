---
layout: post
title: How to use single auth method for every request in collection
tags: Postman
---

If you are just like me tired of messing with authorization header for every request in collection then you're welcome for a very helpful solution.

## Solution 

All you have to do is:
- Click edit on a collection;
- Navigate to 'authorization' tab;
- Choose and setup an authorization type, that will be used for every request in this collection;

![postman-collection-authorization-tab](/images/post/postman-collection-authorization-tab.png){: .center-image }

Once it's done, you need to ensure that an authorization type of every request is set to `Inherit auth from parent`:

![postman-collection-authorization-tab](/images/post/postman-request-authorization-tab.png){: .center-image }

If everything has been done properly, you won't have that problem again.
