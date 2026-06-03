# 🚀 Web App Release Process — QA Cheatsheet

*Quick reference for the new isolated-release-branch SDLC*

## 🎯 In 30 seconds

Every API, Worker, Common Library, and Web App at Jitsu is now on a new isolated-release-branch SDLC. Each release lives in its own branch — separate from ongoing dev work. This means clearer scope, cleaner hotfixes, and better multi-team coordination.

*Branch flow: standard release (top) vs hotfix (bottom).*

## 📊 QA Workflow at Each Environment

*Environment promotion with QA gates.*

| Env | What to do | Scope |
| --- | --- | --- |
| **Staging** | Test feature + smoke test tickets released BEFORE yours on the same repo. Watch Datadog. Log a bug if FAIL. | Feature + previous tickets on same repo. Multi-team Staging — expect concurrent deploys. |
| **Beta** | Re-verify feature + smoke test previous tickets on same repo. Verify config = Prod. Smoke regression related. | Feature + previous tickets + related features. ⚠️ Check Beta availability first (workers / some apps not on Beta). |
| **Production** | Smoke check feature + smoke test previous tickets on same repo. Follow the 8 Production Rules. Cleanup test data. | Primary user flow + previous tickets on same repo. DO NOT run full regression. |
| **Hotfix** | Fast test in a pre-prod env. Smoke check Prod after deploy. | Fix + nearby regression ONLY. NO full E2E. |

> **🔁 Staging / Beta / Prod: smoke test previously-released tickets on the SAME repo**
> On EVERY environment (Staging, Beta, Prod), ALSO smoke test tickets released BEFORE yours on the same repo. This confirms the previous developer's code is still intact and your deploy didn't break earlier work. Quick happy-path check on each — not full regression.

> **⚠️ Beta limitations — not all components are deployed**
> The Beta environment does NOT have all services. Workers may not be available; some web apps may not have a Beta deployment. BEFORE testing on Beta, check whether your ticket's component exists on Beta. If not → SKIP Beta and note it clearly in the Jira comment ("Beta verification skipped — no Beta deployment for [component]").

## ✍️ Verification Comment — Required on Every Env

After testing on EACH environment (Staging, Beta, Prod), post a Jira comment with status + evidence. Evidence is MANDATORY (screenshots or video below the comment).

### Web ticket — minimum example

```
Verified on Staging — PASSED
[Attach evidence: screenshots / video]
```

### Mobile ticket — MUST include verified version

```
Verified on Staging — PASSED
App: Driver App (Jitsu Drive)
Version verified: 2.8.2 (Firebase Staging build)
[Attach evidence]
```

## ⚠️ Edge Cases — Quick Reference

*QA decision tree by release type.*

| Edge case | QA action |
| --- | --- |
| **Mid-sprint partial release** | Confirm with PM/Lead what's IN scope. Test ONLY those tickets, not other in-progress work. |
| **Hotfix** | Test fix + nearby regression. Don't run full E2E. Speed matters. |
| **Dev re-commits to release branch** | RE-VERIFY. New build = new commit hash = different code. Same test cases against the new build. |
| **Multi-team Staging deploys** | Announce the session in #qa-web. Use a `Test_` prefix. Cleanup after. Cross-check unrelated bugs. |
| **Conflict resolution on promote/* branch** | Wait for dev to resolve, then re-verify on the new build. Resolved code may differ. |

## 🆘 Where to Ask (in priority order)

| # | Resource | Best for |
| --- | --- | --- |
| **1** | **Augment / Claude Code in dev session** | RECOMMENDED. Add gojitsucom/jt-sdlc-docs as context, ask step-by-step. |
| **2** | **DevOps GPT (ChatGPT)** | Pre-trained on the SDLC. Fastest for "how do I…", "what does this workflow do", CI errors. |
| **3** | **Web Claude + GitHub connector** | Open claude.ai/new, attach gojitsucom/jt-sdlc-docs via GitHub, ask anything. |
| **4** | **Ping @devops in #devops-support** | Last resort. After trying 1–3. Mention what you tried. |

## 📚 SDLC Docs (gojitsucom/jt-sdlc-docs)

- APIs / Workers — Full Guide: `NEW-code-to-production-guide.md`
- APIs / Workers — Cheatsheet: `NEW-code-to-prod-Quick-Reference-Cheatsheet.md`
- Common Libraries — Full Guide: `NEW-common-libs-sdlc.md`
- Common Libraries — Cheatsheet: `NEW-common-libs-Quick-Reference-Cheatsheet.md`

> **💡 Don't try to memorize the docs**
> DevOps said: "Treat them as reference. Read once, enough to recognize the new commands and gates; come back to specific sections when you need them." The guides are dense by design — they cover edge cases. Use AI tools (options 1–3) to navigate them.

## ✅ DO / ❌ DON'T

| ✅ DO | ❌ DON'T |
| --- | --- |
| Include build version in Jira verification comments for traceability | Assume Staging-pass means Beta-pass — always re-verify on Beta |
| Verify config matches Prod before testing on Beta | Run full regression on Production — smoke check only |
| Re-verify after any new commit on the release branch | Expand scope on partial releases |
| Announce multi-team Staging sessions in #qa-web | Treat a hotfix like a regular ticket — speed and focus matter |
| Use AI tools (Claude Code / DevOps GPT) BEFORE pinging DevOps | Skip cleanup on Production test data |

*For full details: see "Web App Release Process — Full Guide".*

---

*Jitsu QA — Web App Release Process Cheatsheet*
