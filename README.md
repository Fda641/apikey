# Onshape API Key Sample Apps

Simple Node.js and Python apps to demonstrate API key usage

---

### Getting Started

Please see the [node](https://github.com/onshape/apikey/tree/master/node) and
[python](https://github.com/onshape/apikey/tree/master/python) folders for
instructions on working with each of the applications.

### Why API Keys?

We've moved over to using API keys for authenticating requests instead of using
cookies for several reasons.

1. Security: Each request is signed with unique headers so that we can be sure it's
coming from the right place.
2. OAuth: The API key system we're now using for HTTP requests is the same process
developers follow when building full-blown OAuth applications; there's no longer a disconnect
between the two.

### Questions & Concerns

If you need information or have a question unanswered in this documentation,
feel free to chat with us by sending an email to
[api-support@onshape.com](mailto:api-support@onshape.com) or by checking out
the [forums](https://forum.onshape.com/).

### Working w/ API Keys

Read the following and you'll be up and running with using API keys in your
application:

##### Instructions

1. Get the Developer role for your Onshape account by contacting us at
[api-support@onshape.com](mailto:api-support@onshape.com). From the Developer Portal,
you can create and manage your API keys; optionally, you can use HTTP requests by
following the process below:

2. Create a key pair by sending a `POST` request to `/api/users/apikeys`; you
can optionally specify a `scopeNames` array as part of the JSON body. See
[below](#scopes) for information on what scopes are available. If you omit the
`scopeNames` array, your key will be generated with all scopes you have access to.

3. The server will send back a JSON object with the structure:
```json
{
    "state": 0,
    "scopeNames": [
        "scope1",
        "scope2",
        "etc..."
    ],
    "accessKey": "<accessKey>",
    "userId": "<userId>",
    "secretKey": "<secretKey>",
    "name": null,
    "id": "<accessKey>",
    "href": "https://<server>/api/users/apikeys/<accessKey>"
}
```
The two most important pieces of the response are of course the `accessKey` and
`secretKey`; keep them stored safely!

4. Now that you have a key pair, see [below](#generating-a-request-signature) for
information on signing your requests to use our API.

##### Scopes

There are several scopes available for API keys (equivalent to existing OAuth scopes),
which can be passed in as a part of your `POST` request to `/api/users/apikeys`
via the JSON body, in the `scopeNames` array:

* "OAuth2Read" - Read non-personal information (documents, parts, etc.)
* "OAuth2ReadPII" - Read personal information (name, email, etc.)
* "OAuth2Write" - Create and edit documents / etc.
* "OAuth2Delete" - Delete documents / etc.
* "OAuth2Purchase" - Authorize purchases from account

##### Endpoints

Below are the relevant endpoints for working with API keys:

* `POST /api/users/apikeys`: Create a new API key pair
* `GET /api/users/apikeys`: Get all API key pairs for current user
* `GET /api/users/apikeys/:id`: Get a specific API key pair by ID
* `POST /api/users/apikeys/:id`: Toggle key pair state between active (0) and inactive (1)
    * Body should be of form: `{ "accessKey": "key", "state": 0 or 1 }`
* `DELETE /api/users/apikeys/:id`: Delete a key pair

##### Generating A Request Signature

To ensure that a request is coming from you, we have a process for signing
requests that you must follow for API calls to work. Everything is done via HTTP
headers that you'll need to set:

1. *Date*: A standard date header giving the time of the request; must be
accurate within **5 minutes** of request. Example: `Mon, 11 Apr 2016 20:08:56 GMT`
2. *On-Nonce*: A string that satisfies the following requirements (see the code for one possible way to generate it):
    * At least 16 characters
    * Alphanumeric
    * Unique for each request
3. *Authorization*: This is where the API keys come into play. You'll sign the request by implementing this algorithm:
    * **Input**: Method, URL, On-Nonce, Date, Content-Type, AccessKey, SecretKey
    * **Output**: String of the form: `On <AccessKey>:HmacSHA256:<Signature>`
    * **Steps to generate the signature portion**:
        1. Parse the URL and get the following:
            1. The path, e.g. `/api/documents` (no query params!)
            2. The query string, e.g. `a=1&b=2`
                * NOTE: If no query paramaters are present, use an empty string
        2. Create a string by appending the following information in order. Each
        field should be separated by a newline (`\n`) character, and the string
        must be converted to lowercase:
            1. HTTP method
            2. On-Nonce header value
            3. Date header value
            4. Content-Type header value
            5. URL pathname
            6. URL query string
        3. Using SHA-256, generate an [HMAC digest](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code),
        using the API secret key first and then the above string, then encode it in Base64.
        4. Create the `On <AccessKey>:HmacSHA256:<Signature>` string and use that in the Authorization header in your request.

Below is an example function to generate the authorization header, using
Node.js's standard `crypto` and `url` libraries:

```js
// ...at top of file
var u = require('url');
var crypto = require('crypto');

/**
* Generates the "Authorization" HTTP header for using the Onshape API
*
* @param {string} method - Request method; GET, POST, etc.
* @param {string} url - The full request URL
* @param {string} nonce - 25-character nonce (generated by you)
* @param {string} authDate - UTC-formatted date string (generated by you)
* @param {string} contentType - Value of the "Content-Type" header; generally "application/json"
* @param {string} accessKey - API access key
* @param {string} secretKey - API secret key
*
* @return {string} Value for the "Authorization" header
*/
function createSignature(method, url, nonce, authDate, contentType, accessKey, secretKey) {
    var urlObj = u.parse(url);
    var urlPath = urlObj.pathname;
    var urlQuery = urlObj.query ? urlObj.query : ''; // if no query, use empty string

    var str = (method + '\n' + nonce + '\n' + authDate + '\n' + contentType + '\n' +
        urlPath + '\n' + urlQuery + '\n').toLowerCase();

    var hmac = crypto.createHmac('sha256', secretKey)
        .update(str)
        .digest('base64');

    var signature = 'On ' + accessKey + ':HmacSHA256:' + hmac;
    return signature;
}
```
