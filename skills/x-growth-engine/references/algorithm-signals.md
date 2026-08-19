# X For You Algorithm Signals

Everything in the **Verified** sections below is read directly from X's open-source For You
repository (the August 2026 release). File paths are given so claims can be re-checked against
the code. The **Unverified** section is community heuristic — sometimes useful, but not in the
code.

**Scope:** this repository covers the **For You feed only**. It does not govern the Following
(reverse-chron) feed, search, notifications, Communities, or profile timelines. Advice derived
from these weights applies to For You distribution and nothing else.

---

## The one thing to get right about weights

Weights multiply the model's **predicted probability** that *this specific viewer* takes an
action on *this specific post*. They do not multiply raw engagement counts.

Reading the ratios as count equivalences is the most common misreading of this repo, and the code
says so explicitly (`home-mixer/params/param.rs`, comment above the weight block): "one report
cancels 468 likes" is wrong. A report's baseline probability is more than 1000x lower than a
like's, so it carries a large weight simply to register at all.

Two consequences worth stating to anyone asking about brigading:

- Predictions are personalized. Reports from a cluster of hostile accounts mainly move
  recommendations for viewers who resemble those accounts.
- An action only feeds ranking if it happened on a post served in Home Timeline. Navigating
  directly to a post (e.g. a link passed around a group chat) has no ranking impact.

---

## Verified: scoring weights

Source: `home-mixer/params/param.rs`, applied in `home-mixer/scorers/ranking_scorer.rs`.
Final score = Σ (weight × predicted probability), then three adjustments (below).

### Positive

| Action | Weight | Notes |
|--------|--------|-------|
| Share via copy link | 20.0 | Largest positive weight in the model |
| Reply | 5.0 | 20.0 on original posts from mutual follows — see below |
| Quote | 5.0 | |
| Share via DM | 5.0 | |
| Follow author from post | 4.0 | |
| Share | 2.0 | |
| Repost | 1.0 | |
| Favorite (like) | 0.5 | |
| Click into post | 0.4 | |
| Open link | 0.2 | **Positive.** No link penalty exists in this code |
| Photo expand | 0.05 | |
| Video open | 0.05 | |
| Video quality view | 0.05 | Only for videos longer than 10s, and only when the *viewer* has under 10,000 followers (`home-mixer/util/candidates_util.rs`) |
| Quoted post click | 0.05 | |
| Post unexplored | 0.02 | In-network only by default |
| Dwell time (continuous) | 0.004 | Per unit of dwell |
| Dwell (binary) | 0.0 | Currently off |
| Profile click | 0.0 | **Currently zero.** Profile clicks do not move For You ranking |
| Quoted video quality view | 0.0 | Currently off |

### Negative

| Action | Weight |
|--------|--------|
| Report | -234.0 |
| Mute author | -58.8 |
| Not interested | -43.2 |
| Block author | -31.2 |
| Not dwelled | -0.02 |

### Not in the model at all

**Bookmarks are not a scored action.** `bookmark_count` is carried as post metadata in the
retrieval index (`phoenix-rankall/`), but there is no bookmark head and no bookmark weight in
`ranking_scorer.rs`. Advice built on "bookmarks are a top-tier ranking signal" is unsupported by
this code.

Also absent: hashtag counts, post length, time of day, and author follower count or account age
(except in the cold-start rule below).

### The bidirectional follow boost

`home-mixer/scorers/ranking_scorer.rs`, `bidirectional_boost_eligible()`:

Reply weight goes from 5.0 to **20.0** (5.0 plus a boost of 15.0) when all three hold:

1. the post is an original post — not a reply,
2. not a repost, and
3. viewer and author **mutually follow** each other.

This is the largest single lever in the ranking code. It rolled out in July 2026 at a boost of
20.0, then settled at 15.0; the history is in `docs/BIDIRECTIONAL_BOOST_CHANGE.md`. A parallel
dwell boost exists in the code but is set to 0.

---

## Verified: the three post-scoring adjustments

`home-mixer/scorers/ranking_scorer.rs`

**1. Author diversity decay.** Within a single request's candidate slate, the *k*-th post from
the same author is multiplied by `(1 - floor) x decay^k + floor`, with `decay = 0.5` and
`floor = 0.25`:

| Post from same author in one slate | Multiplier |
|---|---|
| 1st | 1.0 |
| 2nd | 0.625 |
| 3rd | 0.4375 |
| 4th | 0.34 |
| 5th+ | approaches the 0.25 floor |

This is per-request, not per-day. Volume does not multiply your share of one viewer's feed.

**2. Out-of-network discount.** Posts from accounts the viewer does not follow are multiplied by
**0.75**. With `EnableOonRescoreForInNetworkRepliesRetweets` on (the default), replies and
reposts from accounts the viewer *does* follow take the same 0.75. Original posts from followed
accounts do not. Under a topic request the factor is 0.5.

**3. New-author cold start.** Per request, at most **one** post is lifted to the score sitting at
rank 15-16 of the slate. To be eligible (`home-mixer/scorers/author_cold_start.rs`):

- original post — not a reply, not a repost,
- author has **1,000 followers or fewer**,
- post has **fewer than 1,000 impressions**,
- the post is not already ranked in the bottom 15% of the slate,
- (treatment arm) post is under 24h old.

For a small account this is the clearest structural advantage in the code, and it applies only to
original posts.

---

## Verified: hard filters

**Pre-scoring** (`home-mixer/filters/`, order as constructed in
`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`):

| Filter | Removes |
|---|---|
| `AgeFilter` | **Posts older than 48 hours** (`MAX_POST_AGE`, `home-mixer/params/config.rs`) |
| `OONRetweetReplyFilter` | **All replies and reposts from accounts the viewer does not follow**, plus replies with a missing parent |
| `SelfTweetFilter` | The viewer's own posts |
| `DropDuplicatesFilter`, `RetweetDeduplicationFilter` | Repeats across sources; repeated reposts of one post |
| `PreviouslySeen/ServedPostsFilter` (x3) | Anything already shown or served |
| `MutedKeywordFilter`, `AuthorSocialgraphFilter` | Muted keywords; blocked and muted accounts |
| `OONNsfwSimclustersFilter` | SimClusters posts from adult-flagged authors, for non-followers |
| `IneligibleSubscriptionFilter` | Subscriber-only posts the viewer cannot access |
| `NewUserMinEngagementFilter` | For new accounts, low-engagement out-of-network posts |
| `Brazil2026ElectionFilter` | Accounts reported to Brazil's Electoral Court, for non-followers |
| `InventoryHoldoutFilter` | A configured share of posts (default 0%) |

The 48-hour cutoff is absolute. There is no recency decay curve anywhere in the ranking code — a
post is eligible or it is not.

**Post-selection**, after the order is fixed:

| Filter | Removes |
|---|---|
| `VFFilter` | Posts visibility filtering answered `drop` for |
| `AncillaryVFFilter` | Posts whose parent, quoted or reposted post was dropped |
| `DedupConversationFilter` | **All but the highest-scoring post per conversation** |

`DedupConversationFilter` matters for threads: a self-reply chain is one conversation, so at most
one post from it reaches the feed. A thread does not occupy multiple slots.

**Slate size:** 50 candidates selected (`TOP_K_CANDIDATES_TO_SELECT`), 35 posts returned
(`RESULT_SIZE`), Who-to-Follow inserted at position 6 — `home-mixer/params/config.rs`.

---

## Verified: visibility filtering

`visibility-filtering/rules/registry.rs` returns ALLOW, INTERSTITIAL, or DROP per post and
viewer. The first rule that says drop ends evaluation. Two rule sets:

**Base rules — apply to everyone, followers included.** Suspended, deactivated or erased author,
protected account, viewer blocks or mutes, spam label, legal takedown, several "freedom of speech
not reach" labels (hateful conduct, violent speech, abuse, civic integrity), nullcasted posts,
plus interstitials for adult and graphic media.

**Recommendation-only rules — drop the post for non-followers while followers still see it.**
This set is where account-level reputation bites:

- spam high recall (post and account level)
- do-not-amplify (post label, and a non-follower account label)
- adult content: high recall, high precision, text, card image, near-perfect account
- **NSFW avatar image and NSFW banner image** — profile images can suppress reach
- abusive high recall, impersonation high precision
- compromised account, read-only account
- malicious URL, DMCA media, geo-restricted media

The practical read: a label you cannot see can cut out-of-network distribution to zero while
everything still looks normal to your followers. X's **Under the Hood** tool
(`x.com/i/under_the_hood`, code in `under-the-hood/`) reports the labels on your own account and
posts. Check it before diagnosing a reach drop as anything else.

Labels come from `botmaker/` + `botmaker-rules/` + `scarecrow/` (event rules),
`abuse-enforcement-service/` (model scores), `agatha/` (blocks and reports relative to
favorites), `bdsm/` (inauthentic behavior patterns), `user-cred-v2/` (PageRank over the follow
and engagement graph), and `grox/` + `media-model-proxy/` + `clip/` (text and media classifiers).
Some botmaker rules and all Grox prompts are deliberately withheld from the repo.

---

## Verified: candidate sources

`home-mixer/params/param.rs`

| Source | Cap | What it is |
|---|---|---|
| `thunder/` | 1,200 | In-network — recent posts from followed accounts, held in memory |
| `phoenix/` retrieval | 1,000 | Out-of-network — viewer and post embedded as vectors, nearest neighbours |
| `simclusters/` | — | Out-of-network — accounts and posts clustered by who engages with what |

The viewer's recent action sequence is capped at 1,024 events over a 300-second aggregation
window (`MaxSeqLengthScoring`, `UAS_WINDOW_TIME_MS`). That sequence is the model's main input:
what a viewer recently engaged with drives what they get, far more than anything static.

---

## Unverified: community heuristics

Not present in the open-source code. Treat as folklore that may still be worth testing, and label
it as such when advising:

- **"Links in the body kill reach."** No link penalty exists in the ranking code; `open_link`
  carries a positive 0.2 weight. Only `MaliciousUrlOonDropRule` touches links, and only for
  flagged malicious URLs. Note the real cost of the workaround: replies from accounts a viewer
  does not follow are dropped outright by `OONRetweetReplyFilter`, so a link parked in your own
  first reply reaches nobody out-of-network.
- **"More than 2 hashtags is a spam signal."** No hashtag handling in the ranking or filtering
  code at all.
- **"Editing within 30 minutes suppresses reach."** Nothing in the code.
- **"Numbered threads (1/, 2/) get boosted."** Nothing in the code.
- **"Tweet 1 gets 10-50x the impressions of tweet 5."** Plausible as a funnel effect, but
  `DedupConversationFilter` means later tweets in a chain are generally not competing for feed
  slots in the first place.
- **Optimal posting times.** No time-of-day logic in the ranking code. Timing matters through the
  48-hour window and through whether your audience is online to generate early engagement.
- **"Image posts get 2-3x engagement."** Media carries small direct weights (photo expand 0.05,
  video open 0.05, video quality view 0.05). Any media advantage runs through dwell and
  downstream engagement, not a media bonus.
- **Premium/Blue "higher reply ranking."** Not in this repo. Subscription handling here only
  covers access to subscriber-only posts.

## Sources

- X open-source For You repository, August 13-14 2026 releases (repo `README.md`)
- `home-mixer/params/param.rs` — weights, mirrored from production feature-switch defaults
- `home-mixer/scorers/ranking_scorer.rs`, `home-mixer/scorers/author_cold_start.rs`
- `visibility-filtering/rules/registry.rs`
- `docs/BIDIRECTIONAL_BOOST_CHANGE.md`
