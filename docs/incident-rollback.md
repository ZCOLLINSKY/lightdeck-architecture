# Production incident rollback

Paths in backticks that are not linked live in the private repository.

**For:** Zach, alone, at 7pm, when a deploy landed and the app throws on the main path.
**Goal:** get production back to a build that worked, then put git back in agreement with it.

This is the only file in the repo that authorizes an emergency Vercel dashboard action.
Nothing else in this document overrides the ordinary release chain in
`docs/OWNER_GO_LIVE.md` ("HOW PRODUCTION RELEASES WORK"): that paragraph governs shipping
code **forward**, and it stays in force. Rolling **back** to a deployment Vercel already
built from a merged commit is a different action, and it is sanctioned here.

---

## STEP 0: Decide whether this is a rollback at all

Roll back when the current production build is broken for contractors or homeowners:
the app throws on load, a proposal link 500s, sign-in fails, money math is wrong.

**Do NOT roll back when a database migration has already been applied for this release.**
Reverting the code under an applied migration is not covered by this runbook and can
destroy data. That case escalates to `https://github.com/ZCOLLINSKY/supabase-production-patterns/blob/main/docs/backup-and-restore.md` and stops here.

Check before you touch anything:

```bash
git log --oneline origin/integrate/prod-truth -10
ls scripts/migrations/ | tail -20
```

If the release you are undoing added or applied a migration, STOP and go to
`https://github.com/ZCOLLINSKY/supabase-production-patterns/blob/main/docs/backup-and-restore.md`.

---

## STEP 1: Find the previous-good SHA

The production deployment history is the source of truth for what was live before.
This is the same call `scripts/os-health.mjs` already makes for its "repo == prod" check.

```bash
gh api 'repos/ZCOLLINSKY/lighting-os/deployments?environment=Production&per_page=10' \
  --jq '.[] | "\(.sha[0:40])  \(.created_at)  \(.description // "")"'
```

The first row is the bad build now live. The **previous-good SHA** is the next row down
that you know was healthy. Write both down:

- `BAD_SHA` = <40-char sha now live>
- `GOOD_SHA` = <40-char sha you are rolling back to>

Confirm what is live right now:

```bash
curl -s https://www.lightdeck.tech/api/release-info | head -c 400
```

`gitSha` in that response must equal `BAD_SHA`. If it does not, you are looking at the
wrong deployment; re-read the list before doing anything else.

---

## STEP 2: emergency exception, Vercel Instant Rollback

**This is sanctioned.** Vercel dashboard → the `lighting-os` project → Deployments →
find the deployment whose commit is `GOOD_SHA` → the `...` menu → **Instant Rollback**
(older projects label the same control **Promote to Production**).

Why this is allowed when `docs/OWNER_GO_LIVE.md` forbids a "dashboard redeploy": that
rule exists to stop *unproven code* reaching production without the gate, and to stop
the split-brain that the retired `deploy.sh` caused by pushing an uncommitted local copy
with `vercel --prod`. Instant Rollback does neither. It re-points the production alias at
an immutable build Vercel already produced from a merged, gate-passed commit. No new
bytes are created and nothing uncommitted can ride along.

The exception is bounded by STEP 4. It is not permission to run the app off a
dashboard-chosen build indefinitely.

The git-gated path (branch → revert → green `gate` on the exact head SHA → merge commit →
Vercel build → verify) is still the *correct* path and is what STEP 4 does. It just
cannot complete in ten minutes, which is why STEP 2 exists.

---

## STEP 3: Verify the rollback landed (the verification triple)

All three, in order. Two out of three is not a verified rollback.

1. **Release info reports the good SHA.**

   ```bash
   curl -s https://www.lightdeck.tech/api/release-info
   ```

   `gitSha` must now equal `GOOD_SHA`.

2. **A deployment receipt exists for that SHA.** The Vercel deployment you promoted must
   show as the current Production deployment, and:

   ```bash
   gh api 'repos/ZCOLLINSKY/lighting-os/deployments?environment=Production&per_page=1' \
     --jq '.[0].sha'
   ```

3. **Live smoke passes against the exact SHA.**

   ```bash
   npm run smoke:live -- --expect-sha <GOOD_SHA>
   ```

   A pass here is the proof. Do not tell anyone production is fixed before this exits 0.

---

## STEP 4: MANDATORY: open the revert PR the same night

The dashboard rollback moved production. It did **not** move git. Until this step is
done, `integrate/prod-truth` still carries the broken commit, and the very next merge
re-deploys the failure on top of your rollback.

This is not optional and it is not a next-morning task. Same night:

```bash
git fetch origin
git checkout -b revert/<BAD_SHA_SHORT>-incident origin/integrate/prod-truth
git revert --no-edit <BAD_SHA>          # use -m 1 if BAD_SHA is a merge commit
git push -u origin revert/<BAD_SHA_SHORT>-incident
gh pr create --base integrate/prod-truth --title "revert: <what broke>" --body "..."
```

Then the ordinary rules apply: the required `gate` check must be green on the exact head
SHA, the merge is a real merge commit and never a squash, and the deploy that follows is
verified with the same triple in STEP 3.

Git and production reconverge when that merge deploys. Only then is the incident closed.

---

## STEP 5: Write down what happened

Append one paragraph to `docs/OWNER_BLOCKERS.md`: what broke, `BAD_SHA`, `GOOD_SHA`,
when the rollback landed, when the revert PR merged, and what test would have caught it.
The last field is the only one that prevents a repeat.

---

## Quick reference

| Situation | Action |
|---|---|
| Broken deploy, no migration applied | STEP 2 Instant Rollback, then STEP 4 revert PR the same night |
| Broken deploy, migration applied | STOP: `https://github.com/ZCOLLINSKY/supabase-production-patterns/blob/main/docs/backup-and-restore.md` |
| Not sure whether a migration applied | Treat it as applied. STOP. |
| Rollback done, revert PR not yet merged | The incident is OPEN. Do not merge anything else to `integrate/prod-truth`. |
