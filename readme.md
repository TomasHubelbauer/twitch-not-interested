# Twitch "Not Interested"

[Unwanted Twitch](https://github.com/kwaschny/unwanted-twitch) is a browser
extension for hiding categories and tags on Twitch. It works by hiding the
items on the list in the DOM using CSS.

A Twitch-native way to hide categories is to use the **Not Interested** overflow
menu item.

I am not aware of a native way to hide entire tags on Twitch.

I would like to find a way to mass-mark categories as Not Interested on Twitch
and potentially devise a way for the Unwanted Twitch extension to mark items as
well whenever a user hides a category.

My motivation for this is slow performance of the Twitch tab whenever there are
many hidden items as I hide all gaming content and keep just a handful of
categories I am interested in.

When on the Categories page, `https://www.twitch.tv/directory`, this code gets
the IDs and names of the loaded categories:

```js
[...document.querySelectorAll('.tw-image[src*="ttv-boxart"]')]
  .map(img => ({ id: +img.src.match(/\d+/)[0], name: img.alt.slice(0, -' cover image'.length) }))
```

The result is an array of objects with the `id` number and `name` string fields.

The ID is pulled from the `img` element `src` attribute value. The fact that the
first number is the ID was verified by checking the GraphQL payload issued when
pressing Not Interested on the item with the given image. More on this below.
The URL example: `https://static-cdn.jtvnw.net/ttv-boxart/509658-285x380.jpg`.
In this example, `509658` is the **Just Chatting** category ID.

The name is pulled from the `img[alt]` attribute with the ` cover image` suffix
removed. This assumes English Twitch locale, which I am using.

The data of categories I want to keep:

| ID         | Name                          |
|------------|-------------------------------|
| 509658     | Just Chatting                 |
| 26936      | Music                         |
| 509660     | Art                           |
| 743        | Chess                         |
| 27284      | Retro Gaming                  |
| 509673     | Makers & Crafting             |
| 1469308723 | Software and Game Development |
| 509670     | Science & Technology          |

Whenever the Not Interested button is pressed, a GraphQL endpoint of the Twitch
API at `https://gql.twitch.tv/gql` is hit. `fetch` code obtained by right-click
menu item Copy > Copy as Fetch found in the Network tab on the line representing
the Not Interested drop-down menu item being clicked in the Firefox dev tools:

```js
let userAgent, clientId, deviceId, clientVersion, sessionId, authorization;
let itemId, sha256Hash;
const body = [
  {
    operationName: 'AddRecommendationFeedback',
    variables: {
      input: {
        category: 'NOT_INTERESTED',
        itemID: itemId, // The category ID as mentioned above
        itemType: 'CATEGORY', // TODO: Does this support tags?
        sourceItemPage: 'twitch_home',
        sourceItemRequestID: 'JIRA-VXP-2397', // TODO: Needed? Meaning?
        sourceItemTrackingID: '', // NOTE: Always empty
      }
    },
    extensions: {
      persistedQuery: {
        version: 1,
        sha256Hash, // TODO: Needed? Meaning?
      }
    }
  }
];

await fetch("https://gql.twitch.tv/gql#origin=twilight", {
    "credentials": "include",
    "headers": {
        "User-Agent": userAgent,
        "Accept": "*/*",
        "Accept-Language": "en-US",
        "Client-Id": clientId,
        "X-Device-Id": deviceId,
        "Client-Version": clientVersion,
        "Client-Session-Id": sessionId,
        "Authorization": authorization,
        "Content-Type": "text/plain;charset=UTF-8",
        "Sec-Fetch-Dest": "empty",
        "Sec-Fetch-Mode": "cors",
        "Sec-Fetch-Site": "same-site",
        "Pragma": "no-cache",
        "Cache-Control": "no-cache"
    },
    "referrer": "https://www.twitch.tv/",
    "body": JSON.stringify(body),
    "method": "POST",
    "mode": "cors"
});
```

## To-Do

Come up with a way to quickly fill in the sensitive items in the dev tools JS
context. Probably by instantiating one Not Interested action and using its GQL
payload as an input. The script would then use regex to match out the various
bits and set them to the global scope. Next, all the images would be queried as
per the above snippet and in a loop, the JQL query would be repeated replacing
the category ID each time.
