<!-- omit in toc -->
# API Documentation

Documentation for the Discuit API.

Base url: `https://discuit.net/api/`

## Table of contents

- [Table of contents](#table-of-contents)
- [Authentication](#authentication)
- [Errors](#errors)
- [Endpoints](#endpoints)
  - [`/api/_initial [GET]`](#api_initial-get)
  - [`/api/_login [POST]`](#api_login-post)
  - [`/api/_signup [POST]`](#api_signup-post)
    - [Possible errors](#possible-errors)
  - [`/api/_user [GET]`](#api_user-get)
  - [`/api/users/{username} [GET]`](#apiusersusername-get)
  - [`/api/users/{username}/feed [GET]`](#apiusersusernamefeed-get)
    - [Query parameters](#query-parameters)
    - [Possible errors](#possible-errors-1)
  - [`/api/posts [GET]`](#apiposts-get)
    - [Query parameters](#query-parameters-1)
    - [Possible errors](#possible-errors-2)
  - [`/api/posts [POST]`](#apiposts-post)
  - [`/api/posts/{postId} [GET]`](#apipostspostid-get)
  - [`/api/posts/{postId} [PUT]`](#apipostspostid-put)
    - [Possible errors](#possible-errors-3)
  - [`/api/posts/{postId} [DELETE]`](#apipostspostid-delete)
  - [`/api/_postVote [POST]`](#api_postvote-post)
  - [`/api/posts/{postID}/comments [GET]`](#apipostspostidcomments-get)


## Authentication

The Discuit API uses [HTTP
Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) for
authentication.

The first call to any GET API endpoint (that does not require authentication)
returns the cookies that are needed for authenticated API requests. Namely, the
`SID` (session id) cookie and the `csrftoken` cookie. However, it is recommended
to use the `/api/_initial` endpoint for this.

The value in the `csrftoken` cookie needs to be passed in an `X-CSRF-TOKEN` HTTP
header for all except GET requests to mitigate Cross Site Request Forgeries.


## Errors

Most API errors return a JSON object of the following type, along with the
appropriate HTTP status code:

```
{
    "status", // HTTP status code
    "code" // Custom error code
}
```

Henceforth this object shall be referred to as APIError.

Some API endpoints return only a simple HTTP status code with an empty HTTP body.

A 500 HTTP status may be returned by any endpoint, if anything goes wrong on the
server. 

For any request that requires a JSON HTTP body, if the JSON object does
not match the expected object, a 400 APIError with `"invalid_json"` code will
be returned.

If any endpoint has any rate-limiting, and if rate-limits are exceeded, a 429
HTTP status code is returned.

And these general errors are not mentioned again in the tables of errors on the
rest of this page.

## Endpoints

### `/api/_initial [GET]`

JSON response:

```
{
    // An array of reasons to report a post or a comment.
    "reportReasons",

    // If authenticated, the user object of the authenticated user. Otherwise,
    // null.
    "user",

    // If authenticated, the list of communities that the user is subscribed to.
    // Otherwise, the list of default communities is returned.
    "communities",

    // Discuit total user count.
    "noUsers",

    // The list of communities that the authenticated user is banned from. If
    // not authenticated, this value is null.
    "bannedFrom"
}
```

### `/api/_login [POST]`

Requests must have the following JSON body:

```
{
    "username",
    "password"
}
```

If the username and password matches, the user object is returned. Otherwise, a
401 (unauthorized) error is returned.

If the user account is suspended, a 403 (forbidden) status is returned with an APIError code `"account_suspended"`.

If the user is authenticated and the request URL has the query parameter
`action=logout`, user is logged out from the session. No request body is
required for logout. In other words, to logout a user, send a POST request to
`/api/_login?action=logout`.

### `/api/_signup [POST]`

Requests must have the following JSON body:

```
{
    "username",
    "password",

    // reCAPTCHA v2 token
    "captchaToken",
}
```

If everything goes well the JSON object of the newly created account is
returned.

Upon successful signup, the user is logged in automatically in the current
session, so there's no need to call `/api/_login` again.

#### Possible errors

| HTTP status code | APIError code     |
| ---------------- | ----------------- |
| 409              | user_exists       |
| 409              | email_exists      |
| 400              | invalid_username  |
| 400              | already_logged_in |

### `/api/_user [GET]`

Returns the logged in user. If not, a 401 error.

### `/api/users/{username} [GET]`

Returns a user object, or a 404 error.

### `/api/users/{username}/feed [GET]`

Returns the feed items of a user. The result sets are cursor paginated.

Feeds of banned users can only be viewed by admins. For anyone else, a 403 error
is returned.

#### Query parameters

| Name  | Description                                                 |
| ----- | ----------------------------------------------------------- |
| next  | The pagination cursor. If null, there's no next result set. |
| limit | The max number of items in a result set.                    |

#### Possible errors

| HTTP status code | APIError code  |
| ---------------- | -------------- |
| 401              | user_banned    |
| 404              | user_not_found |
| 403              | not_admin      |
| 400              | invalid_cursor |
| 400              | invalid_limit  |

### `/api/posts [GET]`

Retrieve community and site-wide posts by 'hot', 'activity', etc.

Response for normal feeds (for a request like: `/api/posts?feed=home&sort=hot`):
```
{
    "posts", // Array of posts
    "next", // Pagination cursor (null implies end of pagination)
}
```

If the query parameter `filter` is set and it's not `all`, the request is only
allowed for moderators and admins. And these result sets are page paginated
rather than cursor paginated.

Reponse for moderator feeds (for a request like `/api/posts?communityId=17692e122def73f25bd757e0&filter=deleted`):
```
{
    "noPosts", // Posts count
    "limit",
    "page", // Current page number
    "posts", // Array of posts
}
```

#### Query parameters

| Name        | Description                                                                 |
| ----------- | --------------------------------------------------------------------------- |
| feed        | One of: `home`, `all`, `community`.                                         |
| filter      | One of: `all`, `deleted`, `locked`. If not set, `all` is the default.\*     |
| sort        | One of: `latest`, `hot`, `activity`, `day`, `week`, `month`, `year`, `all`. |
| communityId | If this field is set, posts of this community will be returned.             |
| next        | The pagination cursor. If null, there's no next result set                  |
| limit       | The max number of items in a result set.                                    |

\* The filter parameter is available only if the authenticated user is a
moderator or an admin.

#### Possible errors

| HTTP Status Code | APIError code  |
| ---------------- | -------------- |
| 403              | not_admin      |
| 400              | invalid_cursor |
| 400              | invalid_limit  |
| 400              | invalid_filter |

### `/api/posts [POST]`

Creates a post object and returns it.

Request must have the following JSON body:

```
{
    // Post type. One of "text", "image", "link". Default is "text".
    "type", 

    "title", // Required
    "body", 
    "community", // Name of community, required
    "url", // Only valid for link-posts
}
```

If successful, a newly created post is returned.

### `/api/posts/{postId} [GET]`

Returns a post object.

If the query parameter `fetchCommunity` is set to `true`, `post.community` field
is populated (otherwise it's null).

### `/api/posts/{postId} [PUT]`

Updates a post. Must have the correct permissions.

JSON request body:

```
{
    "title", // New title of post
    "body", // New body of post (for text-posts)
}
```

Omit any of these fields, if you do not wish to change it.

Moderators and admins can lock a post. To lock a post pass in these query
parameters to the PUT request: `action=lock&lockAs=mods`. `action` could be one
of `lock` or `unlock`. And `lockAs` could be one of `admins` or `mods`.

Moderators and admins can change the "officially speaking" indicator by passing
in `action=changeAsUser&userGroup=admins` query parameters. Here, `userGroup`
could be one of `normal`, `mods`, or `admins`.

Post body or title is not updated on requests with an `action=` query parameter.

#### Possible errors

| HTTP Status Code | APIError code |
| ---------------- | ------------- |
| 403              | not_admin     |
| 403              | not_mod       |

### `/api/posts/{postId} [DELETE]`

Deletes a post. Returns the deleted post.

| Name          | Description                                                          |
| ------------- | -------------------------------------------------------------------- |
| deleteAs      | One of: `normal`, `mods`, `admins`. With `normal` being the default. |
| deleteContent | Boolean. If `true`, the body of the post is also deleted.\*          |

\* For link-posts this is the link. For text post this is the text. For image
posts these are the image(s).

### `/api/_postVote [POST]`

Votes a post up or down and returns the post. If already voted, then changes the
vote.

Request must have the following JSON body:
```
{
    "postId",
    "up" // true for upvotes, false for downvotes
}
```
### `/api/posts/{postID}/comments [GET]`

Returns a list of comments.

`postId` in the URL is the `publicId` of the post.