# The X Growth Playbook for Accounts Under 1,000 Followers

**Based strictly on the X algorithm repository** (`home-mixer/`, `phoenix/`, and related paths in this repo).

**Scope note:** The number **1,000** does **not** appear as a special threshold in the code; “under 1,000 followers” framing below is **inferred only** from mechanisms that shrink **in-network** reach (fewer people follow you ⇒ fewer Thunder in-network pulls). Constants like `NEW_USER_MIN_FOLLOWING` are loaded from external `params` — **exact values are not supported by this repository.**

---

## 1. The Big Idea

**What the For You / Home feed is trying to do (in plain English)**  
For each person opening the feed, the system builds a **candidate list** of posts from several **sources** (people they follow, plus recommended posts from other systems), then **scores** those posts using a machine-learning ranker that outputs many **“would this person do X with this post?”** guesses (favorite, reply, repost, dwell, follow you, etc.), blends in **negative** guesses (not interested, block, mute, report), applies **rules that remove or downrank** posts, and finally keeps what survives—**including a step that stops the same conversation from crowding the feed** (`DedupConversationFilter`).

**Evidence:** `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` (sources ~250–257, filters ~274–289, scorers ~291–300, post-selection filters ~317–321); `home-mixer/scorers/ranking_scorer.rs` (`compute_weighted_score` ~125–170, diversity ~190–217, out-of-network multiplier ~220–275); `home-mixer/filters/dedup_conversation_filter.rs` (~8–37).

**What that means for a small account**  
You are not “speaking to the whole platform” in one step. You are trying to get your post into **someone else’s candidate pool** (especially **outside** their follow list), then win a **comparison** against everything else in that pool using **predicted reactions** for *that viewer*.

**Biggest mindset shift**  
Treat reach as a **pipeline**, not a single moment: **candidate → score → filter → slot**. A post can die **before** ranking (filtered out), or **after** ranking (conversation dedup, visibility filters), or simply **lose the score race** against stronger candidates (`TopKScoreSelector` uses `candidate.score` — `home-mixer/selectors/top_k_score_selector.rs` ~8–14).

**Confidence:** High for “pipeline mental model”; Medium for “small account” framing (follower count not encoded as a special case in the cited files).

---

## 2. How Reach Actually Happens on X

### Step 1 — Your post has to become a candidate

**What happens:** Multiple **sources** add posts: recent posts from accounts the **viewer** follows (`ThunderSource`), recommended posts (`TweetMixerSource`), Phoenix retrieval (`PhoenixSource`, plus topics/MoE variants), or cached posts when the request is in cache mode (`CachedPostsSource`).

**Why it matters for a small account:** In-network reach only reaches people who **follow you** (Thunder uses the viewer’s `followed_user_ids` — `thunder_source.rs` ~31–38). Everyone else depends on **out-of-network** sources, which are **turned off** if the request is `in_network_only` (`phoenix_source.rs` ~63–67; `tweet_mixer_source.rs` ~22–24).

**What to do:** Design for **discovery paths**, not only for your current followers—and know that some viewers may have recommendations disabled (see Section 5, Reach Killer: recommendations off).

**Evidence:** `phoenix_candidate_pipeline.rs` ~250–257; `thunder_source.rs` ~31–43; `phoenix_source.rs` ~62–68; `tweet_mixer_source.rs` ~22–24.

### Step 2 — The system matches posts to people using history and retrieval context

**What happens:** Before scoring, the pipeline loads **aggregated action sequences** for the viewer (`ScoringSequenceQueryHydrator`, `RetrievalSequenceQueryHydrator` registered in `phoenix_candidate_pipeline.rs` ~186–191). Phoenix retrieval **requires** `retrieval_sequence` (`phoenix_source.rs` ~73–76). The ranker model in `phoenix/` consumes **hashed user, history, and candidate posts/authors** (`phoenix/recsys_model.py`, `RecsysBatch` ~126–144).

**Why it matters:** The system is built to personalize from **what people actually did before**, not from your follower count alone.

**What to do:** You cannot “configure” this in the UI from this repo—but you *can* understand that **your post competes on predicted fit for each viewer**, not on a global popularity score in this code.

**Evidence:** `phoenix_candidate_pipeline.rs` ~186–191; `phoenix_source.rs` ~73–76; `phoenix/recsys_model.py` ~126–144.

### Step 3 — The system scores it

**What happens:** `PhoenixScorer` calls the prediction service with `ProductSurface::HomeTimelineRanking` for For You (`phoenix_scorer.rs` ~72–77). If `scoring_sequence` is missing, it returns **default** scores (`phoenix_scorer.rs` ~79–81). Then `RankingScorer` builds a **weighted sum** of many predicted actions, applies **author diversity** (same author repeated in one batch gets downweighted), and applies an **extra multiplier** to **out-of-network** posts (`ranking_scorer.rs` ~125–275). An optional `VMRanker` may **replace** the final `score` when enabled (`vm_ranker.rs` ~17–58).

**Why it matters:** “Good post” in this code means **high predicted positive actions and low predicted negative actions for that viewer**, after diversity and out-of-network rules.

**What to do:** Think in terms of **many behaviors** (favorite, reply, repost, dwell, follow, shares, quotes—not one vanity metric). See Section 4.

**Evidence:** `phoenix_scorer.rs` ~72–81, ~107–114; `ranking_scorer.rs`; `vm_ranker.rs` ~17–58.

### Step 4 — The system filters risky or ineligible posts

**What happens:** A long list of **pre-score filters** removes duplicates, failed hydration, old posts, self-posts for the viewer, subscription-ineligible posts, previously seen/served, muted keywords, social graph blocks, video exclusions when requested, topic filters, etc. (`phoenix_candidate_pipeline.rs` ~274–288). After scoring, **visibility filters** and **conversation dedup** run (~317–321).

**Why it matters:** A post can be “great” and still **never show** if it fails a filter.

**What to do:** Avoid behaviors that map to **blocks/mutes**, avoid **paywalled mismatch** for non-subscribers, and understand **freshness** is a hard gate (`AgeFilter` wired at `phoenix_candidate_pipeline.rs` ~277).

**Evidence:** `phoenix_candidate_pipeline.rs` ~274–321; individual filters (e.g. `home-mixer/filters/author_socialgraph_filter.rs`, `muted_keyword_filter.rs`, `ineligible_subscription_filter.rs`, `age_filter.rs`).

### Step 5 — It competes for a slot in the feed mix

**What happens:** Top candidates are selected by score (`top_k_score_selector.rs`). The separate **For You feed composer** blends organic posts with ads and modules (`home-mixer/selectors/blender_selector.rs` ~24–51).

**Why it matters:** Ranking wins you a chance; **mixing** decides final placement of non-post items.

**Evidence:** `top_k_score_selector.rs`; `blender_selector.rs`.

---

## 3. The Small Account Problem

**Limited follower graph (supported)**  
Thunder only pulls candidates from IDs in the **viewer’s** `followed_user_ids` (`thunder_source.rs` ~31–38). If few people follow you, **fewer viewers** get you through that path—not because the code names “1,000,” but because the **in-network surface area** is smaller.

**Need for out-of-network discovery (supported)**  
`PhoenixSource` and `TweetMixerSource` are explicit **non–in-network-only** paths (`phoenix_source.rs` ~63–67; `tweet_mixer_source.rs` ~22–24). If a viewer is forced to `in_network_only`, those sources are off (`server.rs` ~75–76 with `allow_for_you_recommendations == Some(false)`).

**“Legibility” to the recommender (partially supported)**  
The model side uses **hashed identifiers and viewer history** (`phoenix/recsys_model.py` `RecsysBatch`). The repo does **not** spell out what content features Phoenix learns — **not supported by this repository** beyond “history + candidates go in, many action predictions come out.”

**“Early engagement” as a live feedback knob (not supported here)**  
Serving reads **aggregated sequences** (`phoenix_candidate_pipeline.rs` ~186–191). Whether *your* new likes instantly change the next ranking call for others is **not supported by this repository.**

**Avoiding negative signals (supported as model outputs + filters)**  
The score explicitly includes predicted **not interested / block / mute / report / not dwelled** terms (`ranking_scorer.rs` ~166–170), and graph filters remove blocked/muted relationships (`author_socialgraph_filter.rs`).

**SimClusters / RealGraph / TwHIN / “author reputation score” (not supported)**  
Not present as named systems in the files reviewed for this playbook.

---

## 4. What the Algorithm Appears to Reward

### Signal: Predicted positive engagement bundle

**Plain-English meaning:** The ranker adds up weighted predictions that you will favorite, reply, repost, click, expand photos, watch video (VQV), share (including DM / copy link), dwell, quote, click quoted material, follow the author, plus continuous dwell signals (`ranking_scorer.rs` ~146–165; weight keys ~41–66).

**What this means for a small creator:** You are not optimizing one button. You are optimizing **a bundle of behaviors the system explicitly models** (also listed as `ACTIONS` in `phoenix/runners.py` ~233–252).

**Why it can help reach:** Higher combined weighted score survives `TopKScoreSelector` (`top_k_score_selector.rs`).

**What to do:** Create posts where **many healthy interactions are plausible** for strangers (not manipulative engagement farming).

**What not to do:** Assume only one signal matters unless your live weights say so—**weights are not in this repo** (`ScoringWeights::from_params` — `ranking_scorer.rs` ~42–66).

**When it will not help:** If you never enter candidates, or filters remove you, or `VMRanker` overwrites scoring (`vm_ranker.rs`).

**Evidence:** File `home-mixer/scorers/ranking_scorer.rs`, struct `ScoringWeights` / `compute_weighted_score`, lines ~12–170; file `phoenix/runners.py`, `ACTIONS`, lines ~233–252.

**Confidence:** High (structure); Medium (relative importance of each action).

### Signal: “In-network” status for a viewer

**Plain-English meaning:** If the viewer follows you (or it’s their own post), the candidate is marked in-network (`in_network_candidate_hydrator.rs` ~29–36).

**Why it can help reach:** Out-of-network posts get multiplied by `effective_oon_weight` (`ranking_scorer.rs` ~220–275).

**What to do:** Growing **real followers** moves more people onto the **in-network** path and avoids the extra out-of-network multiplier for those viewers.

**What not to do:** Buy fake followers — harmful, not analyzed, systems outside this repo may apply.

**When it will not help:** If the viewer still does not follow you, you remain out-of-network for them.

**Evidence:** `in_network_candidate_hydrator.rs` ~29–36; `ranking_scorer.rs` ~220–275.

**Confidence:** High.

### Signal: Freshness (post age)

**Plain-English meaning:** Posts older than the configured max age are removed (`AgeFilter` in pipeline `phoenix_candidate_pipeline.rs` ~277; TweetMixer also filters by age `tweet_mixer_source.rs` ~75–77).

**What this means for a small creator:** Old posts stop being eligible in this pipeline.

**Why it can help reach:** You stay inside the candidate window.

**What to do:** Treat distribution as **time-sensitive** in this system.

**What not to do:** Expect weeks-old posts to keep competing here.

**Evidence:** `phoenix_candidate_pipeline.rs` ~277; `tweet_mixer_source.rs` ~75–77; `age_filter.rs`.

**Confidence:** High.

### Signal: Follows (as a modeled outcome)

**Plain-English meaning:** `follow_author_score` is part of the weighted sum (`ranking_scorer.rs` ~165).

**What this means:** The system explicitly models probability of **following the author** from the post.

**What to do:** Make the **author** worth following in the context of the post — **how** the model maps content to that score is **not supported by this repository.**

**Evidence:** `ranking_scorer.rs` ~59, ~165.

**Confidence:** High that it exists; Low/Medium that any specific tactic increases it.

### Signal: Video watch (VQV) — conditional

**Plain-English meaning:** Video-related prediction is scaled by helper weights that depend on **minimum video duration** parameters (`ranking_scorer.rs` ~132–137, ~152).

**What this means:** Very short videos can end up with **zero** contribution from the VQV term if the duration gate zeroes the weight.

**What to do:** If you use video, understand the system **gates** video engagement contribution by duration rules from server config — **exact threshold not supported by this repository** (external `params`).

**Evidence:** `ranking_scorer.rs` ~132–144, ~152.

**Confidence:** Medium.

---

## 5. What Can Hurt Reach

### Reach Killer: Block / mute / block-by relationships

**Plain-English meaning:** If the viewer blocked or muted the author (or related parties), the post is removed (`author_socialgraph_filter.rs`).

**Why it hurts:** Hard filter — never reaches scoring for that viewer.

**What it looks like in real creator behavior:** Aggressive reply tactics, harassment, or audiences who mass-mute topics/accounts.

**How to avoid it legitimately:** Do not post in ways that cause people to **block or mute** you.

**Evidence:** `home-mixer/filters/author_socialgraph_filter.rs` (~10–59).

**Confidence:** High.

### Reach Killer: Muted keywords (viewer-side)

**Plain-English meaning:** If tweet text matches a viewer’s muted keyword tokens, the post is removed (`muted_keyword_filter.rs`).

**Why it hurts:** Hard filter.

**What it looks like:** Certain words or topics your target audience mutes.

**How to avoid:** You cannot see others’ mute lists — this repo only proves **the mechanism exists**.

**Evidence:** `home-mixer/filters/muted_keyword_filter.rs` (~20–55).

**Confidence:** High for mechanism; Low for “which words.”

### Reach Killer: Visibility / safety filtering

**Plain-English meaning:** Posts with bad visibility reasons are dropped (`vf_filter.rs`); posts flagged ancillary drop are removed (`ancillary_vf_filter.rs`).

**Why it hurts:** Hard post-selection removal.

**What it looks like:** Content classes the visibility system treats as undeliverable (exact mapping **not supported by this repository** beyond drop vs keep).

**How to avoid:** Stay inside normal, legitimate content boundaries; do not try to hack filters.

**Evidence:** `phoenix_candidate_pipeline.rs` ~304–320; `vf_filter.rs`; `ancillary_vf_filter.rs`.

**Confidence:** High that filters exist; Medium on exact triggers.

### Reach Killer: Subscription-only without subscription

**Plain-English meaning:** If a post is subscription-only for an author the viewer does not subscribe to, it is removed (`ineligible_subscription_filter.rs`).

**Why it hurts:** Hard filter.

**What it looks like:** Paywalled posts shown to non-subscribers.

**How to avoid:** Match paywalled posts to **subscribed** audiences — or use non-paywalled posts for broad discovery.

**Evidence:** `ineligible_subscription_filter.rs` (~9–28).

**Confidence:** High.

### Reach Killer: Already seen / already served

**Plain-English meaning:** Posts related to IDs the viewer already saw (client `seen_ids` + bloom filters) or already served in-session can be removed.

**Why it hurts:** You cannot “re-win” the same impression surface endlessly in this pipeline.

**What it looks like:** Reposting the same thing repeatedly.

**How to avoid:** Expect **deduping of attention**, not infinite repeats.

**Evidence:** `previously_seen_posts_filter.rs`; `previously_served_posts_filter.rs`.

**Confidence:** High.

### Reach Killer: Retweet deduplication

**Plain-English meaning:** Only one candidate survives per underlying original tweet ID (`retweet_deduplication_filter.rs`).

**Why it hurts:** Many RTs of the same original **do not multiply** candidates here.

**Evidence:** `retweet_deduplication_filter.rs` (~10–27).

**Confidence:** High.

### Reach Killer: Conversation crowding

**Plain-English meaning:** After scoring, only the **highest-scoring** post per conversation id remains (`dedup_conversation_filter.rs`).

**Why it hurts:** A thread of many tweets can **collapse to one slot** in this step.

**What it looks like:** Splitting one idea across many rapid replies.

**How to avoid:** If you thread, know **one winner** survives — make it count.

**Evidence:** `dedup_conversation_filter.rs` (~8–37).

**Confidence:** High.

### Reach Killer: Predicted negative reactions (model outputs)

**Plain-English meaning:** The score subtracts weighted predictions of not interested / block / mute / report / not dwelled (`ranking_scorer.rs` ~166–170, ~83–84).

**Why it hurts:** Pushes the combined score down vs competitors.

**What it looks like:** Content patterns people signal against.

**How to avoid:** Avoid content and behavior patterns that drive **real** negative feedback — not gaming.

**Evidence:** `ranking_scorer.rs` ~59–64, ~166–170.

**Confidence:** High for “exists in score”; Medium for operational control.

### Reach Killer: Recommendations turned off for a viewer

**Plain-English meaning:** If `allow_for_you_recommendations == Some(false)`, the query becomes `in_network_only`, which **disables** Phoenix/TweetMixer sourcing paths (`server.rs` ~75–76; `phoenix_source.rs` ~63–67; `tweet_mixer_source.rs` ~22–24).

**Why it hurts:** You lose discovery routes to that viewer.

**Evidence:** `server.rs` ~75–76; source `enable` methods cited.

**Confidence:** High.

### Reach Killer: Same author spam in one ranking batch

**Plain-English meaning:** Author diversity downweights additional posts from the same author within the same scored batch (`ranking_scorer.rs` ~190–217).

**Why it hurts:** Flooding many posts at once can **hurt relative ranking** even if each post is okay alone.

**Evidence:** `ranking_scorer.rs` ~190–217.

**Confidence:** High.

---

## 6. The Practical Growth Playbook

### A. Before You Post

| Checklist item | Why it matters | Code-backed mechanism | Evidence |
|----------------|----------------|-------------------------|----------|
| Ask: “Will this viewer even get recommendations?” | If they are `in_network_only`, you lose OON sources | `server.rs` `allow_for_you_recommendations` → `in_network_only` | `server.rs` ~75–76 |
| Ask: “Is this post eligible on age?” | Old posts are filtered | `AgeFilter` | `phoenix_candidate_pipeline.rs` ~277; `age_filter.rs` |
| Avoid posting text that matches common mute tokens **for your target audience** | Hard removal | `MutedKeywordFilter` | `muted_keyword_filter.rs` |
| If paywalled: match audience to subscribers | Hard removal | `IneligibleSubscriptionFilter` | `ineligible_subscription_filter.rs` |
| If threading: assume **one** tweet may survive dedup | Post-selection collapse | `DedupConversationFilter` | `dedup_conversation_filter.rs` ~8–37 |
| Know RT campaigns don’t stack as many candidates | Dedup by original tweet | `RetweetDeduplicationFilter` | `retweet_deduplication_filter.rs` ~10–27 |

### B. When You Post

- **Freshness:** You are racing the clock (`AgeFilter`, TweetMixer age check). **Evidence:** `phoenix_candidate_pipeline.rs` ~277; `tweet_mixer_source.rs` ~75–77.
- **Reply / quote / media:** The model receives **tweet type signals** in `TweetInfo` (e.g. `is_reply`, `is_quote`, `has_media` in `home-mixer/models/candidate.rs` `as_tweet_info` ~133–148). **What the model does with them** is **not supported by this repository** beyond “they exist in the payload.”
- **Links / hashtags:** **Not supported by this repository** as explicit ranking inputs in the reviewed files.
- **Video:** VQV-related terms are **gated** by duration logic (`ranking_scorer.rs` ~132–137).

### C. After You Post

- You **cannot** read Phoenix probabilities from this playbook; production logs are **not supported by this repository.**
- **Supported conceptual checks:** Did the post likely pass **age**, **graph**, **keyword**, **subscription**, **seen/served** gates? Those are the hard failures in code. **Evidence:** filter list `phoenix_candidate_pipeline.rs` ~274–288 and post-selection ~317–321.

### D. Weekly Growth System

| Review | Why | Evidence |
|--------|-----|----------|
| Follower growth (real) | More viewers pull you in-network via Thunder | `thunder_source.rs` ~31–38; `in_network_candidate_hydrator.rs` ~29–36 |
| Whether you rely on one tweet vs many in a thread | Conversation dedup | `dedup_conversation_filter.rs` |
| Whether you depend on RT chains | Retweet dedup | `retweet_deduplication_filter.rs` |
| Whether you hit paywall mismatches | Subscription filter | `ineligible_subscription_filter.rs` |

---

## 7. What Works According to the Codebase

### Grow real followers (still the clearest “surface area” win)

**What to do:** Earn follows from people you actually want as readers.

**Why it works:** Thunder builds the in-network candidate pool from each viewer’s `followed_user_ids` (`thunder_source.rs` ~31–38), and in-network posts skip the out-of-network multiplier (`ranking_scorer.rs` ~272–275).

**What algorithmic mechanism it connects to:** `ThunderSource` + `InNetworkCandidateHydrator` + `RankingScorer`.

**How a small account should apply it:** Prioritize **real followers** who care over noisy tactics that create blocks/mutes.

**When it fails:** Blocks/mutes/VF still remove you for that viewer.

**Evidence:** `thunder_source.rs`; `in_network_candidate_hydrator.rs`; `ranking_scorer.rs`.

**Confidence:** High.

### Design for the full engagement bundle the model scores

**What to do:** Posts where favorites, replies, reposts, quotes, profile visits, shares, dwell, and “follow author” are realistically possible — **without** manipulation.

**Why it works:** `RankingScorer` sums weighted predictions across those heads (`ranking_scorer.rs` ~146–165).

**What algorithmic mechanism it connects to:** Phoenix predictions → `RankingScorer`.

**How a small account should apply it:** Clear premise, clear reason to reply, clear reason to follow — **as outcomes**, not hacks.

**When it fails:** Missing `scoring_sequence` neutralizes Phoenix scoring path (`phoenix_scorer.rs` ~79–81).

**Evidence:** `ranking_scorer.rs`; `phoenix/runners.py` `ACTIONS`; `phoenix_scorer.rs`.

**Confidence:** High for structure; Medium for tactics.

### Respect the “one post per conversation” reality

**What to do:** Put your strongest version in **one** tweet when you care about For You slotting.

**Why it works:** `DedupConversationFilter` keeps the highest score per conversation (`dedup_conversation_filter.rs` ~18–29).

**Evidence:** `dedup_conversation_filter.rs`.

**Confidence:** High.

### Stay inside eligibility rules (especially paywalls + safety)

**What to do:** Match subscription content to subscribers; avoid content classes that trip VF drops.

**Why it works:** Hard filters remove ineligible posts (`ineligible_subscription_filter.rs`; `vf_filter.rs`).

**Evidence:** cited files.

**Confidence:** High.

---

## 8. What Does Not Work According to the Codebase

| Claim | Repository verdict | Why | Evidence | Better code-backed alternative |
|--------|-------------------|-----|----------|-------------------------------|
| “Hashtags always help reach” | **Not supported** | No hashtag feature in reviewed filters/scorers | File review | Focus on candidate + score pipeline (Sections 2–4). |
| “Posting more always helps” | **Unsupported / can backfire** | Author diversity downweights repeats in one batch | `ranking_scorer.rs` ~190–217 | Space posts; avoid spam volume. |
| “Threads are always better” | **Not supported; can hurt slotting** | Conversation dedup keeps one | `dedup_conversation_filter.rs` | One flagship post per convo. |
| “Links always kill reach” | **Not supported** | No URL-based filter found in reviewed files | N/A | N/A |
| “Blue check guarantees reach” | **Not supported** | No verification field in `RankingScorer` | `ranking_scorer.rs` | N/A |
| “Likes are all that matter” | **Disproved as ‘only thing’** | Many other weighted heads | `ranking_scorer.rs` ~146–165 | Optimize bundle, not one metric. |
| “Replying to big accounts guarantees growth” | **Not supported** | No “big account” bonus in `RankingScorer` | `ranking_scorer.rs` | N/A |
| “Deleting low posts helps future reach” | **Not supported** | No delete-history signal in cited pipeline | Not found | N/A |
| “Engagement pods help” | **Unsupported + disallowed** | Manipulation; not modeled as legitimate | Policy + not in code | N/A |
| “One viral post guarantees future reach” | **Not supported** | No ‘momentum memory’ in these serving files | Not found | N/A |

---

## 9. Low-Reach Diagnosis

### Problem: My post got low reach

**Possible cause 1: It never became a candidate for most viewers**  
**Why this could happen:** OON sources off (`in_network_only`), or retrieval missing (`retrieval_sequence` required for Phoenix retrieval — `phoenix_source.rs` ~73–76), or Thunder never includes you for viewers who do not follow you (`thunder_source.rs`).  
**How to recognize it:** Strong engagement only from followers, little from non-followers — **analytics not supported by this repository**; this is a hypothesis aligned to sources.  
**What to change:** Grow real follows; understand some viewers may be in-network-only (`server.rs` ~75–76).  
**Evidence:** `phoenix_source.rs`; `tweet_mixer_source.rs`; `thunder_source.rs`; `server.rs`.

**Possible cause 2: It lost the score race**  
**Why this could happen:** Weighted predictions low vs other candidates (`ranking_scorer.rs`).  
**How to recognize it:** **Not supported by this repository** (no public score UI).  
**What to change:** Rework content toward the **multi-head** bundle (Section 4).  
**Evidence:** `ranking_scorer.rs`.

**Possible cause 3: Hard filter removed it**  
**Why this could happen:** Age, blocks/mutes, muted keywords, subscription rules, seen/served, VF (`phoenix_candidate_pipeline.rs` filter lists).  
**How to recognize it:** **Analytics not supported by this repository.**  
**What to change:** Fix eligibility issues (Section 5).  
**Evidence:** `phoenix_candidate_pipeline.rs` ~274–321.

**Possible cause 4: Conversation dedup ate your extra tweets**  
**Why this could happen:** Only one highest-scoring tweet per conversation remains (`dedup_conversation_filter.rs`).  
**How to recognize it:** Thread parts get uneven distribution.  
**What to change:** Consolidate.  
**Evidence:** `dedup_conversation_filter.rs`.

**Possible cause 5: VM ranker changed ordering**  
**Why this could happen:** If enabled, VM ranker sets `score` from an external service (`vm_ranker.rs` ~17–58).  
**How to recognize it:** **Not supported by this repository.**  
**Evidence:** `vm_ranker.rs`.

---

## 10. The Small Account Daily Operating System

### Daily

- **Do:** Ship posts that fit **freshness** windows. **Why it matters:** Age filters. **Code-backed mechanism:** `AgeFilter` / TweetMixer age. **Evidence:** `phoenix_candidate_pipeline.rs` ~277; `tweet_mixer_source.rs` ~75–77.
- **Do:** Avoid patterns that create **blocks/mutes**. **Why it matters:** Hard removal + negative prediction heads. **Code-backed mechanism:** `AuthorSocialgraphFilter`; `RankingScorer` negative terms. **Evidence:** `author_socialgraph_filter.rs`; `ranking_scorer.rs` ~166–170.

### Every post

- **Do:** Assume **one** tweet may carry the conversation in For You. **Why it matters:** Dedup. **Code-backed mechanism:** `DedupConversationFilter`. **Evidence:** `dedup_conversation_filter.rs`.
- **Do:** If paywalled, match subscribers. **Why it matters:** `IneligibleSubscriptionFilter`. **Evidence:** `ineligible_subscription_filter.rs`.

### Weekly

- **Review:** Whether you depend on **followers vs discovery** paths. **Why it matters:** Thunder vs Phoenix/TweetMixer. **Evidence:** `thunder_source.rs`; `phoenix_source.rs`; `tweet_mixer_source.rs`.
- **Review:** RT dedup effects on campaigns. **Why it matters:** `RetweetDeduplicationFilter`. **Evidence:** `retweet_deduplication_filter.rs`.

### Stop doing

- **Stop:** Flooding many tweets from the same account in one blast expecting each to rank independently. **Why it can hurt:** Author diversity downweights repeats. **Evidence:** `ranking_scorer.rs` ~190–217.
- **Stop:** Assuming hashtags/links/verification are magic levers. **Why:** **Not supported by this repository** in cited code.

---

## 11. Experiments for Accounts Under 1,000 Followers

| Experiment | Hypothesis | Why the code suggests this may matter | What to change | What to measure | What result would confirm it | What result would disprove it | Evidence |
|------------|------------|----------------------------------------|----------------|-----------------|-------------------------------|--------------------------------|----------|
| A vs B post “bundle” styles | Different patterns change predicted multi-head totals | `RankingScorer` sums many heads | Two formats | Saves, replies, reposts, follows if you have analytics | More multi-type engagement | No difference | `ranking_scorer.rs` |
| Single flagship vs multi-tweet thread | One tweet survives dedup | `DedupConversationFilter` | Same idea: one tweet vs split | Distribution across thread parts | One part dominates | Even split | `dedup_conversation_filter.rs` |
| Video length above duration gate | VQV term may matter more when weight non-zero | `vqv_weight` / duration params | Shorter vs longer video | Relative performance | Longer wins | No difference | `ranking_scorer.rs` ~132–137 |

---

## 12. Myths and Unsupported Advice

| Claim | Supported by Repository? | Verdict | Explanation |
|-------|---------------------------|---------|---------------|
| SimClusters / RealGraph / TwHIN drive this codepath | No | **Not supported** | Names not found in reviewed files. |
| Hashtag algorithm | No | **Not supported** | No hashtag module in cited pipeline. |
| Link penalty | No | **Not supported** | No URL filter in cited files. |
| Blue-check boost in this ranker | No | **Not supported** | Not in `RankingScorer`. |
| “Post consistently” (cadence) | No | **Not supported** | No cadence optimizer in cited files (diversity is batch-level). |
| “Reply guys grow fast” via big-account reply bonus | No | **Not supported** | No such term in `RankingScorer`. |
| Raw public like count directly ranks posts here | No | **Misleading** | Ranking uses **predicted** actions via Phoenix + weights (`ranking_scorer.rs`). |

---

## 13. The Final Playbook Summary

### Do more of this

- Win **real followers** (Thunder in-network path + skip OON penalty for those viewers). **Evidence:** `thunder_source.rs`; `in_network_candidate_hydrator.rs`; `ranking_scorer.rs` ~272–275.
- Design for **many positive modeled behaviors**, not one vanity metric. **Evidence:** `ranking_scorer.rs`; `phoenix/runners.py` `ACTIONS`.
- Protect eligibility: **freshness**, **graph**, **paywall rules**, **visibility**. **Evidence:** filters in `phoenix_candidate_pipeline.rs`.

### Do less of this

- **Spam volume** from the same account in one scoring batch (diversity downweight). **Evidence:** `ranking_scorer.rs` ~190–217.
- **Threads that assume every tweet gets For You space** (dedup). **Evidence:** `dedup_conversation_filter.rs`.
- **Duplicate RT strategies** expecting multiple candidates for the same original tweet. **Evidence:** `retweet_deduplication_filter.rs`.

### Ignore this unless more evidence appears

Hashtag science, link superstitions, verification myths, SimClusters/RealGraph storytelling, “engagement pods,” and anything requiring **missing** modules (`build_prediction_request`, full `params` / `util` implementations) — **not supported by this repository.**

### The main rule for small accounts

**Reach is a pipeline:** get into the right **candidate sources** for each viewer, then win on **multi-action predictions** while avoiding **hard filters** and **conversation dedup**. Small accounts feel it hardest because **in-network is tiny**—so every **eligible, high-scoring, non-duplicated** post matters more.

---

## Evidence appendix (primary files)

| File | What it proves |
|------|----------------|
| `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` | Full post pipeline: sources, filters, scorers, post-selection. |
| `home-mixer/scorers/ranking_scorer.rs` | Weighted multi-head score, diversity, OON multiplier. |
| `home-mixer/scorers/phoenix_scorer.rs` | Phoenix predict + gating on `scoring_sequence`. |
| `home-mixer/sources/thunder_source.rs` | In-network candidate generation from follows. |
| `home-mixer/sources/phoenix_source.rs` | Out-of-network Phoenix retrieval + `enable` / `retrieval_sequence`. |
| `home-mixer/sources/tweet_mixer_source.rs` | TweetMixer recommendations + age gate. |
| `home-mixer/server.rs` | `allow_for_you_recommendations` → `in_network_only`. |
| `home-mixer/filters/*` | Hard removals. |
| `phoenix/recsys_model.py` (`RecsysBatch`) | Model input shape (hashes, history). |
| `phoenix/runners.py` (`ACTIONS`) | Named prediction heads. |
