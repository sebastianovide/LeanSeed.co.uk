---
category: tech
tags: notes
meta: How to let users download files from Google Cloud Storage when they are not publicly accessible.
---

# Introduction
Geovation has just developed a [Progressive Web App](https://developers.google.com/web/progressive-web-apps/) for [Timepix](https://www.timepix.uk). The app stores photos in different resolutions and allows users to download them based on their ownership. For this project we've decided to implement an API with [Node.js](https://nodejs.org) deployed in [Google App Engine](https://cloud.google.com/appengine/). The client is based on [ionic](https://ionicframework.com/), authentication on [Firebase Authentication](https://firebase.google.com/docs/auth/) and the users ownership  (used for authorization) are saved in [Google Cloud Datastore](https://cloud.google.com/datastore/).

## Problem
The client must provide a link to download a picture that the user has purchased using the less API power as possible. We could simply create an endpoint that lets the API read the image and just respond with it. But that would use the network twice (from the client to the API and from the API to the storage) and extra CPU power. The pictures are not public otherwise they would be accessible even to the users than don't have purchased them.

![download via GAE](/assets/download_via_GAE.png)

## Solution
An endpoint that returns a temp url that lets the user download the image leveraging the Google infrastructure. When the user clicks on the image, we call Firebase to retrieve a user idToken and then call the API with the image ID and the idToken just retrieved. The API would call Firebase again with that idToken and retrieve the user ID. With that user ID, the API can validate if the user has access to the picture querying the Datastore and then generate a link that will be valid for just few seconds: just enough time to let the browser start the download. The download starts as soon as the browser gets the temp URL.

![download via direct link](/assets/download_via_direct_link.png)

## Some relevant code
### get Firebase user's idToken
```firebase.auth().currentUser.getIdToken() ```: See more at [https://firebase.google.com/docs/auth/web/manage-users](https://firebase.google.com/docs/auth/web/manage-users)
### get Firebase userId from idToken
```firebaseAdmin.auth().verifyIdToken(idtoken) ```: See more at [https://firebase.google.com/docs/auth/admin/verify-id-tokens](https://firebase.google.com/docs/auth/admin/verify-id-tokens)
### generate temp URL
```javascript
export function genTempPubUrl(pPath): Promise <string> {
    const expires = new Date().getTime() + 10 * 1000;
    const imageFile = gcloud("key-file.json").bucket("my-bucket").file(pPath);
    return imageFile.getSignedUrl({ action: "read", expires}).then(urls => urls[0]);
}
```
It returns a URL pointing to the image that is valid for only 10 seconds. See more at [https://cloud.google.com/storage/docs/access-control/signed-urls](https://cloud.google.com/storage/docs/access-control/signed-urls)

## Concerns
* Who has the URL can download the image: True. But as the links are valid just for few seconds, it doesn't present any relevant risk of image leaking in the net.
* the token can leak in some hackers blog: True but not usable. The authentication is using well established security practices and all managed by Firebase.
