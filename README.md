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
}```

Usage examples:

Delete EVERYTHING:

```DeleteMyOldTweets({
    username: "valipokkann"
});```

Delete only before 2024:

```DeleteMyOldTweets({
    username: "valipokkann",
    deleteBefore: "2024-01-01"
});```

Delete only after 2020:

DeleteMyOldTweets({
    username: "valipokkann",
    deleteAfter: "2020-01-01"
});

Delete only between 2021 and 2023:

DeleteMyOldTweets({
    username: "valipokkann",
    deleteAfter: "2021-01-01",
    deleteBefore: "2024-01-01"
});

Safe testing mode:

DeleteMyOldTweets({
    username: "valipokkann",
    deleteBefore: "2024-01-01",
    dryRun: true
});

Then switch:

dryRun: false

once verified.
