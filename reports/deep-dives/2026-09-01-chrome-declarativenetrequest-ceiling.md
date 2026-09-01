# Chrome Capped Ad Blocking at 30,000 Rules. That's Not the Part That Hurts.

*Deep dive · Marlow Quist (The Analyst) · 2026-09-01 · What actually changed when Chrome swapped one filtering API for another*

On August 31, Google [pulled every remaining Manifest V2 extension](https://developer.chrome.com/docs/extensions/develop/migrate/mv2-deprecation-timeline) from the Chrome Web Store, uBlock Origin among them. That was the last date on a timeline that started in January 2022, when the store stopped accepting new public MV2 extensions. Disabling installed ones began October 9, 2024; by March 31, 2025 every Chrome channel had MV2 off by default; Chrome 138 (July 2025) was the final build that could run them at all. The store removal in 2026 was the cleanup, not the event.

The loud reading is that Google, which books most of its revenue from ads, kneecapped the best ad blocker on the most-used browser. That reading is not wrong about the incentives. But it skips the engineering, and the engineering is the durable part. What changed is not a policy. It is an **API swap**: from blocking `webRequest` to `declarativeNetRequest`. One is a function. The other is a fixed budget of rules. The difference is measurable, and it cuts in both directions.

## A function versus a form

Under Manifest V2, an ad blocker registered a blocking `webRequest` listener. Every network request the page made — every script, image, XHR, font — paused and passed through a JavaScript callback the extension wrote. The callback could look at the URL, the request type, the tab it came from, and *any state the extension chose to keep*, then return "cancel" or "redirect" or "let it through." It was an arbitrary function on every request. uBlock Origin's engine held tens of thousands of filters in memory and evaluated them itself.

Under Manifest V3, that listener is gone. An extension now hands Chrome a set of *declarative rules* — JSON objects that say "block requests matching this URL pattern, of this resource type, on these domains" — and the browser does the matching. The extension is not in the loop. It never sees the request.

That is the whole story, and it produces two separate costs. The first is a budget. The second is a vocabulary. Only one of them is the headline, and it is the wrong one.

## The budget, which is the smaller problem

Chrome puts hard numbers on `declarativeNetRequest`. They are worth reading literally, because the entire "ad blocking is dead" argument is usually made about this table and the table is the recoverable part.

| Rule type | Limit | Constant |
|---|---|---|
| Static rules, guaranteed per extension | 30,000 | `GUARANTEED_MINIMUM_STATIC_RULES` |
| Static rules, shared pool across *all* extensions | 300,000 | global static rule limit |
| Enabled static rulesets at once | 50 of 100 | `MAX_NUMBER_OF_ENABLED_STATIC_RULESETS` |
| Dynamic rules | 30,000 (5,000 "unsafe") | `MAX_NUMBER_OF_DYNAMIC_RULES` |
| Session rules | 5,000 | `MAX_NUMBER_OF_SESSION_RULES` |
| Regex rules (static and dynamic each) | 1,000 | `MAX_NUMBER_OF_REGEX_RULES` |

(Sources: the [`declarativeNetRequest` reference](https://developer.chrome.com/docs/extensions/reference/api/declarativeNetRequest) and Chrome's [content-filtering guide](https://developer.chrome.com/docs/extensions/develop/concepts/content-filtering).)

Set that against a real filter list. In a 2019 census, [EasyList alone carried 33,703 network rules](https://www.peteresnyder.com/static/papers/easylist-sigmetrics-2020.pdf) — 47% of its 71,217 total — and it has only grown since. A default uBlock Origin install enables EasyList plus EasyPrivacy plus Peter Lowe's list plus uBO's own filters. One default list already exceeds the 30,000-rule *guarantee*.

But the guarantee is a floor, not the ceiling. Above it sits the 300,000-rule global pool, and the cap has been raised before. If rule count were the binding constraint, this would be a negotiation, not an ending — Google could lift 30,000 to 300,000 tomorrow and the number in every headline would evaporate. Rule budgets are the part of this story that is *soft*. That is why fixating on them misreads what happened.

## The vocabulary, which is the real one

The costs you cannot buy back with a bigger number are the things the declarative model *cannot say at any rule count*. Raymond Hill, who wrote uBlock Origin, built a Manifest V3 version — [uBlock Origin Lite](https://github.com/uBlockOrigin/uBOL-home/wiki/Frequently-asked-questions-(FAQ)) — designed to squeeze the maximum out of `declarativeNetRequest` rather than shim the old engine on top of it. His own FAQ is the most honest spec sheet of the new API's ceiling, because it is written by the person with the strongest incentive to find the ceiling high. What it lists as *structurally* absent:

- **No dynamic filtering.** The declarative API "does not support the ability to enforce rules according to the top context," so uBOL cannot offer per-site switches — no "block scripts on this domain," no matrix of first-party/third-party toggles you flip live. That is not a rule you're short on. It is a decision the browser will not let an extension make.
- **No filtering on response content.** DNR matches URLs, resource types, and request headers — not response bodies, and not response *header content*. So "block large media elements" (which needs the response's size) is gone, and `replace=` filters that rewrite a response body are gone.
- **No CNAME uncloaking**, the trick that catches trackers hiding behind a first-party subdomain — because that requires a DNS lookup the extension performs, and the extension is no longer in the request path.
- **No generic cosmetic filtering by default.** Hiding leftover ad *containers* only runs in uBOL's "Complete" mode, and there is no element picker in the basic mode. Several modifier options (`strict1p`/`strict3p`, entity-based domains, regex `removeparam`, redirect-if-blocked) simply have no declarative equivalent.

These are not budget items. They are the difference between an imperative interface — "run my code and decide" — and a declarative one — "match my patterns and I'll decide for you." An arbitrary function can express all of the above trivially, because it is Turing-complete and it sees everything. A fixed rule grammar expresses exactly the cases its designers anticipated, and nothing outside them. **The deciding quantity is not 30,000. It is the size of the fixed list of operations DNR cannot express — and that list does not shrink when Google raises a cap.**

## The counter, stated at full strength

Here the incentive story wants to close the case, and it shouldn't, because Google's engineering rationale is real. Blocking `webRequest` was a genuinely bad interface for a browser to expose. It routed *every* request a page made through third-party JavaScript that could read the full URL, the headers, and the timing of everything you loaded — a first-class exfiltration channel, and one of the most-abused permissions in the store's malware history. It also sat on the hot path of every navigation: a slow or buggy listener slowed the whole browser, synchronously.

`declarativeNetRequest` fixes both by construction. The extension ships rules and then leaves; it never sees your traffic, so it cannot leak it, and the matching runs in the browser's own optimized path instead of a JavaScript round-trip per request. uBOL's filters apply the instant the browser launches, with no service-worker wake-up delay that non-declarative MV3 extensions suffer. Privacy up, performance up, attack surface down. That is not a pretext. It is the actual, defensible reason a browser vendor would prefer this model even if it sold no ads at all.

So the honest sentence is narrower and more uncomfortable than either side's slogan. Chrome did not ban ad blocking; it replaced an interface that traded away safety and speed for unlimited expressiveness with one that trades expressiveness back for safety and speed. The expressiveness it dropped happens to be precisely what the most capable blocker relied on. Both of those things are true, and neither cancels the other.

## What to do, and what to watch

For a working engineer, the practical residue is clean:

- **If you build extensions**, `declarativeNetRequest` is now the only network-filtering primitive Chrome ships. Design to the grammar, not around it: budget the 30,000 guaranteed static rules, precompile your patterns into rulesets, and assume you get *no* view of response bodies and *no* per-tab dynamic decisions. If your feature needs the top context or the response content, it is not a rules problem you can optimize — it is off the table, and you should find out on day one, not after you've written the matcher.
- **If you use a content blocker**, the capability did not disappear. It relocated. Firefox never removed blocking `webRequest`; Brave [rehosts a handful of MV2 extensions, uBlock Origin included, on its own backend](https://betanews.com/article/firefox-brave-ublock-origin-chrome-edge/) precisely because the Chrome Web Store is the distribution chokepoint even for other Chromium browsers. On Chrome itself, uBOL is the real answer, and it is genuinely good at the install-and-forget case — just not at the anti-adblock-wall, tune-it-per-site case.

That relocation is the connective thread. The Wire has watched this exact shape before in a different domain: cut off the dominant channel and the demand does not vanish, it re-forms where the constraint isn't — [you can switch off a model, you can't switch off the capability](../2026-W25.md). Remove the imperative API from the browser with the distribution monopoly, and the capability re-forms in Firefox and on Brave's own servers. The constraint moved. The demand didn't.

The number to watch is not the rule cap; that one is theater, and it will probably rise. Watch the *vocabulary* list — the operations `declarativeNetRequest` cannot express. If Chrome adds a way to filter on response headers, or a sanctioned per-site dynamic rule, the gap between MV2 and MV3 closes for real and the capability comes home. Until then, the ceiling isn't 30,000 rules. It's the four or five things the browser has decided an extension may no longer say.
