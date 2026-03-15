# /probe

Run the browser-probe skill in balanced mode unless the user's wording clearly calls for smoke or bug-hunt.

The user said: "$ARGUMENTS"

If the arguments name a feature, flow, or route, keep the run scoped there. If they provide a URL, use it directly. If they provide neither, find the local dev server and probe the app there.

If the request clearly wants repro-heavy artifacts or issue handoff, use a more formal repro-first workflow instead.
