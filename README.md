1. Press CMD + I or CTRL + I or RightClick > Inspect element
2. Paste the below script in console and press enter
```
async function CleanTwitter({
    username,

    deleteTweets = true,
    deleteReposts = true,

    deleteBefore = null,
    deleteAfter = null,

    protectedKeywords = [],

    keywordMatchType = "partial", // "partial" or "full"

    protectReposts = false,

    waitAfterAction = 2500,
    scrollDelay = 1500,

    dryRun = false
}) {

    const DELETE_BEFORE =
        deleteBefore
            ? new Date(deleteBefore)
            : null;

    const DELETE_AFTER =
        deleteAfter
            ? new Date(deleteAfter)
            : null;

    window.stopCleaning = false;

    function delay(ms) {
        return new Promise(r => setTimeout(r, ms));
    }

    function log(msg) {
        console.log(
            `[${new Date().toLocaleTimeString()}] ${msg}`
        );
    }

    function getTweetDate(article) {

        const timeEl =
            article.querySelector("time");

        if (!timeEl) return null;

        const datetime =
            timeEl.getAttribute("datetime");

        return datetime
            ? new Date(datetime)
            : null;
    }

    function shouldDeleteByDate(date) {

        if (!date) return false;

        if (
            DELETE_BEFORE &&
            date >= DELETE_BEFORE
        ) {
            return false;
        }

        if (
            DELETE_AFTER &&
            date <= DELETE_AFTER
        ) {
            return false;
        }

        return true;
    }

    function containsProtectedKeyword(article) {

        const tweetTextNode =
            article.querySelector(
                '[data-testid="tweetText"]'
            );

        if (!tweetTextNode) {
            return null;
        }

        const text =
            tweetTextNode.innerText.toLowerCase();

        for (const keyword of protectedKeywords) {

            const lowerKeyword =
                keyword.toLowerCase();

            let matched = false;

            if (
                keywordMatchType === "full"
            ) {

                const regex = new RegExp(
                    `\\b${lowerKeyword}\\b`,
                    "i"
                );

                matched =
                    regex.test(text);

            } else {

                matched =
                    text.includes(lowerKeyword);
            }

            if (matched) {

                log(
                    `Skipped protected keyword: "${keyword}"`
                );

                return keyword;
            }
        }

        return null;
    }

    function isMyTweet(article) {

        const links = [
            ...article.querySelectorAll(
                "a[href]"
            )
        ];

        return links.some(link => {

            const href =
                link.getAttribute("href");

            return (
                href === `/${username}` ||
                href.startsWith(
                    `/${username}/`
                )
            );
        });
    }

    function isRepost(article) {

        return !!article.querySelector(
            'button[data-testid="unretweet"]'
        );
    }

    async function deleteTweet(article) {

        article.scrollIntoView({
            behavior: "instant",
            block: "center"
        });

        await delay(500);

        const caret =
            article.querySelector(
                "[data-testid='caret']"
            );

        if (!caret) {

            log(
                "Caret button missing."
            );

            return false;
        }

        caret.click();

        await delay(1000);

        const menuItems = [
            ...document.querySelectorAll(
                "[role='menuitem']"
            )
        ];

        let deleteItem = null;

        for (const item of menuItems) {

            const text =
                item.innerText.toLowerCase();

            if (
                text.includes("delete")
            ) {

                deleteItem = item;

                break;
            }
        }

        if (!deleteItem) {

            log(
                "Delete option not found."
            );

            document.body.click();

            return false;
        }

        if (dryRun) {

            log(
                `DRY RUN: would delete tweet | Total deleted: ${deletedTweets + 1}`
            );

            document.body.click();

            return true;
        }

        deleteItem.click();

        await delay(1000);

        const confirm =
            document.querySelector(
                '[data-testid="confirmationSheetConfirm"]'
            );

        if (!confirm) {

            log(
                "Delete confirm missing."
            );

            return false;
        }

        confirm.click();

        log(
            `Tweet deleted | Total deleted: ${deletedTweets + 1}`
        );

        return true;
    }

    async function undoRepost(article) {

        article.scrollIntoView({
            behavior: "instant",
            block: "center"
        });

        await delay(500);

        const btn =
            article.querySelector(
                'button[data-testid="unretweet"]'
            );

        if (!btn) {

            log(
                "Unretweet button missing."
            );

            return false;
        }

        if (dryRun) {

            log(
                `DRY RUN: would undo repost | Total reposts removed: ${removedReposts + 1}`
            );

            return true;
        }

        btn.click();

        await delay(1000);

        const confirm =
            document.querySelector(
                'div[role="menuitem"][data-testid="unretweetConfirm"]'
            );

        if (!confirm) {

            log(
                "Unretweet confirm missing."
            );

            return false;
        }

        confirm.click();

        log(
            `Repost removed | Total reposts removed: ${removedReposts + 1}`
        );

        return true;
    }

    let processed = 0;
    let deletedTweets = 0;
    let removedReposts = 0;
    let skipped = 0;

    let noNewArticlesCount = 0;

    while (!window.stopCleaning) {

        const articles = [
            ...document.querySelectorAll(
                "article"
            )
        ];

        if (!articles.length) {

            log(
                "No articles found."
            );

            break;
        }

        const unprocessedArticles =
            articles.filter(
                a => !a.dataset.cleaned
            );

        if (
            !unprocessedArticles.length
        ) {

            noNewArticlesCount++;

            log(
                `No new articles found (${noNewArticlesCount}/5)`
            );

        } else {

            noNewArticlesCount = 0;
        }

        if (
            noNewArticlesCount >= 5
        ) {

            log(
                "No new tweets loaded after multiple scrolls."
            );

            break;
        }

        let acted = false;

        for (const article of articles) {

            if (
                window.stopCleaning
            ) {

                log(
                    "STOP REQUESTED"
                );

                break;
            }

            if (
                article.dataset.cleaned
            ) {
                continue;
            }

            processed++;

            log(
                `Checking article #${processed} | Deleted: ${deletedTweets} | Reposts removed: ${removedReposts} | Skipped: ${skipped}`
            );

            const date =
                getTweetDate(article);

            if (!date) {

                log(
                    "No date found."
                );

                article.dataset.cleaned =
                    "true";

                skipped++;

                continue;
            }

            log(
                `Tweet date: ${date.toDateString()}`
            );

            if (
                !shouldDeleteByDate(
                    date
                )
            ) {

                log(
                    "Skipped by date."
                );

                article.dataset.cleaned =
                    "true";

                skipped++;

                continue;
            }

            const repost =
                isRepost(article);

            log(
                `Is repost: ${repost}`
            );

            if (
                repost &&
                deleteReposts
            ) {

                if (
                    protectReposts
                ) {

                    const repostKeyword =
                        containsProtectedKeyword(
                            article
                        );

                    if (
                        repostKeyword
                    ) {

                        skipped++;

                        article.dataset.cleaned =
                            "true";

                        log(
                            `Protected repost keyword "${repostKeyword}" | Skipped: ${skipped}`
                        );

                        continue;
                    }
                }

                log(
                    `Target repost: ${date.toDateString()}`
                );

                const success =
                    await undoRepost(
                        article
                    );

                if (success) {

                    article.dataset.cleaned =
                        "true";

                    removedReposts++;

                    acted = true;

                    await delay(
                        waitAfterAction
                    );

                    break;
                }

                continue;
            }

            const matchedKeyword =
                containsProtectedKeyword(
                    article
                );

            if (
                matchedKeyword
            ) {

                skipped++;

                article.dataset.cleaned =
                    "true";

                log(
                    `Protected keyword match "${matchedKeyword}" | Skipped: ${skipped}`
                );

                continue;
            }

            const mine =
                isMyTweet(article);

            log(
                `Is mine: ${mine}`
            );

            if (
                mine &&
                deleteTweets
            ) {

                log(
                    `Target tweet: ${date.toDateString()}`
                );

                const success =
                    await deleteTweet(
                        article
                    );

                if (success) {

                    article.dataset.cleaned =
                        "true";

                    deletedTweets++;

                    acted = true;

                    await delay(
                        waitAfterAction
                    );

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

    log(
        "========== FINISHED =========="
    );

    log(
        `Processed: ${processed}`
    );

    log(
        `Deleted tweets: ${deletedTweets}`
    );

    log(
        `Removed reposts: ${removedReposts}`
    );

    log(
        `Skipped: ${skipped}`
    );
}
```

3. Now paste the below function call (customise as needed) and press enter.


Example:
Protect keywords only for your tweets:
```
CleanTwitter({
    username: "VALIPOKKANN",
    deleteTweets: true,
    deleteReposts: true,
    protectReposts: false,
    protectedKeywords: [
        "anekaroopam",
        "valiroopam"
    ],
    keywordMatchType: "partial",
    deleteBefore: "2024-01-01",
    dryRun: false
});
```


Protect reposts too: uses filtered keywords for reposts too
```
protectReposts: true
```
Stop script:
```
window.stopCleaning = true
```
