# n8n WhatsApp Trigger receives nothing, and everything looks correct

The workflow is published. Meta shows a green checkmark on the callback URL. The
"Send to Server" test in Meta's dashboard arrives in n8n. Then a real message
from a real phone gets two grey ticks and never reaches the workflow. No error,
no execution, nothing in the logs.

Three different faults produce that one symptom. The threads go in circles
because people try fixes for one while sitting on another. These checks separate
them in about five minutes.

**Start here, because it is the cheapest and it needs no Meta credentials.**

## Check 1: is the route registered in n8n at all?

Copy the Production URL from the WhatsApp Trigger node, then:

```bash
curl -i -X POST 'https://YOUR-INSTANCE/webhook/<uuid>/webhook' \
  -H 'Content-Type: application/json' \
  --data '{}'
```

- `Webhook call received` means the route is live. No execution is expected, the
  request has no valid Meta signature. Go to check 2.
- `404 {"message":"The requested webhook POST <uuid>/webhook is not registered."}`
  means **the fault is on the n8n side**. Meta is irrelevant until this is fixed.
  Go to fault B below, and stop reading about proxies.

Test both verbs if you want to be thorough. WhatsApp uses `GET` for verification
and `POST` for messages, and a `GET`-only 404 means something different from a
`404` on both.

## Check 2: is your WhatsApp Business Account subscribed to your app?

In Meta's Graph API Explorer, with your app selected:

```
GET /{WABA_ID}/subscribed_apps
```

That is the **WhatsApp Business Account ID**, not the phone number ID.

- Your app is listed: the link is healthy, go to check 3.
- `{"data": []}`, or a generic Meta app such as "WA DevX Webhook Events":
  **this is your fault**, see fault A.

## Check 3: does Meta's callback match the node?

```
GET /{app-id}/subscriptions
```

Compare `callback_url` against the Production URL in the node, character for
character. A different UUID means the subscription points at an older workflow
or an older webhook, see fault C.

---

## Fault A: the WABA is not linked to your app

Meta's internal link between the WhatsApp Business Account and the app holding
your webhook configuration is missing. Your number receives the message and
WhatsApp never forwards it, because as far as Meta is concerned no app is
subscribed.

**The fix is one call.** Same path as check 2, with the verb changed:

```
POST /{WABA_ID}/subscribed_apps
```

Expect `{"success": true}`. Send a real message again.

**Why this one wastes so much time:** Meta's "Send to Server" button posts
directly to your callback URL. It bypasses the WABA to app link entirely. So the
test passes while real delivery fails, and the passing test sends people off to
blame Nginx, Traefik, Cloudflare or a header-stripping proxy. The test proves
your URL is reachable. It proves nothing about whether WhatsApp will ever call
it.

## Fault B: Meta is configured, n8n never registered the route

Symptom: Meta reports `active: true` with the correct `callback_url`, and the
same URL returns `404 ... is not registered` from n8n for both `GET` and `POST`.

The thing that makes this invisible: **when a Meta subscription already exists,
n8n skips the Meta setup step on publish.** If activation fails, there is no
error to see, because the step that would have reported it never ran. The stale
subscription hides the fault.

To clear it:

1. Unpublish the workflow.
2. Delete the subscription so n8n cannot skip the setup:
   ```
   DELETE /{app-id}/subscriptions?object=whatsapp_business_account
   ```
   using an app access token, `{app-id}|{app-secret}`.
3. Publish again. n8n re-creates the subscription and Meta re-verifies the URL.
4. **Read the publish step carefully.** If it throws, that error is the real
   cause and it is the thing to report.
5. Re-run check 1.

Also worth ruling out first: only one workflow may hold a WhatsApp Trigger for a
given Meta app. A second one quietly takes the registration.

If check 1 still returns 404 after a clean republish, it is an instance-side
registration problem. On Cloud you cannot fix that yourself. Report it with the
workflow ID from the editor URL, the republish time with timezone, the exact
`curl` output, and any error shown while publishing. The webhook UUID is not the
workflow ID, and support will ask for both.

## Fault C: the callback points somewhere else

n8n believes there is a subscription, Meta disagrees, or the UUIDs differ. Same
sequence as fault B: unpublish, delete the subscription, publish, compare the
UUIDs again.

---

## The shape of this, for anyone triaging something similar

Three faults, one symptom, and the usual first move, retrying the thing that
already works, distinguishes none of them. What does:

1. **Find the check that cannot lie.** `curl` against the production URL
   needs no Meta account and splits the problem in half in one request.
2. **Distrust a passing test you did not design.** "Send to Server" skips the
   exact link that is broken, which is why it passes.
3. **A silent failure usually means a skipped step, not a working one.** The
   stale subscription did not break activation, it stopped activation from being
   attempted.

## Provenance

Compiled in October 2026 from nine n8n community threads reported between 16 and
30 September 2026. I have not reproduced fault A on my own WhatsApp Business
Account, so treat the Graph API steps as reported rather than verified by me.
Faults B and C are quoted from the request and response output posted in the
threads.

Credit where it is due:

- Fault A and the "Send to Server" insight: [Kmikzee's writeup](https://community.n8n.io/t/solucionado-whatsapp-trigger-webhooks-no-reciben-mensajes-reales-pero-el-evento-de-prueba-de-meta-si-funciona/316532)
- The `curl` check and the `GET` versus `POST` distinction: `Anshul_Namdev` in
  [this thread](https://community.n8n.io/t/whatsapp-trigger-production-webhook-is-not-registered-on-n8n-cloud/316928)
- The stale subscription masking a failed activation: `Lopez`, same thread
- [n8n Cloud reports an existing subscription, Meta returns none](https://community.n8n.io/t/n8n-cloud-whatsapp-trigger-reports-existing-webhook-subscription-but-meta-api-returns-none/316729)
- [Production webhook fails silently, manual trigger works](https://community.n8n.io/t/production-webhook-returns-generic-error-and-fails-silently-no-execution-logged-but-manual-trigger-works-perfectly/317145)

Corrections welcome as issues. If you hit a fourth variant, I would like to know.
