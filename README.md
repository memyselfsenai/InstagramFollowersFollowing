# InstagramFollowersFollowing

A script that compares who follows an Instagram account with who that account follows, and lists the mismatches:

- **PeopleIDontFollowBack**: accounts that follow you, but you don't follow back
- **PeopleNotFollowingMeBack**: accounts you follow, but that don't follow you back

It runs in the browser console on instagram.com, using your existing logged-in session. It does not log in, store passwords, or use an official API.

## How to run it

1. Log in to instagram.com in Chrome.
2. Open DevTools (Cmd+Option+J on macOS) and go to the **Console** tab.
3. Open `script.js`, change `username = "example_username"` at the bottom to your username, and paste the whole file into the console.
4. The first line of output is `Promise {<pending>}`. That's normal. The result arrives after the lookup and loading finish, and it's printed as an object with the two lists.

To run it again, reload the page first. Pasting into the same tab a second time fails with `Identifier 'fetchOptions' has already been declared`.

## How the code works

The script runs in four steps.

### 1. Look up the user ID (`getUserId`, `script.js:67`)

Instagram's API uses numeric user IDs, not usernames. `getUserId` tries two methods in order:

1. **`searchUserId`**: sends the same GraphQL search query that Instagram's own search box uses (`PolarisSearchBoxRefetchableQuery`) to `https://www.instagram.com/api/graphql`. It returns the matching user's `pk`. The request needs the page's `fb_dtsg` token, which is read from the page with `require("DTSGInitialData")`.
2. **`profilePageUserId`** (fallback): fetches `https://www.instagram.com/<username>/` and reads `"profile_id"` from the HTML.

If the first method throws an error or finds no exact username match, the second one runs.

### 2. Load followers and following (`concatFriendshipsApiResponse`, `getFollowers`, `getFollowing`)

These call `https://www.instagram.com/api/v1/friendships/<id>/followers/` and `.../following/`. Results come back in pages of up to `count=50`. Each response includes `next_max_id` when more pages exist. The function calls itself with that value until the last page, then combines all pages into one list.

It waits 800–1500 ms between pages so Instagram doesn't rate-limit the requests.

### 3. Compare the lists (`getUserFriendshipStats`)

Usernames are lowercased and put into two `Set`s. The script then keeps the names that appear in only one of the two sets, which gives the two output lists.

### 4. Run it

The last line calls `getUserFriendshipStats(username).then(console.log).catch(console.error)`. Progress messages are printed along the way, so you can see which stage is running.

## Known limits

- **The user search uses an internal `doc_id`** (`27706427925724183`). Instagram changes these every few weeks, so the search will eventually fail. The profile-page fallback keeps the script working in the meantime. To update the ID, open the Network tab, filter for `graphql`, type a name in Instagram's search box, and copy the `doc_id` from the request body.
- **The old `topsearch` endpoint is gone.** It returned 404 in October 2026. Don't use it.
- **The friendship endpoints (`/api/v1/friendships/`) haven't been re-tested** against the current Instagram site. If the followers or following calls return 404 or an HTML page, the same method used for search applies: watch what Instagram's own UI requests and copy that.
- **Accounts with very large follow lists** take a long time because of the pauses between pages, and they're more likely to hit rate limits (HTTP 429).
- **Instagram's terms of service** prohibit scraping and automated access. Use this only on your own account, at a low rate.
- **Results are only as accurate as the data returned.** Private accounts, blocked accounts, or pages that fail to load can make the lists incomplete. The script prints a warning about this.

## Next steps

### Fix the doc comment at the bottom

The comment above the username line still says "Replace `example_username`" with a stale "Make sure you don't delete the quotes" note. It should describe the current steps: change the username, paste the script on instagram.com, and expect a promise to return the lists.

### Run it on a schedule

The script can't run on a schedule as it is now. It depends on the browser: `window`, `document.cookie`, `require("DTSGInitialData")`, and the login cookies all exist only inside a logged-in instagram.com tab. A scheduler (cron, launchd, or GitHub Actions) has no such tab.

Scheduling needs a different setup:

1. **Rewrite it to run in Node** (or Python), with your session cookies from a file or an environment variable instead of the browser. Keep the cookies out of the repo: add a `.env` file to `.gitignore`.
2. **Get `fb_dtsg` without the browser.** It's in the HTML of any logged-in instagram.com page, so the script can fetch the home page and extract it, the same way the profile fallback reads `profile_id`.
3. **Save the results.** Write each run's output to a dated JSON file, or compare it with the previous run and print only the changes. That's what makes a scheduled job useful.
4. **Schedule it** with cron or launchd on your Mac. Pick a schedule that's not too frequent: once a day is plenty.

A headless browser (Playwright) is another option. It runs the script in a real browser profile on a schedule, so the session stays logged in. It's heavier, but it's closer to what the script already does.

### Other ideas

- Handle the 429 response: back off for longer, and stop the run after repeated failures instead of hammering the endpoint.
- Remove the `window.alert` redirect at the top if the script moves to Node.
- Add a `.gitignore` for any cookie or output files.
