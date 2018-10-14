---
title: Don't accidently mutate constants
date: "2018-10-14T05:22:45.212Z"
---

I had this one happen to me a couple of months ago and it took me quite a while to figure out.

The project I'm currently working on is a GraphQL API and makes a lot of API requests to downstream services. In order to be consistent in how we handle errors and logging when making these requests, I created a wrapper function around the `node-fetch` library we were using. This function is used for all requests (GET, POST, PUT etc.).

In the module that contains the function I defined a global object that contained default headers for all requests:
```javascript
const defaultHeaders = {
  GET: {
    'Content-Type': 'application/json'
  },
  POST: {
    'Content-Type': 'application/json'
  }
  ...
}
```

Then in the request wrapper, I took any passed in headers and merged them with the default headers like so:
```javascript
// `merge` is a lodash function
const headers = merge(defaultHeaders[method], customHeaders);
fetch(url, { method, options: { headers } });
```

This worked fine and everything was good. Except one day when a certain request started to fail (luckily we weren’t in production yet). The strange thing was, the request was only failing on the second time is was made. The first attempt worked fine.

After quite a lot of debugging, both on my local machine and on our staging environment (I thought it might have been an environment issue) I got desperate and started scouring the request wrapper and decided to read about the lodash [`merge`](https://lodash.com/docs/4.17.10#merge) function. It's there that I read these fateful words: `Note: This method mutates object.`

What was happening was there was another request using the wrapper function that passed in a `Content-Type` header of `application/www-form-url-encoded` and the merge was mutating the global `defaultHeaders` object! That meant that subsequent requests were using the wrong `Content-Type` header and the request was failing.

I fixed this quickly by passing an empty object as the first argument to `merge` (meaning merge everything into a new object) and everything went back to normal.

**Lesson learnt:** Don’t mutate global state. Or better yet, don’t have global state to begin with. I could have avoided this by defining the `defaultHeaders` in the scope of the function, or by doing something like [`Object.freeze`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) so I would get an error if I tried to mutate it.
