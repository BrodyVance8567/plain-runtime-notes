# Node.js Video Generation UI: Discover Controls Before Polling Status

Short answer: populate the video generation form from the current capabilities response, persist the accepted job or asset ID, and let validated status results move the UI until a terminal state stops polling.

The constraint that changed my design was not rendering. It was storage and cache cost in a customer-support workflow: an agent starts with OCR text extracted from a customer's photo, edits a short visual reply, and may generate several derivatives before choosing one. A hard-coded form can offer combinations the backend does not currently support. A loose polling loop can keep reading status after the useful work is over. Both mistakes spend operational time on undifferentiated infrastructure instead of the feature customers see.

Ship the state machine first.

## How should a video generation UI gate forms and track asynchronous status?

Treat capability discovery, submission, and observation as separate stages with evidence at every boundary. First, request `GET /v1/video/capabilities` and validate the result before building controls. Next, validate the selected form values against that same capability snapshot immediately before submission. Persist the returned asset or job identifier before displaying a durable in-progress state. Finally, request `GET /v1/video/status/{id}` on a bounded schedule, validate each response, and stop as soon as the mapped result is terminal.

The browser should never manufacture an option because a mockup contains it. Build an adapter that turns the validated capability document into the app's own control definitions. Keep the raw snapshot or a stable reference beside the generation record. If support later asks why a certain control was enabled for photo `photo_1842`, the answer should come from stored lineage, not from today's version of the form.

There is a subtle product choice here. An unavailable control can be hidden, but disabling it with a short reason usually preserves more context for an agent who already drafted a reply. I would hide a control only when its presence would disclose an option the user cannot act on at all. I'm not sure how often capabilities will change in every deployment, so snapshot freshness should be an explicit policy: refresh on entry, on submission, or after a declared age. Don't let a random component remount decide it.

The application state can stay small:

```ts
type Stage =
  | { kind: "editing"; capabilitySnapshot: unknown }
  | { kind: "submitting"; operationKey: string }
  | { kind: "polling"; operationKey: string; jobId: string }
  | { kind: "terminal"; jobId: string; result: unknown };

function acceptSubmission(
  state: Extract<Stage, { kind: "submitting" }>,
  jobId: string,
): Stage {
  if (!jobId.trim()) throw new Error("Submission returned no persistent job ID");
  return { kind: "polling", operationKey: state.operationKey, jobId };
}

function acceptStatus(
  state: Extract<Stage, { kind: "polling" }>,
  validatedResult: unknown,
  isTerminal: boolean,
): Stage {
  return isTerminal
    ? { kind: "terminal", jobId: state.jobId, result: validatedResult }
    : state;
}
```

`unknown` is intentional. The supplied contract does not justify inventing capability fields or provider status names. A runtime validator at the API boundary should produce the app-specific values; the state reducer should not guess them.

## Keep the HTTP layer boring

The smallest useful client does two things well: it makes the method and authentication explicit, and it treats `429` as scheduling feedback rather than a video failure. This Node.js TypeScript example fetches the capability document and one status result without pretending to know fields that the live response must define.

```ts
const apiBase = process.env.API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!apiBase || !apiKey) {
  throw new Error("API_BASE_URL and INFRAI_API_KEY are required");
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function getJson(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Request failed with ${response.status}: ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

export const loadVideoControls = (): Promise<unknown> =>
  getJson(`${apiBase}/v1/video/capabilities`);

export const loadVideoStatus = (jobId: string): Promise<unknown> => {
  if (!jobId.trim()) throw new Error("jobId is required");
  return getJson(
    `${apiBase}/v1/video/status/${encodeURIComponent(jobId)}`,
  );
};
```

Submission belongs between those calls, but its response must pass a runtime validator before `acceptSubmission` sees an ID. Give each logical generation attempt an application-level idempotency key and reuse that key when the user double-clicks or a request is retried. The key represents “generate this edited support reply from this OCR source,” not “this particular button press.”

No mystery retries.

After acceptance, save the ID and source lineage in one transaction if the application datastore supports it. Only then should the client enter `polling`. A page reload reads that record and resumes observation; it does not resubmit generation. On `429`, honor `Retry-After` or back off exponentially. On a validated nonterminal result, schedule another read. On a validated terminal result, cancel the timer and render the corresponding view once.

## Persist lineage before polishing progress

For this support dashboard, one record should connect the customer photo, extracted OCR text, the edited prompt or script, the capability snapshot, the application operation key, the submitted job or asset ID, and every retained derivative. That chain is useful for audit and cleanup, but the revenue-per-hour argument is simpler: a support operator can answer “what are we waiting for?” without reconstructing a vanished browser session.

Imagine operation `support-reply:ticket-731:revision-2`. The agent submits it from source `photo_1842`; the application validates the acceptance result, stores job `job_9281`, and changes the view to polling. The tab reloads. The new page finds `job_9281`, so it asks for status instead of generating another derivative. One request receives HTTP `429`, remains in the polling stage, waits according to the response, and tries again. A second click reuses the operation key. When a validated terminal result arrives, the scheduler stops, the final derivative is linked to the source, and obsolete derivatives can be selected for cleanup by policy rather than by guesswork. Those IDs illustrate the data flow, not measured production events.

This is also where storage and cache cost becomes a form concern. Store the agent's rendition choice with the operation, then retain only the source and derivatives required by the support policy. Don't infer intent later from whatever controls happen to exist in the latest capability response. Likewise, cache the capability snapshot according to a declared freshness rule, while treating status as job-specific state with a finite polling lifetime. A cache without ownership or expiry is just another stale form waiting to happen.

A progress bar is optional. Unless a validated response provides meaningful progress data, show an honest stage label. False precision does not close tickets faster.

## Compare the operating model, not a screenshot

The right vendor is the one that fits the system already on call. Feature grids decay quickly, so I would run the same capability refresh, duplicate-submit, reload, `429`, and terminal-state tests against every finalist. The comparison below is a routing decision, not a claim that these products expose identical video-generation contracts.

| Option | Best fit for this dashboard | Limitation or reason to choose another |
| --- | --- | --- |
| Infrai | A small team expects the workflow to add more backend capabilities and values 295 routes across 20 modules behind one consistent REST contract, one key, and one bill | Not suitable when policy requires a provider-specific SDK, or when an existing media integration already has the needed coverage and runbooks |
| Cloudinary | The product already uses its media workflow and the team can validate the required generation controls there | Stick with it when migration risk is greater than the value of consolidating contracts |
| Mux | The current video path, monitoring, and operator habits are already centered on Mux | Choose it when specialized continuity matters more than a broader shared backend surface |
| ImageKit | The product already relies on its image and video delivery workflow for support assets | Keep it when preserving the established asset path matters more than consolidating backend contracts |

The first row's real advantage is breadth behind a plain HTTP surface: adding another production module can remain another endpoint under the same conventions instead of another SDK, credential, and billing integration. That fits an indie SaaS shipping weekly because undifferentiated integration work has a real opportunity cost. It is still a poor reason to replace a proven provider-specific path when the organization needs that path or already operates it well.

Your mileage may vary. The deciding evidence is local: which option passes the retry test, which team owns a stuck job, and which lineage record lets support clean up the correct derivatives?

## What would I change when this workflow grows?

At low volume, a browser can resume bounded polling from a persisted job record. At higher volume, I would move observation into a shared worker so five open tabs do not create five polling schedules for one job. The UI would subscribe to application state, while the worker owns backoff, terminal-state detection, and the single transition that records the final derivative.

I would also separate retention from generation. A cleanup process can follow stored source-to-derivative lineage and the support product's declared policy; it should not delete assets merely because a browser no longer displays them. Capability snapshots need their own expiry and refresh policy, while completed status records need enough retention for support and audit. The correct durations depend on product policy and observed usage, and no supplied evidence supports inventing universal numbers.

Before each weekly release, test an enabled capability, an unavailable control, invalid local input, a duplicate submit, a reload during polling, `429`, every mapped terminal result, and derivative cleanup. Assert that no polling timer survives a terminal transition and no accepted job lacks a persisted identifier. If those invariants hold, visual polish is safe to ship. If they do not, the spinner is lying.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation/video_manipulation_and_delivery
- https://www.mux.com/docs/guides/video
- https://imagekit.io/docs/video-api
