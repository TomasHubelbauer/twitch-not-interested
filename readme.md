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

## The snippet

Start by going to the [Categories page](https://www.twitch.tv/directory). Open
the developer tools, find a category you want to permanently hide, hover over it
and click ⋮ > Not Interested. Go to the Network tab and find the GraphQL hit to
`https://gql.twitch.tv/gql`. Its payload will include the `operationName` field
set to `AddRecommendationFeedback` value. Right-click on it and select Use as
Fetch in Console. This will open the Console drawer with pre-filled code whose
first line is this:

```js
await fetch("https://gql.twitch.tv/gql#origin=twilight", {
```

Change this line to this:

```js
seed = ({
```

This will set a global variable named `seed` the script needs to work. Run this:

```js
delete seed.credentials;
delete seed.headers['Cache-Control'];
delete seed.headers['Pragma'];
delete seed.headers['User-Agent'];
```

This will remove some pre-filled headers that would prevent the `fetch` from
completing.

### Hide a single category snippet

After the above setup, run this code substituting `itemID` with a value obtained
from the `img` element in the query mentioned way above:

```javascript
seed.body = JSON.parse(seed.body);
seed.body[0].variables.input.itemID = ;
seed.body = JSON.stringify(seed.body);
await fetch('https://gql.twitch.tv/gql#origin=twilight', seed)
  .then(response => response.text())
  .then(text => console.log(text))
  .catch(console.error);
```

Run this code and it will cause the category with the provided `itemID` to be
permanently hidden.

Check the [Settings page](https://www.twitch.tv/settings/recommendations) to
verify this.

### Hide all but the allowed categories snippet

The allowed categories array is taken from the table above. The setup is as per
above and then run this script:

```js
void async function() {
  // Just Chatting: 509658, Music: 26936, Art: 509660, Chess: 743
  // Retro Gaming: 27284, Makers & Crafting: 509673,
  // Software and Game Development: 1469308723, Science & Technology: 509670
  const allowedItemIds = [509658, 26936, 509660, 743, 27284, 509673, 1469308723, 509670];
  const items = [...document.querySelectorAll('.tw-image[src*="ttv-boxart"]')]
    .map(img => ({ id: +img.src.match(/\d+/)[0], name: img.alt.slice(0, -' cover image'.length) }));
  for await (const item of items) {
    if (allowedItemIds.includes(item.id)) {
      continue;
    }
    
    seed.body = JSON.parse(seed.body);
    seed.body[0].variables.input.itemID = item.id.toString();
    seed.body = JSON.stringify(seed.body);
    await fetch('https://gql.twitch.tv/gql#origin=twilight', seed)
      .then(response => response.text())
      .then(text => console.log(item, text))
      .catch(console.error);
    await new Promise(resolve => setTimeout(resolve, 5000));
  }
}()
```

Same as with the single item script, whether it works or not can be verified on
the above linked settings page.

## Recap

1. Go to [Categories page](https://www.twitch.tv/directory)
2. Open browser developer tools
3. Find a sample category you want to delete and click ⋮ > Not Interested
4. Find the Network tab record for this action (`https://gql.twitch.tv/gql`)
5. Right-click on it and press Use as Fetch in Console
6. Change the first line of the pre-filled code to `seed = ({` and execute it
7. Run this code to avoid CORS issues with the pre-filled `fetch` headers
   ```js
   delete seed.credentials;
   delete seed.headers['Cache-Control'];
   delete seed.headers['Pragma'];
   delete seed.headers['User-Agent'];
   ```
8. Run the below code to remove all categories that are not on the list:

```js
void async function() {
  // Just Chatting: 509658, Music: 26936, Art: 509660, Chess: 743
  // Retro Gaming: 27284, Makers & Crafting: 509673,
  // Software and Game Development: 1469308723, Science & Technology: 509670
  const allowedItemIds = [509658, 26936, 509660, 743, 27284, 509673, 1469308723, 509670];
  const items = [...document.querySelectorAll('.tw-image[src*="ttv-boxart"]')]
    .map(img => ({ id: +img.src.match(/\d+/)[0], name: img.alt.slice(0, -' cover image'.length) }));
  for await (const item of items) {
    if (allowedItemIds.includes(item.id)) {
      continue;
    }
    
    seed.body = JSON.parse(seed.body);
    seed.body[0].variables.input.itemID = item.id.toString();
    seed.body = JSON.stringify(seed.body);
    await fetch('https://gql.twitch.tv/gql#origin=twilight', seed)
      .then(response => response.text())
      .then(text => console.log(item, text))
      .catch(console.error);
    await new Promise(resolve => setTimeout(resolve, 5000));
  }
}()
```
