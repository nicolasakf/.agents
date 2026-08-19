---
name: "x-growth-engine"
description: "X/Twitter growth engine for building audience, crafting viral content, and analyzing engagement. Use when the user wants to grow on X/Twitter, write tweets or threads, analyze their X profile, research competitors on X, plan a posting strategy, or optimize engagement. Complements social-content (generic multi-platform) with X-specific depth: algorithm mechanics, thread engineering, reply strategy, profile optimization, and competitive intelligence via web search."
license: MIT
metadata:
  version: 1.2.0
  author: Nicolas Fonteyne
  category: marketing
  updated: 2026-08-18
---

# X/Twitter Growth Engine

X-specific growth skill. For general social media content across platforms, see `social-content`. For social strategy and calendar planning, see `social-media-manager`. This skill goes deep on X.

Algorithm claims here are read from X's open-source For You repository (August 2026 release) and
are cited to specific files in [references/algorithm-signals.md](references/algorithm-signals.md).
That file also lists the widely-repeated claims that are **not** in the code. When a user asks why
something is or is not ranking, answer from the code and say plainly when a mechanic is folklore.

## Human voice (required)

Before delivering tweet, thread, reply, or hook copy, apply [../linkedin-posts/references/human-voice-social-copy.md](../linkedin-posts/references/human-voice-social-copy.md).

**Non-negotiable for paste-ready text:**

- No em dashes (—) in tweets or threads. Use line breaks, periods, or colons.
- No arrow bullets or markdown lists in the text the user posts to X.
- Avoid AI tells: "Unpopular opinion:" openers every time, perfect parallel lists, stacked hyphenated adjectives, chatbot CTAs.
- Example templates in this skill are structural guides only; **output copy** must be rewritten in plain human voice.

## When to Use This vs Other Skills

| Need | Use |
|------|-----|
| Write a tweet or thread | **This skill** |
| Plan content across LinkedIn + X + Instagram | social-content |
| Analyze engagement metrics across platforms | social-media-analyzer |
| Build overall social strategy | social-media-manager |
| X-specific growth, algorithm, competitive intel | **This skill** |

---

## Step 1 — Profile Audit

Before any growth work, audit the current X presence. Run `scripts/profile_auditor.py` with the handle, or manually assess:

### Bio Checklist
- [ ] Clear value proposition in first line (who you help + how)
- [ ] Specific niche — not "entrepreneur | thinker | builder"
- [ ] Social proof element (followers, title, metric, brand)
- [ ] CTA or link (newsletter, product, site)
- [ ] No hashtags in bio (signals amateur)

### Pinned Tweet
- [ ] Exists and is less than 30 days old
- [ ] Showcases best work or strongest hook
- [ ] Has clear CTA (follow, subscribe, read)

### Recent Activity (last 30 posts)
- [ ] Posting frequency: minimum 1x/day, ideal 3-5x/day
- [ ] Mix of formats: tweets, threads, replies, quotes
- [ ] Ratio of original posts to replies and reposts — only original posts get out-of-network
      distribution in For You, and only original posts qualify for the new-author boost
- [ ] Mutual follows in the niche — these carry the 20.0 reply weight on your original posts
- [ ] Engagement trend: improving, flat, or declining

### Visibility Labels (check this before anything else)
- [ ] Open **x.com/i/under_the_hood** and review the labels on the account and its posts
- [ ] Avatar and banner images clear of adult-content flags — both are account-level labels that
      drop every post for non-followers
- [ ] No spam, do-not-amplify, abusive or impersonation labels

A labelled account can post perfectly and still reach only its existing followers. Rule this out
first; the rest of the audit is wasted otherwise.

Run: `python3 scripts/profile_auditor.py --handle @username`

---

## Step 2 — Competitive Intelligence

Research competitors and successful accounts in your niche using web search.

### Process
1. Search `site:x.com "topic" min_faves:100` via Brave to find high-performing content
2. Identify 5-10 accounts in your niche with strong engagement
3. For each, analyze: posting frequency, content types, hook patterns, engagement rates
4. Run: `python3 scripts/competitor_analyzer.py --handles @acc1 @acc2 @acc3`

### What to Extract
- **Hook patterns** — How do top posts start? Question? Bold claim? Statistic?
- **Content themes** — What 3-5 topics get the most engagement?
- **Format mix** — Ratio of tweets vs threads vs replies vs quotes
- **Posting times** — When do their best posts go out?
- **Engagement triggers** — What makes people reply vs like vs retweet?

---

## Step 3 — Content Creation

### Tweet Types (ordered by growth impact)

#### 1. Threads (depth and follow conversion)
```
Reality check from the code: only the highest-scoring post from a conversation reaches
For You (DedupConversationFilter), so a thread competes for one slot, not many. Tweet 1
carries the thread. Later tweets earn dwell time and follows from people who opened it.

Structure:
- Tweet 1: Hook. Must stop the scroll in <7 words.
- Tweet 2: Context or promise ("I spent 6 months on this:")
- Tweets 3-N: One idea per tweet, each standalone-worthy
- Final tweet: Summary + explicit CTA ("Follow @handle for more")
- Reply to tweet 1: Restate hook + "Follow for more [topic]"

Rules:
- 5-12 tweets optimal (under 5 feels thin, over 12 loses people)
- Each tweet should make sense if read alone
- Use line breaks for readability
- No tweet should be a wall of text (3-4 lines max)
- Number the tweets or use "↓" in tweet 1 (a reader convention, not an algorithm boost)
- No em dashes or arrow lists in final copy (see Human voice)
```

#### 2. Atomic Tweets (breadth, impression farming)
```
Formats that work:
- Observation: "[Thing] is underrated. Here's why:"
- Listicle: "10 tools I use daily:\n\n1. X for Y\n2. ..."
- Contrarian: "[Bold statement]. Most teams get this wrong."
- Lesson: "I [did X] for [time]. Biggest lesson:"
- Framework: "[Concept] in 30 seconds:"

Rules:
- Under 200 characters gets more engagement
- One idea per tweet
- Links in the body carry no ranking penalty in the code (open_link weight is +0.2). Parking
  the link in your own reply has a real cost: replies from accounts a viewer does not follow
  are dropped from For You, so that reply reaches nobody out-of-network
- Question tweets drive replies, weighted 5.0 against a like's 0.5 — and 20.0 from mutuals
- Write something worth sending to one person: share-via-copy-link is the top weight at 20.0
- Write like a person, not a template (see Human voice)
```

#### 3. Quote Tweets (authority building; quote weight 5.0, same as a reply)
```
Formula: Original tweet + your unique take
- Add data the original missed
- Provide counterpoint or nuance
- Share personal experience that validates/contradicts
- Never just say "This" or "So true"
```

#### 4. Replies (relationship building, not out-of-network reach)
```
Reality check from the code: replies from an account a viewer does not follow are dropped
from For You before scoring (OONRetweetReplyFilter). Replies reach your own followers and
the conversation surface, not strangers' feeds. They are also excluded from the new-author
cold-start boost. Replies still build the mutual follows that unlock the 20.0 reply weight
on your original posts, which is where their real growth value sits.

Strategy:
- Reply to accounts 2-10x your size to get noticed and eventually followed back
- Add genuine value, not "great post!"
- Be first to reply on accounts with large audiences
- Your reply IS your content — make it tweet-worthy
- Controversial/insightful replies get quote-tweeted (that quote is what carries reach)
- Track which relationships turn into mutual follows; that is the compounding asset
```

Run: `python3 scripts/tweet_composer.py --type thread --topic "your topic" --audience "your audience"`

---

## Step 4 — Algorithm Mechanics

Ground all algorithm claims in [references/algorithm-signals.md](references/algorithm-signals.md),
which is read from X's open-source For You repository. Do not repeat folklore as fact: the
reference file marks which claims are in the code and which are not.

**Scope:** these mechanics govern the **For You feed only** — not Following, search,
notifications, or Communities.

**How weights actually work.** Each weight multiplies the model's *predicted probability* that
this viewer takes that action on this post. Weights never multiply raw engagement counts, so
ratios between them are not count equivalences. "One report cancels 468 likes" is wrong, and X's
own code comments say so.

### Weights that matter, from the code

| Action | Weight | What it means for you |
|--------|--------|----------------------|
| Share via copy link | 20.0 | The strongest positive signal. Write things people send to one person |
| Reply | 5.0 → **20.0** | 20.0 on original posts between accounts that **mutually follow** each other |
| Quote / Share via DM | 5.0 | Both far above a like |
| Follow author from post | 4.0 | Posts that convert strangers into followers are heavily rewarded |
| Share | 2.0 | |
| Repost | 1.0 | |
| Like | 0.5 | Worth 1/10th of a reply, 1/40th of a copy-link share |
| Click into post | 0.4 | |
| Open link | 0.2 | Positive. There is no link penalty in the code |
| Photo expand / video open / video quality view | 0.05 | Media has small direct weight |
| Dwell time | 0.004 per unit | Small per unit, but accrues on long reads |
| Profile click | **0.0** | Currently contributes nothing to ranking |
| Not dwelled | -0.02 | |
| Block author | -31.2 | |
| Not interested | -43.2 | |
| Mute author | -58.8 | |
| Report | -234.0 | |

**Bookmarks carry no ranking weight.** There is no bookmark weight in `ranking_scorer.rs`, so
bookmark-bait has no direct effect on For You ranking. Note the narrower claim: Phoenix does
train an `IsBookmarked` head, and bookmarks are a positive retrieval action for the *immersive*
feed, just not for home. See `references/algorithm-signals.md`.

### The mutual-follow boost is the biggest lever

An original post — not a reply, not a repost — from someone the viewer mutually follows has its
reply weight raised from 5.0 to 20.0. Practical consequences:

- Build genuine mutual follows in your niche. Ten mutuals who reply is worth more distribution
  than a thousand one-way followers who like.
- The boost applies to **original posts only**. Your replies and reposts do not get it.

### Three adjustments applied after scoring

1. **Author diversity decay.** Within one viewer's request, your 2nd post is multiplied by 0.625,
   3rd by 0.44, tailing off to a 0.25 floor. Posting 10x/day does not get you 10 slots in one
   feed. Post more to catch different refreshes, not to stack one.
2. **Out-of-network discount ×0.75.** Applied to posts from accounts the viewer does not follow —
   and also to replies and reposts from accounts they *do* follow. Original posts are the only
   format that escapes it in-network.
3. **New-author cold start.** One post per request gets lifted to roughly rank 15-16. Eligibility:
   **original post, author with ≤1,000 followers, post with <1,000 impressions**, under 24h old.
   If you are under 1,000 followers, original posts are your only path into this; replies and
   reposts are excluded by rule.

### Hard filters you cannot rank your way past

- **48 hours.** Posts older than that are dropped before scoring. There is no gradual decay curve
  in the code — the cutoff is absolute.
- **Replies and reposts from accounts a viewer does not follow are dropped entirely** from For
  You. Your replies reach your followers and the conversation surface, not strangers' feeds. This
  is the strongest correction to common reply-guy advice: replies build relationships and profile
  traffic, but they do not get out-of-network distribution here.
- **One post per conversation.** Only the highest-scoring post from a conversation survives, so a
  thread occupies one slot, not many.
- Already-seen posts, muted keywords, blocked and muted accounts, and the viewer's own posts are
  all filtered before ranking.

### What actually kills reach

Not hashtags, not editing, not links in the body — none of those appear in the code. What does
appear is **visibility filtering**, which can drop your post for non-followers while your
followers still see it normally. The recommendation-only drop rules include spam high recall,
do-not-amplify, abusive high recall, impersonation, compromised or read-only accounts, malicious
URLs, and adult-content labels — including **NSFW avatar and banner image labels**, which are
account-level and apply to every post you make.

If reach falls off a cliff, check labels first at **x.com/i/under_the_hood** before rewriting
your content strategy.

### Optimal Posting Cadence

Not from the code — the ranking code has no time-of-day or per-day-volume logic. These are
community heuristics, and author diversity decay caps how much any single viewer sees of you.

| Account size | Posts/day | Threads/week | Replies/day |
|-------------|------------|--------------|-------------|
| < 1K followers | 2-3 | 1-2 | 10-20 |
| 1K-10K | 3-5 | 2-3 | 5-15 |
| 10K-50K | 3-7 | 2-4 | 5-10 |
| 50K+ | 2-5 | 1-3 | 5-10 |

---

## Step 5 — Growth Playbook

### Week 1-2: Foundation
1. Optimize bio and pinned tweet (Step 1). Check x.com/i/under_the_hood for account labels
   before anything else — a visibility label caps out-of-network reach no matter what you post
2. Identify 20 accounts in your niche to engage with daily, with mutual follows as the goal
3. Reply 10-20 times per day to larger accounts (genuine value only). Replies do not reach
   strangers' For You feeds; they are how you get noticed and followed back
4. Post 2-3 **original** posts per day testing different formats. Under 1,000 followers,
   original posts are the only format eligible for the cold-start boost
5. Publish 1 thread

### Week 3-4: Pattern Recognition
1. Review what formats got most engagement
2. Double down on top 2 content formats
3. Increase to 3-5 posts per day
4. Publish 2-3 threads per week
5. Start quote-tweeting relevant content daily

### Month 2+: Scale
1. Develop 3-5 recurring content series (e.g., "Friday Framework")
2. Cross-pollinate: repurpose threads as LinkedIn posts, newsletter content
3. Build mutual follows with 5-10 accounts your size. Each one raises the reply weight on your
   original posts from 5.0 to 20.0 in their feed — the largest lever in the ranking code
4. Experiment with spaces/audio if relevant to niche
5. Run: `python3 scripts/growth_tracker.py --handle @username --period 30d`

---

## Step 6 — Content Calendar Generation

Run: `python3 scripts/content_planner.py --niche "your niche" --frequency 5 --weeks 2`

Generates a 2-week posting plan with:
- Daily tweet topics with hook suggestions
- Thread outlines (2-3 per week)
- Reply targets (accounts to engage with)
- Optimal posting times based on niche

---

## Scripts

| Script | Purpose |
|--------|---------|
| `scripts/profile_auditor.py` | Audit X profile: bio, pinned, activity patterns |
| `scripts/tweet_composer.py` | Generate tweets/threads with hook patterns |
| `scripts/competitor_analyzer.py` | Analyze competitor accounts via web search |
| `scripts/content_planner.py` | Generate weekly/monthly content calendars |
| `scripts/growth_tracker.py` | Track follower growth and engagement trends |

## Common Pitfalls

1. **Repeating algorithm folklore as fact** — Hashtag penalties, edit penalties and link-in-body
   penalties are not in X's published code. Check
   [references/algorithm-signals.md](references/algorithm-signals.md) before asserting a mechanic
2. **Diagnosing a reach drop as a content problem** — Check x.com/i/under_the_hood first. An
   account-level label (spam high recall, do-not-amplify, an adult-content flag on your avatar or
   banner) drops your posts for non-followers while your followers see everything as normal
3. **Chasing likes** — A like is weighted 0.5. A reply is 5.0, a quote or DM share 5.0, a
   copy-link share 20.0. Write things people forward, argue with, or send to one friend
4. **Thread tweet 1 is weak** — If the hook doesn't stop scrolling, nothing else matters, and
   only one post from the conversation reaches the feed anyway
5. **Posting more to get more slots** — Author diversity decay drops your 2nd post in a viewer's
   slate to 0.625 and floors it at 0.25. Volume catches different refreshes; it does not stack
6. **Living in the replies** — Replies are dropped from For You for anyone who does not follow
   you. Use them to build mutual follows, then let original posts carry the reach
7. **Sitting on old material** — Nothing older than 48 hours is eligible. There is no slow decay
8. **Generic bio** — "Helping people do things" tells nobody anything
9. **Copying formats without adapting** — What works for tech Twitter doesn't work for marketing Twitter

## Related Skills

- `social-content` — Multi-platform content creation
- `social-media-manager` — Overall social strategy
- `social-media-analyzer` — Cross-platform analytics
- `content-production` — Long-form content that feeds X threads
- `copywriting` — Headline and hook writing techniques
