# BULK-DELETE-SOCIALMEDIA-ACTIVITY
BULK DELETE TWITTER, INSTAGRAM and FEW OTHER CODE SNIPPETS


```
async function DeleteMyOldTweets({
    username,
    deleteBefore = null,
    deleteAfter = null,
    waitAfterDelete = 2500,
    scrollDelay = 1500,
    dryRun = false
}) {
    const DELETE_BEFORE = deleteBefore ? new Date(deleteBefore) : null;
    const DELETE_AFTER = deleteAfter ? new Date(deleteAfter) : null;
    function delay(ms) {
        return new Promise(r => setTimeout(r, ms));
    }
    function log(msg) {
        console.log(`[${new Date().toLocaleTimeString()}] ${msg}`);
    }
    async function deleteTweet(article) {
        const caret = article.querySelector("[data-testid='caret']");
        if (!caret) {
            log("No caret button.");
            return false;
        }
        caret.click();
        await delay(1000);
        const menuItems = [...document.querySelectorAll("[role='menuitem']")];
        let deleteItem = null;
        for (const item of menuItems) {
            const text = item.innerText.toLowerCase();
            if (text.includes("delete")) {
                deleteItem = item;
                break;
            }
        }
        if (!deleteItem) {
            log("Delete option not found.");
            document.body.click();
            return false;
        }
        if (dryRun) {
            log("DRY RUN: would delete tweet.");
            document.body.click();
            return true;
        }
        deleteItem.click();
        await delay(1000);
        const confirm = document.querySelector("[data-testid='confirmationSheetConfirm']");
        if (!confirm) {
            log("Confirm button missing.");
            return false;
        }
        confirm.click();
        log("Deleted tweet.");
        return true;
    }
    function isMyTweet(article) {
        const links = [...article.querySelectorAll("a[href]")];
        return links.some(link => {
            const href = link.getAttribute("href");
            return (
                href === `/${username}` ||
                href.startsWith(`/${username}/`)
            );
        });
    }
    function getTweetDate(article) {
        const timeEl = article.querySelector("time");
        if (!timeEl) return null;
        const datetime = timeEl.getAttribute("datetime");
        if (!datetime) return null;
        return new Date(datetime);
    }
    function shouldDelete(date) {
        if (!date) return false;
        if (DELETE_BEFORE && date >= DELETE_BEFORE) {
            return false;
        }
        if (DELETE_AFTER && date <= DELETE_AFTER) {
            return false;
        }
        return true;
    }
    let deleted = 0;
    let skipped = 0;
    while (true) {
        const articles = [...document.querySelectorAll("article")];
        if (!articles.length) {
            log("No tweets found.");
            break;
        }
        let acted = false;
        for (const article of articles) {
            if (article.dataset.processed) continue;
            article.dataset.processed = "true";
            const mine = isMyTweet(article);
            if (!mine) {
                skipped++;
                continue;
            }
            const date = getTweetDate(article);
            if (!date) {
                log("Tweet date not found.");
                continue;
            }
            if (!shouldDelete(date)) {
                log(`Keeping tweet from ${date.toDateString()}`);
                continue;
            }
            log(`Target tweet: ${date.toDateString()}`);
            const success = await deleteTweet(article);
            if (success) {
                deleted++;
                acted = true;
                await delay(waitAfterDelete);
                break;
            }
        }
        window.scrollBy(0, 1500);
        await delay(scrollDelay);
        if (!acted) {
            log("Scrolling for more tweets...");
        }
    }
    log(`Finished. Deleted: ${deleted}, Skipped: ${skipped}`);
}
```

Usage examples:

Delete EVERYTHING:

```
DeleteMyOldTweets({
    username: "valipokkann"
});
```

Delete only before 2024:

```
DeleteMyOldTweets({
    username: "valipokkann",
    deleteBefore: "2024-01-01"
});
```

Delete only after 2020:
```
DeleteMyOldTweets({
    username: "valipokkann",
    deleteAfter: "2020-01-01"
});
```
Delete only between 2021 and 2023:
```
DeleteMyOldTweets({
    username: "valipokkann",
    deleteAfter: "2021-01-01",
    deleteBefore: "2024-01-01"
});
```
Safe testing mode:
```
DeleteMyOldTweets({
    username: "valipokkann",
    deleteBefore: "2024-01-01",
    dryRun: true
});
```
Then switch:
```
dryRun: false
```
once verified.

#V2

support:

* protected keywords
* repost undo
* dry run
* date range
* own tweets only

all in ONE unified script.

Much cleaner.

This version:

* deletes tweets
* removes reposts
* skips protected keywords
* respects date filters
* works even if dates are null
* supports dryRun mode
```
async function CleanTwitter({
    username,
    deleteTweets = true,
    deleteReposts = true,

    deleteBefore = null,
    deleteAfter = null,

    protectedKeywords = [],

    waitAfterAction = 2500,
    scrollDelay = 1500,

    dryRun = false
}) {

    const DELETE_BEFORE = deleteBefore ? new Date(deleteBefore) : null;
    const DELETE_AFTER = deleteAfter ? new Date(deleteAfter) : null;

    window.stopCleaning = false;

    function delay(ms) {
        return new Promise(r => setTimeout(r, ms));
    }

    function log(msg) {
        console.log(`[${new Date().toLocaleTimeString()}] ${msg}`);
    }

    function getTweetDate(article) {

        const timeEl = article.querySelector("time");

        if (!timeEl) return null;

        const datetime = timeEl.getAttribute("datetime");

        return datetime ? new Date(datetime) : null;
    }

    function shouldDeleteByDate(date) {

        if (!date) return false;

        if (DELETE_BEFORE && date >= DELETE_BEFORE) {
            return false;
        }

        if (DELETE_AFTER && date <= DELETE_AFTER) {
            return false;
        }

        return true;
    }

   function containsProtectedKeyword(article) {

    const tweetTextNode = article.querySelector(
        '[data-testid="tweetText"]'
    );

    if (!tweetTextNode) {
        return null;
    }

    const text = tweetTextNode.innerText.toLowerCase();

    for (const keyword of protectedKeywords) {

        const regex = new RegExp(
            `\\b${keyword.toLowerCase()}\\b`,
            "i"
        );

        if (regex.test(text)) {

            log(`Skipped protected keyword: "${keyword}"`);

            return keyword;
        }
    }

    return null;
}

    function isMyTweet(article) {

        const links = [...article.querySelectorAll("a[href]")];

        return links.some(link => {

            const href = link.getAttribute("href");

            return (
                href === `/${username}` ||
                href.startsWith(`/${username}/`)
            );
        });
    }

    function isRepost(article) {

        return !!article.querySelector(
            'button[data-testid="unretweet"]'
        );
    }

    async function deleteTweet(article) {

        const caret = article.querySelector(
            "[data-testid='caret']"
        );

        if (!caret) {
            log("Caret button missing.");
            return false;
        }

        caret.click();

        await delay(1000);

        const menuItems = [
            ...document.querySelectorAll("[role='menuitem']")
        ];

        let deleteItem = null;

        for (const item of menuItems) {

            const text = item.innerText.toLowerCase();

            if (text.includes("delete")) {
                deleteItem = item;
                break;
            }
        }

        if (!deleteItem) {
            log("Delete option not found.");
            document.body.click();
            return false;
        }

        if (dryRun) {
            log("DRY RUN: would delete tweet.");
            document.body.click();
            return true;
        }

        deleteItem.click();

        await delay(1000);

        const confirm = document.querySelector(
            '[data-testid="confirmationSheetConfirm"]'
        );

        if (!confirm) {
            log("Delete confirm missing.");
            return false;
        }

        confirm.click();

        log("Tweet deleted.");

        return true;
    }

    async function undoRepost(article) {

        const btn = article.querySelector(
            'button[data-testid="unretweet"]'
        );

        if (!btn) {
            log("Unretweet button missing.");
            return false;
        }

        if (dryRun) {
            log("DRY RUN: would undo repost.");
            return true;
        }

        btn.click();

        await delay(1000);

        const confirm = document.querySelector(
            'div[role="menuitem"][data-testid="unretweetConfirm"]'
        );

        if (!confirm) {
            log("Unretweet confirm missing.");
            return false;
        }

        confirm.click();

        log("Repost removed.");

        return true;
    }

    let processed = 0;
    let deletedTweets = 0;
    let removedReposts = 0;
    let skipped = 0;

    while (!window.stopCleaning) {

        const articles = [
            ...document.querySelectorAll("article")
        ];

        if (!articles.length) {
            log("No articles found.");
            break;
        }

        let acted = false;

        for (const article of articles) {

            if (window.stopCleaning) {
                log("STOP REQUESTED");
                break;
            }

            if (article.dataset.cleaned) continue;

            article.dataset.cleaned = "true";

            processed++;

            log(`Checking article #${processed}`);

            const date = getTweetDate(article);

            if (!date) {

                log("No date found.");

                skipped++;

                continue;
            }

            log(`Tweet date: ${date.toDateString()}`);

            if (!shouldDeleteByDate(date)) {

                log("Skipped by date.");

                skipped++;

                continue;
            }

            const matchedKeyword =
                containsProtectedKeyword(article);

            if (matchedKeyword) {

                skipped++;

                log(
                    `Protected keyword match "${matchedKeyword}"`
                );

                continue;
            }

            const repost = isRepost(article);

            log(`Is repost: ${repost}`);

            if (repost && deleteReposts) {

                log(
                    `Target repost: ${date.toDateString()}`
                );

                const success = await undoRepost(article);

                if (success) {

                    removedReposts++;

                    acted = true;

                    await delay(waitAfterAction);

                    break;
                }
            }

            const mine = isMyTweet(article);

            log(`Is mine: ${mine}`);

            if (mine && deleteTweets) {

                log(
                    `Target tweet: ${date.toDateString()}`
                );

                const success =
                    await deleteTweet(article);

                if (success) {

                    deletedTweets++;

                    acted = true;

                    await delay(waitAfterAction);

                    break;
                }
            }
        }

        window.scrollBy(0, 1500);

        await delay(scrollDelay);

        if (!acted) {
            log("Scrolling...");
        }
    }

    if (window.stopCleaning) {
        log("STOPPED BY USER");
    }

    log("========== FINISHED ==========");
    log(`Processed: ${processed}`);
    log(`Deleted tweets: ${deletedTweets}`);
    log(`Removed reposts: ${removedReposts}`);
    log(`Skipped: ${skipped}`);
}
```
Example usage:

Safe dry run:
```
CleanTwitter({
    username: "valipokkann",
    deleteTweets: true,
    deleteReposts: true,
    deleteBefore: "2024-01-01",
    protectedKeywords: [
        "anekaroopam",
        "valiroopam",
        "kurmagati"
    ],
    dryRun: true
});
```
Real execution:
```
CleanTwitter({
    username: "valipokkann",
    deleteTweets: true,
    deleteReposts: true,
    deleteBefore: "2024-01-01",
    protectedKeywords: [
        "anekaroopam",
        "valiroopam",
        "kurmagati"
    ],
    dryRun: false
});
```
Stop script anytime:
```
window.stopCleaning = true
```

