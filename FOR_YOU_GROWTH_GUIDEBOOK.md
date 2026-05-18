# X/Twitter For You Growth Guidebook

**Based strictly on the provided algorithm codebase** (README, `home-mixer/`, `phoenix/`, `thunder/`, `candidate-pipeline/`, `grox/` under this repository).

**Repository gaps:** `home-mixer` references `crate::util` (e.g. `phoenix_request`, `candidates_util`, `score_normalizer`) and `crate::params` but those modules are **not present** in this workspace snapshot. Anything that would depend only on those files is marked **not supported by this repository**.

---

## 1. Bottom Line for Growth

### What seems to increase reach (viewer-side predictions the system optimizes for)

The ranker’s objective is **not** a single “engagement” number. `PhoenixScorer` calls the Phoenix prediction service with the viewer’s **aggregated action sequence** (`scoring_sequence`) and `ProductSurface::HomeTimelineRanking` for For You (`home-mixer/scorers/phoenix_scorer.rs`, ~67–84). `RankingScorer` then forms a **linear combination** of many predicted action probabilities (favorite, reply, repost, dwell, follow author, shares, quotes, etc.) minus negative actions (not interested, block, mute, report, not dwelled), with weights from feature switches (`home-mixer/scorers/ranking_scorer.rs`, `ScoringWeights::from_params` and `compute_weighted_score`, ~41–173). **Higher predicted positive actions and lower predicted negative actions** raise that combined score before selection.

### What seems to reduce reach

- **Never entering the candidate pool** for a viewer: out-of-network sources are skipped when `in_network_only` is true (`TweetMixerSource` / `PhoenixSource` `enable`, `home-mixer/sources/tweet_mixer_source.rs` ~22–24, `phoenix_source.rs` ~63–67); query can be forced in-network when `viewer_data.allow_for_you_recommendations == Some(false)` (`home-mixer/server.rs` ~75–76).
- **Pre-scoring removal:** self-posts, duplicates, age, blocked/muted graph, muted keywords, previously seen/served, subscription paywall without subscription, video exclusion when requested, topic/snooze/new-user topic filters (see `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` ~274–288 and individual `home-mixer/filters/*.rs`).
- **Lower score vs others:** OON multiplier on `in_network == Some(false)` (`RankingScorer::effective_oon_weight`, `ranking_scorer.rs` ~220–238, 271–275); **author diversity** downweights repeated same-author posts in the same batch (`apply_author_diversity`, ~186–217).
- **Post-selection removal:** `VFFilter`, `AncillaryVFFilter`, `DedupConversationFilter` (`home-mixer/filters/vf_filter.rs`, `ancillary_vf_filter.rs`, `dedup_conversation_filter.rs`).
- **Optional score override:** when `EnableVMRanker` is true, `VMRanker` may **overwrite** `score` (`home-mixer/scorers/vm_ranker.rs` ~17–58).

### What is not proven here

Exact numeric weights, production-only behavior inside Phoenix/TweetMixer/Thunder/TES/Strato/VM ranker services, full contents of missing `util` / external `params`, and whether production still matches this snapshot.

### The most important creator takeaway

Reach is **(a)** eligibility, **(b)** being in the **candidate set** for viewers who can receive you, then **(c)** Phoenix + `RankingScorer` predicting many **positive and negative** outcomes per viewer, then **(d)** filters and dedup. There is no separate “manual quality score” module in this repo beyond ML predictions, explicit weights, diversity, OON multiplier, optional VM ranker, and the listed filters.

---

## 2. What This Repository Can and Cannot Prove

### Can prove

Orchestration in `home-mixer/`: query hydration, candidate **sources**, **hydrators**, **filters**, **scorers**, **selectors**, post-selection steps, and how the For You **feed** blends posts with ads/modules (`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs`, `for_you_candidate_pipeline.rs`, `selectors/blender_selector.rs`).

The **shape** of Phoenix ranker inputs/outputs in `phoenix/recsys_model.py` (`RecsysBatch`, hash embeddings) and the **action head layout** in `phoenix/runners.py` (`ACTIONS` ~233–253, probability mapping ~403–424).

### Cannot prove

- Exact production weight values (`RankingScorer` reads `Params` from `xai_feature_switches`, not defined in this tree).
- Internal algorithms of Thunder, Phoenix retrieval, TweetMixer, TES, Strato, VM ranker (only client calls appear).
- Whether `WeightedScorer` in `home-mixer/scorers/weighted_scorer.rs` is used in production (**not** registered in `PhoenixCandidatePipeline::build_with_clients`; live stack uses `RankingScorer`, `phoenix_candidate_pipeline.rs` ~291–300).
- Any detail that exists only in missing `util` / `params` sources.

---

## 3. How the For You / Home Timeline Appears to Work

End-to-end: **`ForYouFeedServer` → `ForYouCandidatePipeline` → `ScoredPostsSource` → `ScoredPostsServer::run_pipeline` → `PhoenixCandidatePipeline`** (`home-mixer/for_you_server.rs`, `for_you_candidate_pipeline.rs`, `sources/scored_posts_source.rs`, `scored_posts_server.rs`). The outer For You pipeline’s own `hydrators` / `filters` / `scorers` are **empty** (`for_you_candidate_pipeline.rs` ~247–257); ranking work lives inside `PhoenixCandidatePipeline`.

### Candidate sources → Candidate generation

`PhoenixCandidatePipeline` registers **five** sources (`phoenix_candidate_pipeline.rs` ~250–257):

1. **Thunder** — in-network posts from followed accounts (`home-mixer/sources/thunder_source.rs`).
2. **TweetMixer** — `Product::HOME_RECOMMENDED_TWEETS` (`home-mixer/sources/tweet_mixer_source.rs`), disabled when `in_network_only` or `has_cached_posts`.
3. **Phoenix retrieval** — requires `retrieval_sequence` (`home-mixer/sources/phoenix_source.rs` ~73–76).
4. **PhoenixTopicsSource** / **PhoenixMOESource** — additional retrieval variants (`phoenix_candidate_pipeline.rs` ~241–246).
5. **CachedPostsSource** — when `has_cached_posts` (`home-mixer/sources/cached_posts_source.rs`).

**Why it matters:** If you are not returned by Thunder, TweetMixer, or Phoenix paths, you are not ranked in this pipeline.

### Hydration (pre-score)

Includes in-network flag, core TES data, quotes, video duration, media, subscription author, Gizmoduck, blocked-by, filtered topics, language (`phoenix_candidate_pipeline.rs` ~259–272). **In-network** is true if author is viewer or in `followed_user_ids` (`home-mixer/candidate_hydrators/in_network_candidate_hydrator.rs` ~29–36).

### Feature extraction

No separate hand-built feature vector for the main Phoenix ranker in-repo: Phoenix uses **hashed post/author IDs and history** (`phoenix/recsys_model.py`, `RecsysBatch` ~126–144). Additional fields (e.g. engagement counts, mutual-follow Jaccard) are hydrated; mutual-follow Jaccard runs in **post-selection** hydrators (`phoenix_candidate_pipeline.rs` ~304–314).

### Ranking / scoring

1. **PhoenixScorer** — gRPC `predict`; disabled when `has_cached_posts` (`phoenix_scorer.rs` ~63–65). If `scoring_sequence` is missing, returns default scores (~79–81).
2. **RankingScorer** — weighted sum of `PhoenixScores` + author diversity + OON multiplier (`ranking_scorer.rs`).
3. **VMRanker** — optional `score` override (`vm_ranker.rs`).

### Filtering (pre-score)

List in `phoenix_candidate_pipeline.rs` ~274–288 (`DropDuplicatesFilter`, `CoreDataHydrationFilter`, `AgeFilter`, `SelfTweetFilter`, `RetweetDeduplicationFilter`, `IneligibleSubscriptionFilter`, seen/served filters, `MutedKeywordFilter`, `AuthorSocialgraphFilter`, `VideoFilter`, `TopicIdsFilter`, `NewUserTopicIdsFilter`).

### Selection

`TopKScoreSelector` uses `candidate.score` and `TOP_K_CANDIDATES_TO_SELECT` (`home-mixer/selectors/top_k_score_selector.rs` ~8–14).

### Post-selection hydration & filters

VF, brand safety, tweet type metrics, following-replied users, mutual-follow Jaccard (`phoenix_candidate_pipeline.rs` ~304–314); then `VFFilter`, `AncillaryVFFilter`, `DedupConversationFilter` (~317–321).

### Mixing / blending → final output

`BlenderSelector` merges organic `ScoredPost`s with ads, inserts prompts and Who-To-Follow, pins Push-to-Home (`home-mixer/selectors/blender_selector.rs`).

---

## 4. The Ranking Signals That Matter

| Signal | Positive / Negative / Filter | Pipeline stage | Creator can influence? | Practical meaning | Evidence | Confidence |
|--------|------------------------------|----------------|------------------------|-------------------|----------|------------|
| Predicted P(favorite), P(reply), P(repost), P(photo_expand), P(click), P(profile_click), P(vqv), P(share*), P(dwell), P(quote), P(quoted_click), P(follow_author), dwell_time, click_dwell_time | Positive (weighted) | Ranking | Indirectly | Model associates your content with these actions for similar viewers | `ranking_scorer.rs`; `phoenix/runners.py` | High for “in formula”; Medium for “how to move it” |
| Predicted P(not_interested), P(block), P(mute), P(report), P(not_dwelled) | Negative (weighted) | Ranking | Indirectly | Drag linear score down | `ranking_scorer.rs` | High |
| `in_network == false` × `effective_oon_weight` | Downweight (factor) | Ranking | Indirect (follows → in-network) | OON scores multiplied by factor | `ranking_scorer.rs` ~220–275 | High |
| Author diversity multiplier | Downweight repeats | Ranking | Indirect | Same author multiple slots in one batch | `ranking_scorer.rs` ~186–217 | High |
| VM ranker score | Overrides score if enabled | Ranking | Not supported (external) | May replace Phoenix+RankingScorer score | `vm_ranker.rs` | High when flag on |
| Block/mute / author blocks viewer | Filter | Pre-score | Mostly no (viewer) | Removed | `author_socialgraph_filter.rs` | High |
| Muted keywords | Filter | Pre-score | Partial | Token match on tweet text | `muted_keyword_filter.rs` | High |
| Tweet age | Filter | Pre-score | Yes (recency) | Old tweets dropped | `age_filter.rs`, `tweet_mixer_source.rs` | High |
| Retweet dedup | Filter | Pre-score | Structural | One card per original tweet ID | `retweet_deduplication_filter.rs` | High |
| Conversation dedup | Filter | Post-selection | Structural | One survivor per conversation | `dedup_conversation_filter.rs` | High |
| VF / ancillary VF | Filter | Post-selection | Partial | Drops by visibility | `vf_filter.rs`, `ancillary_vf_filter.rs` | High |
| Subscription-only | Filter | Pre-score | Yes (audience) | Non-subscribers dropped | `ineligible_subscription_filter.rs` | High |
| `allow_for_you_recommendations == false` | Disables OON sources | Sources | No | `in_network_only` forced | `server.rs` ~75–76 | High |
| Seen / served IDs | Filter | Pre-score | No | Re-show suppressed | `previously_seen_posts_filter.rs`, `previously_served_posts_filter.rs` | High |
| Topic / excluded / new-user topic | Filter | Pre-score | Partial | Topic taxonomy | `topic_ids_filter.rs`, `new_user_topic_ids_filter.rs` | High |

---

## 5. Practical Growth Playbook

### High-Confidence Growth Moves

**Tactic: Earn follows (in-network candidacy)**  
- **What to do:** Grow followers so Thunder includes you in `following_user_ids` for those viewers (`thunder_source.rs` ~31–38).  
- **Why it works:** Thunder is the first source; in-network avoids OON multiplier in `RankingScorer` (~272–275).  
- **Code-backed mechanism:** `ThunderSource` + `InNetworkCandidateHydrator`.  
- **Pipeline:** Candidate generation + ranking.  
- **When it will not work:** Block/mute/VF/age/seen/dedup removes you for that viewer.  
- **What not to do:** Manipulation / ToS violations — not analyzed here.  
- **Evidence:** `thunder_source.rs`, `in_network_candidate_hydrator.rs`, `ranking_scorer.rs`.  
- **Confidence:** High.

**Tactic: Stay eligible (avoid hard filters)**  
- **What to do:** Avoid visibility states that map to VF drop; avoid ancillary drop; avoid blocked/muted relationships (`author_socialgraph_filter.rs`, `vf_filter.rs`, `ancillary_vf_filter.rs`).  
- **Pipeline:** Pre-score + post-selection.  
- **Evidence:** `phoenix_candidate_pipeline.rs` filter lists.  
- **Confidence:** High.

**Tactic: Respect freshness windows**  
- **What to do:** Posts must be within `MAX_POST_AGE` from tweet ID (`AgeFilter` in pipeline ~277; TweetMixer ~75–77).  
- **Evidence:** `age_filter.rs`, `tweet_mixer_source.rs`.  
- **Confidence:** High (exact threshold in external `params`).

**Tactic: Subscription-only audience**  
- **What to do:** Subscription-gated posts only reach viewers listing the subscription author in `subscribed_user_ids` (`ineligible_subscription_filter.rs` ~15–28).  
- **Confidence:** High.

### Medium-Confidence Growth Moves

**Tactic: Align with modeled positive actions**  
- **Why:** Those probabilities are explicit terms in `compute_weighted_score` (`ranking_scorer.rs` ~146–170).  
- **When it fails:** Zero weights in FS, VM ranker overrides, or no candidacy.  
- **Confidence:** Medium for “which head matters most”; High for “heads are in the formula.”

**Tactic: Avoid patterns that raise negative heads**  
- **Why:** `not_interested`, `block`, `mute`, `report`, `not_dwelled` enter the same weighted sum (~166–170).  
- **Confidence:** Medium for operational control.

### Low-Confidence / Limited-Evidence

**Mutual-follow Jaccard** — hydrated post-selection when flags/minhash exist (`mutual_follow_jaccard_hydrator.rs` ~27–29). **Not referenced** in `RankingScorer` in this tree → effect on final score **not supported by this repository**.

---

## 6. What Works According to the Codebase

| Behavior | Why it works | Mechanism | Evidence | Practical example |
|----------|--------------|-----------|----------|-------------------|
| Being followed | Thunder + in-network scoring path | Follow graph + `in_network` | `thunder_source.rs`, `in_network_candidate_hydrator.rs`, `ranking_scorer.rs` | Followed creators enter Thunder for followers. |
| Posting within max age | Passes age / TweetMixer window | Snowflake age | `age_filter.rs`, `tweet_mixer_source.rs` | Stale posts never ranked. |
| Avoiding blocks/mutes | Socialgraph filter | Set checks | `author_socialgraph_filter.rs` | Blocked author removed for that viewer. |
| Subscriber posts to subscribers | Subscription filter | `subscription_author_id` | `ineligible_subscription_filter.rs` | Paywalled post hidden from non-subscribers. |
| One strong tweet per conversation | Dedup keeps max score | `DedupConversationFilter` | `dedup_conversation_filter.rs` | Thread: one slot after post-selection. |

---

## 7. What Does Not Work According to the Codebase

| Tactic or belief | Status | Why | Evidence | Better alternative (if supported) |
|------------------|--------|-----|----------|-----------------------------------|
| “Hashtags help” | Not supported | No hashtag signal in reviewed scorers/filters | Repo search + file review | N/A |
| “Links hurt” | Not supported | No URL filter in reviewed files | Same | N/A |
| “Blue check boosts For You” | Not supported | No verification field in `RankingScorer` / scoring `PostCandidate` | `models/candidate.rs` vs `ranking_scorer.rs` | N/A |
| “SimClusters / TwHIN / RealGraph” | Not supported | Strings absent | `rg` no matches | N/A |
| “Grox runs inside this For You path” | Not supported in home-mixer | No `grox` import under `home-mixer/` | grep | N/A |
| Many RTs of same tweet | Harmful / redundant | One survives dedup | `retweet_deduplication_filter.rs` | One canonical surface. |

**Claim checks (only where code speaks):**

- **Likes / replies / RTs / quotes “help”** — only as **predicted** heads in `RankingScorer`, not as raw public counts in that scorer (counts appear in VM ranker payload, `vm_ranker.rs` ~66–108).  
- **Video** — VQV-related weight gated by `MinVideoDurationMs` (`ranking_scorer.rs` + param key); `VideoFilter` only when client sets `exclude_videos` (`video_filter.rs`).  
- **Threads** — `DedupConversationFilter` can **remove** extra thread tweets (`dedup_conversation_filter.rs`).  
- **Dwell** — explicit terms: `dwell_score`, `dwell_time`, `click_dwell_time`, `not_dwelled` (`ranking_scorer.rs`).  
- **Freshness** — `AgeFilter` + TweetMixer age.  
- **Duplicate content** — retweet dedup + conversation dedup.  
- **Negative feedback** — supported as **model outputs** in the weighted sum; **not** proven as live event hooks in this repo.

---

## 8. Things That Can Hurt Reach (Code-Backed)

- Viewer block/mute; author blocks viewer; blocked quoted/RT’d users (`author_socialgraph_filter.rs`).  
- Muted keywords in text (`muted_keyword_filter.rs`).  
- VF / ancillary VF (`vf_filter.rs`, `ancillary_vf_filter.rs`).  
- Staleness (`age_filter.rs`, `tweet_mixer_source.rs`).  
- Seen/served suppression (`previously_seen_posts_filter.rs`, `previously_served_posts_filter.rs`).  
- Subscription mismatch (`ineligible_subscription_filter.rs`).  
- OON score multiplier (`ranking_scorer.rs`).  
- Author diversity in-batch (`ranking_scorer.rs`).  
- Recommendations disabled for viewer (`server.rs` + source `enable`).  
- Test users empty pipeline (`scored_posts_server.rs` ~45–49).

---

## 9. Creator Operating System

**Before posting:** subscription eligibility; muted-keyword risk (viewer-specific); know OON is off for viewers with `allow_for_you_recommendations == false` (`server.rs`, `phoenix_source.rs`, `tweet_mixer_source.rs`).

**While posting:** plan **one** high-score tweet per conversation if `DedupConversationFilter` applies; for video, note VQV weight gating (`ranking_scorer.rs` + `MinVideoDurationMs`).

**After posting:** serving uses aggregated sequences (`scoring_sequence_query_hydrator.rs`, `retrieval_sequence_query_hydrator.rs`) — **per-post feedback loop timing not specified in this repo.**

**Weekly review:** follows change Thunder eligibility; VF drops (opaque rules outside repo).

---

## 10. Low-Reach Diagnostic Guide

| Problem | Possible code-backed cause | How to check | What to change | Evidence |
|---------|----------------------------|--------------|----------------|----------|
| No OON | `in_network_only` / source `enable` | Viewer recommendation flag | Grow follows (Thunder) | `server.rs`, `phoenix_source.rs` |
| Zero after spikes | seen/served | `seen_ids` / `served_ids` | New IDs | `previously_seen_posts_filter.rs`, `previously_served_posts_filter.rs` |
| One thread slot | Conversation dedup | Multiple tweets same convo | Consolidate | `dedup_conversation_filter.rs` |
| RT storm dead | Retweet dedup | Same original ID | Expect one card | `retweet_deduplication_filter.rs` |

---

## 11. Creator Experiments to Run

| Experiment | Hypothesis | Code-backed reason | What to measure | Success signal | Evidence |
|------------|------------|--------------------|-----------------|------------------|----------|
| Video length | Longer → VQV weight non-zero | `vqv_weight` + `MinVideoDurationMs` | Metrics you have | Better ranking if model favors VQV | `ranking_scorer.rs` |
| Thread split | One tweet wins dedup | `DedupConversationFilter` | Per-branch impressions | Single winner | `dedup_conversation_filter.rs` |
| Follow growth | More Thunder + in-network | Thunder + `in_network` | Source / served types if logged | More in-network served | `thunder_source.rs`, `ranking_scorer.rs` |

---

## 12. Myths and Unsupported Claims

- Hashtag / link heuristics, verification badge ranking, SimClusters/TwHIN/RealGraph: **not in this repository.**  
- Grox in this For You mixer path: **not in `home-mixer`.**  
- Default weight numerics, `normalize_score`, full `build_prediction_request`: **missing `util` / params — not supported by this repository.**  
- Whether `mutual_follow_jaccard` affects ranking: **not shown** in `RankingScorer`.

---

## 13. Evidence Appendix

| File | What it proves |
|------|----------------|
| `README.md` | High-level architecture (verify against code; some scorer naming differs from `PhoenixCandidatePipeline`). |
| `home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs` | Full post-scoring pipeline wiring. |
| `home-mixer/candidate_pipeline/for_you_candidate_pipeline.rs` | For You response composition (ScoredPosts + modules). |
| `home-mixer/scorers/phoenix_scorer.rs` | Phoenix predict + gating. |
| `home-mixer/scorers/ranking_scorer.rs` | Weighted multi-head score + diversity + OON. |
| `home-mixer/scorers/vm_ranker.rs` | Optional score replacement. |
| `home-mixer/sources/*.rs` | Candidate origins. |
| `home-mixer/filters/*.rs` | Removal rules. |
| `home-mixer/server.rs` | `allow_for_you_recommendations` → `in_network_only`. |
| `phoenix/recsys_model.py` | Hash-based batch structure. |
| `phoenix/runners.py` | Action indices + sigmoid on logits. |

---

## 14. Final Practical Summary

- **Do more of this:** Build **real follows** (Thunder), stay **fresh** (age filters), pass **VF/subscription/socialgraph** rules, design **one strong tweet per conversation** when dedup applies, and align with the **multi-head predictions** fused in `RankingScorer`.  
- **Do less of this:** Duplicate RT shells, multi-tweet threads that lose in `DedupConversationFilter`, expecting reach to blocked/muted viewers or those with recommendations disabled.  
- **Ignore unless more evidence appears:** Hashtag/link/verification folklore; graph algorithms not named in repo.  
- **Watch out for this:** `VMRanker` and live feature-switch weights; missing `util` hides request shaping and normalization.

---

*Generated from codebase inspection; not official X documentation.*
