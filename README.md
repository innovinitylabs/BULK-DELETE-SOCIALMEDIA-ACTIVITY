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
    waitBetweenAttempts = 800,

    maxLocalRetries = 4,

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

    window.deletedTweets = 0;
    window.removedReposts = 0;
    window.skippedItems = 0;

    function delay(ms) {
        return new Promise(resolve =>
            setTimeout(resolve, ms)
        );
    }

    function log(msg) {
        console.log(
            `[${new Date().toLocaleTimeString()}] ${msg}`
        );
    }

    function isVisible(el) {

        return !!(
            el &&
            (
                el.offsetWidth ||
                el.offsetHeight ||
                el.getClientRects().length
            )
        );
    }

    function closeMenus() {

        document.body.click();
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

                const regex =
                    new RegExp(
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
                return keyword;
            }
        }

        return null;
    }

    function isRepost(article) {

        return !!article.querySelector(
            'button[data-testid="unretweet"]'
        );
    }

    function isMyTweet(article) {

        const userLinks = [
            ...article.querySelectorAll(
                'a[role="link"]'
            )
        ];

        for (const link of userLinks) {

            const href =
                link.getAttribute("href");

            if (!href) continue;

            const normalized =
                href.toLowerCase();

            if (
                normalized === `/${username.toLowerCase()}` ||
                normalized.startsWith(
                    `/${username.toLowerCase()}/status/`
                )
            ) {

                const time =
                    article.querySelector(
                        "time"
                    );

                if (
                    time &&
                    link.contains(time)
                ) {

                    return true;
                }
            }
        }

        return false;
    }

    async function findTargetArticle() {

        const articles = Array.from(
            document.querySelectorAll(
                "article[data-testid='tweet']"
            )
        );

        for (const article of articles) {

            const date =
                getTweetDate(article);

            if (!date) {
                continue;
            }

            if (
                !shouldDeleteByDate(date)
            ) {

                window.skippedItems++;

                log(
                    `Skipped by date | ${date.toDateString()}`
                );

                continue;
            }

            const repost =
                isRepost(article);

            if (
                repost &&
                deleteReposts
            ) {

                if (
                    protectReposts
                ) {

                    const keyword =
                        containsProtectedKeyword(
                            article
                        );

                    if (keyword) {

                        window.skippedItems++;

                        log(
                            `Skipped repost keyword "${keyword}"`
                        );

                        continue;
                    }
                }

                return {
                    type: "repost",
                    article,
                    date
                };
            }

            const mine =
                isMyTweet(article);

            if (
                mine &&
                deleteTweets
            ) {

                const keyword =
                    containsProtectedKeyword(
                        article
                    );

                if (keyword) {

                    window.skippedItems++;

                    log(
                        `Skipped keyword "${keyword}"`
                    );

                    continue;
                }

                return {
                    type: "tweet",
                    article,
                    date
                };
            }
        }

        return null;
    }

    async function findCaretWithRetry(
        article
    ) {

        for (
            let i = 0;
            i < maxLocalRetries;
            i++
        ) {

            const caret =
                article.querySelector(
                    "button[aria-label='More']"
                ) ||
                article.querySelector(
                    "[data-testid='caret']"
                );

            if (
                caret &&
                isVisible(caret)
            ) {

                return caret;
            }

            await delay(300);
        }

        return null;
    }

    async function tryDeleteTweet(
        article
    ) {

        for (
            let retry = 1;
            retry <= maxLocalRetries;
            retry++
        ) {

            closeMenus();

            await delay(300);

            const caret =
                await findCaretWithRetry(
                    article
                );

            if (!caret) {

                log(
                    `Caret not found (retry ${retry}/${maxLocalRetries})`
                );

                continue;
            }

            if (dryRun) {

                log(
                    `DRY RUN tweet delete | Total: ${window.deletedTweets + 1}`
                );

                return true;
            }

            caret.click();

            await delay(
                waitBetweenAttempts
            );

            const menuItems =
                document.querySelectorAll(
                    "[role='menuitem']"
                );

            let deleteItem = null;

            for (const item of menuItems) {

                const text =
                    item.innerText.toLowerCase();

                if (
                    text.includes(
                        "delete"
                    )
                ) {

                    deleteItem =
                        item;

                    break;
                }
            }

            if (!deleteItem) {

                log(
                    `Delete menu missing (retry ${retry}/${maxLocalRetries})`
                );

                closeMenus();

                await delay(500);

                continue;
            }

            deleteItem.click();

            await delay(
                waitBetweenAttempts
            );

            const confirm =
                document.querySelector(
                    "button[data-testid='confirmationSheetConfirm']"
                );

            if (
                !confirm ||
                !isVisible(confirm)
            ) {

                log(
                    `Delete confirm missing (retry ${retry}/${maxLocalRetries})`
                );

                closeMenus();

                await delay(500);

                continue;
            }

            confirm.click();

            return true;
        }

        return false;
    }

    async function tryUndoRepost(
        article
    ) {

        for (
            let retry = 1;
            retry <= maxLocalRetries;
            retry++
        ) {

            closeMenus();

            await delay(300);

            const btn =
                article.querySelector(
                    'button[data-testid="unretweet"]'
                );

            if (
                !btn ||
                !isVisible(btn)
            ) {

                log(
                    `Unretweet button missing (retry ${retry}/${maxLocalRetries})`
                );

                continue;
            }

            if (dryRun) {

                log(
                    `DRY RUN repost remove | Total: ${window.removedReposts + 1}`
                );

                return true;
            }

            btn.click();

            await delay(
                waitBetweenAttempts
            );

            const confirm =
                document.querySelector(
                    'div[role="menuitem"][data-testid="unretweetConfirm"]'
                );

            if (!confirm) {

                log(
                    `Unretweet confirm missing (retry ${retry}/${maxLocalRetries})`
                );

                closeMenus();

                await delay(500);

                continue;
            }

            confirm.click();

            return true;
        }

        return false;
    }

    let emptyPasses = 0;

    while (
        !window.stopCleaning
    ) {

        const target =
            await findTargetArticle();

        if (!target) {

            emptyPasses++;

            log(
                `No target found | Scroll ${emptyPasses}/5`
            );

            if (
                emptyPasses >= 5
            ) {

                log(
                    "No more matching tweets/reposts."
                );

                break;
            }

            window.scrollBy(
                0,
                500
            );

            await delay(1200);

            continue;
        }

        emptyPasses = 0;

        log(
            `Target ${target.type} | ${target.date.toDateString()}`
        );

        let success = false;

        if (
            target.type === "tweet"
        ) {

            success =
                await tryDeleteTweet(
                    target.article
                );

            if (success) {

                window.deletedTweets++;

                log(
                    `Tweet deleted | Total deleted: ${window.deletedTweets}`
                );
            }

        } else {

            success =
                await tryUndoRepost(
                    target.article
                );

            if (success) {

                window.removedReposts++;

                log(
                    `Repost removed | Total reposts removed: ${window.removedReposts}`
                );
            }
        }

        if (!success) {

            log(
                "Target failed after retries. Small scroll."
            );

            window.scrollBy(
                0,
                150
            );

            await delay(1000);

            continue;
        }

        await delay(
            waitAfterAction
        );
    }

    if (
        window.stopCleaning
    ) {

        log(
            "STOPPED BY USER"
        );
    }

    log(
        "========== FINISHED =========="
    );

    log(
        `Deleted tweets: ${window.deletedTweets}`
    );

    log(
        `Removed reposts: ${window.removedReposts}`
    );

    log(
        `Skipped items: ${window.skippedItems}`
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
