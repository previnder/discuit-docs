<!-- omit in toc -->
# API Documentation

Documentation for the [Discuit](https://discuit.net/) API.

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
- [Objects](#objects)
  - [User](#user)
  - [Post](#post)
  - [Community](#community)
  - [CommunityRule](#communityrule)
  - [Comment](#comment)
  - [Notification](#notification)
  - [Report](#report)


## Authentication

The Discuit API uses [HTTP
Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) for
authentication.

The first call to any GET API endpoint (that does not require authentication)
returns the cookies that are needed for authenticated API requests. Namely, the
`SID` (session id) cookie and the `csrftoken` cookie. However, it is recommended
to use the `/api/_initial` endpoint for this.

For example, this command,

```bash
curl 'https://discuit.net/api/_initial' -XGET -I
```

would return these HTTP headers:

```bash
Set-Cookie: SID=aVVdZDQLCaDUFnMEwKwpbzwoNVnytESJNRVI; Path=/; Expires=Sat, 06 Jul 2024 18:57:05 GMT; HttpOnly; Secure; SameSite=Lax
Set-Cookie: csrftoken=ciSk6IDY7rQ1pHu9yueb2TXUjJQU8r1pKjisA3S7Px0=; Path=/
Date: Wed, 12 Jul 2023 18:57:05 GMT
Content-Type: text/html; charset=utf-8
```

The value in the `csrftoken` cookie needs to be passed in an `X-Csrf-Token` HTTP
header for all except GET requests to mitigate Cross Site Request Forgeries.

A login request, for example, would look like this:

```bash
curl 'http://discuit.net/api/_login' -XPOST -H 'Cookie: SID=aVVdZDQLCaDUFnMEwKwpbzwoNVnytESJNRVI' -H 'X-Csrf-Token: ciSk6IDY7rQ1pHu9yueb2TXUjJQU8r1pKjisA3S7Px0=' -d '{"username":"neo","password":"whatever"}'
```


## Errors

Most API errors return a JSON object of the following type, along with the
appropriate HTTP status code:

```js
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

```js
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

```js
{
    "username",
    "password"
}
```

If the username and password matches, the user object is returned. Otherwise, a
401 (unauthorized) error is returned.

If the user account is suspended, a 403 (forbidden) status is returned with an
APIError code `"account_suspended"`.

If the user is authenticated and the request URL has the query parameter
`action=logout`, user is logged out from the session. No request body is
required for logout. In other words, to logout a user, send a POST request to
`/api/_login?action=logout`.

### `/api/_signup [POST]`

Requests must have the following JSON body:

```js
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

```js
{
    "posts", // Array of posts
    "next", // Pagination cursor (null implies end of pagination)
}
```

If the query parameter `filter` is set and it's not `all`, the request is only
allowed for moderators and admins. And these result sets are page paginated
rather than cursor paginated.

Reponse for moderator feeds (for a request like `/api/posts?communityId=17692e122def73f25bd757e0&filter=deleted`):

```js
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

```js
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

```js
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

```js
{
    "postId",
    "up" // true for upvotes, false for downvotes
}
```
### `/api/posts/{postID}/comments [GET]`

Returns a list of comments.

`postId` in the URL is the `publicId` of the post.

## Objects

Time values are quoted strings in the RFC 3339 format with sub-second precision.

### User

```js
{
    "id": string, 
    "username": string,
    "email": string, // Could be null
    "emailConfirmedAt": time, // Could be null
    "aboutMe": string, // Could be null
    "points": int,
    "isAdmin": bool,
    "noPosts": int, // Post count of the user
    "noComments": int, // Comment count of the user
    "createdAt": time,
    "deletedAt": time, // Could be null
    "isBanned": bool, // Bool
    "bannedAt": time, // Could be null
    "notificationsNewCount": int, // Number of new notifications the user has
    
    // The list of communities that the user moderates. Could be null. This
    // field is not always populated.
    "moddingList": string[]
}
```

### Post

```js
{
    "id": string, 
    "publicId": string, // The value in https://discuit.net/gaming/post/{publicId}/
    "type": string, // One of "text", "image", "link"

    "userId": string, // Id of the author.
    "username": string, // Username of the author.

    // In what capacity the post was created. One of "normal", "admins", "mods".
    // For "speaking officialy" as a mod or an admin.
    "userGroup": string,

    "userDeleted": bool, // Indicated whether the author's account is deleted 
    "isPinned": bool,
    "communityId": string, // The id of the community the post is posted in
    "communityName": string, // The name of that community

    "title": string,
    "body": string, // Body of the post (only valid for text-posts)

    // Only valid for link-posts. Could be null.
    "link": {}

    "communityProPic": {}, 
    "communityBannerImage": {},

    "locked": bool,
    "lockedBy": string, // Who locked the post. Could be null.
    "lockedByGroup": string, // One of "admins" or "mods". Could be null.
    "lockedAt": time, // Could be null

    "upvotes": int,
    "downvotes": int,
    "hotness": int, // For ordering posts by 'hot'

    "createdAt": time,
    "editedAt": time, // Last edited time. Could be null.
    
    // Either the post created time or, if there are comments on the post, the
    // time the most recent comment was created at.
    "lastActivityAt": time,

    "deleted": bool,
    "deletedAt": time, // Could be null
    "deletedBy": string, // Id of the user who deleted the post. Could be null.
    "deletedAs": string, // One of "normal", "admins", "mods". Could be null.

    // If true, the body of the post and all associated links or images are
    // deleted.
    "deletedContent": bool,

    "deletedContentAs": string,  // One of "normal", "admins", "mods". Could be null.

    "noComments": int, // Comment count

    "commments": [], // Comments of the post. This field is not always populated.
    "commentsNext": string, // Pagination cursor. Could be null.

    // Indicated whether the authenticated user has voted. If not authenticated,
    // the value is null.
    "userVoted": bool,
    "userVotedUp": bool, // Indicates whether the authenticated user's vote is an upvote.

    "community": {} // The community object of the post. Not always populated.
}
```

### Community

```js
{
    "id": string,
    "userId": string, // Id of the user who created the community
    "name": string,
    "nsfw": bool, // Indicates whether the community hosts NSFW content
    "about": string, // Could be null
    "noMembers": int, // Member count
    "proPic": {},
    "bannerImage": {},
    "createdAt": time,
    "deletedAt": time, // Could be null

    "isDefault": bool, // This field is not always populated

    "userJoined": bool, // Indicates whether the authenticated user is a member
    "userMod": bool, // Indicates whether the authenticated user is a moderator

    "mods": [] // An array of User objects. This field is not always populated.

    "rules": [], // Array of CommunityRule. This field is not always populated

    "reportDetails": {
      "noReports": int, // Total reports count
      "noPostReports": int, // Reported posts count
      "noCommentReports": int, // Reported comments count
    }, // This field is not always populated
}
```

### CommunityRule

```js
{
  "id": int,
  "rule": string,
  "description": string, // Could be null
  "communityId": string,
  "zIndex": int, // Determines rule ordering, with lower at top
  "createdBy": string,
  "createdAt": time
}
```

### Comment

```js
{
  "id": string,
  "postId": string,
  "postPublicId": string,
  "communityId": string,
  "communityName": string,
  "userId": string,
  "username": string,

  // The capacity in which the comment was created.
  "userGroup": string, // One of "normal", "admins", "mods".
  "userDeleted": bool, // Indicates whether the author account is deleted
  "parentId": string, // Parent comment id. Could be null.
  "depth": int, // Top-most comments have a depth of 0
  "noReplies": int, // Total number of replies
  "noDirectReplies": int, // Number of direct replies

  // Comment ids of all direct ancestor comments starting from the top-most
  // comment.
  "ancestors": [], // Array of string. Could be null.

  "body": string, // Comment body
  "upvotes": int,
  "downvotes": int,
  "createdAt": time,
  "editedAt": time, // Last edit time. Could be null.

  "deletedAt": time, // Comment deleted time. Could be null.

  // User id of the person who deleted the comment.
  "deletedBy": string, // Could be null

  // In what capacity the comment was deleted.
  "deletedAs": string, // One of "normal", "admins", "mods". Could be null.

  // Indicated whether the authenticated user has voted. If not authenticated,
  // the value is null.
  "userVoted": bool,
  "userVotedUp": bool, // Indicates whether the authenticated user's vote is an upvote

  "postDeleted": bool, // Indicates whether the post the comment belongs to is deleted

  // If the post is deleted, in what capacity.
  "postDeletedAs": string // One of "normal", "admins", "mods".
}
```

### Notification

```js
{
  "id": int,

  // Type of notification. One of "new_comment", "coment_reply", "new_votes",
  // "deleted_post", "mod_add".
  "type": string,

  "notif": {}, // The actual notification

  "seen": bool,
  "seenAt": time, // Could be null
  "createdAt": time
}
```

### Report

A report is a user submitted report of a post or a comment.

```js
{
  "id": int,
  "communityId": string,
  "postId": string,
  "reason": string,
  "description": string,
  "reasonId": int,
  "type": string, // One of "post" or "comment"
  "targetId": string, // Id of the post or the comment
  "dealtAt": time, // Could be null
  "dealtby": string, // Could be null
  "createdAt": time,

  // The comment or the post the report is made against.
  "target": {}
}
```
