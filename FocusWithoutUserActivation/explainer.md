# `focus-without-user-activation` Permissions Policy

## The Problem

Embedded third-party content (ads, widgets, iframes) can steal input focus from the top-level page without any user interaction. This is a real security and usability problem that affects publishers, app platforms, and any site that embeds content from other origins.

### Scenario 1: Ad steals focus from a news article

A user is reading an article on a news site. An ad loaded in an iframe calls `element.focus()` on a hidden text input, silently redirecting the user's keystrokes into the ad:

```html
<!-- Publisher's page: news-site.com -->
<h1>Breaking News Article</h1>
<p>User is reading this content and starts typing a comment...</p>

<iframe src="https://ads.example.com/banner.html"></iframe>
```

```html
<!-- Ad content: ads.example.com/banner.html -->
<input id="steal" type="text" style="opacity: 0; position: absolute;">
<script>
  // As soon as the ad loads, it steals focus from the parent page.
  // The user doesn't notice but keystrokes now go to this hidden input.
  document.getElementById('steal').focus();
</script>
```

The user keeps typing, believing they're interacting with the news site, but their input is being captured by the ad. This [was originally reported](https://github.com/w3c/webappsec-permissions-policy/issues/273) by engineers working on advertising security for large online publishers.

### Scenario 2: App platform iframe steals focus while user is typing

A platform like Microsoft Teams embeds third-party apps in iframes. A user starts typing in the Teams search bar, but a third-party app iframe finishes loading and calls `autofocus` or `element.focus()`, yanking focus away mid-keystroke:

```html
<!-- Host app page (e.g. Teams) -->
<input id="search" placeholder="Search..." autofocus>

<iframe src="https://third-party-app.example.com/"></iframe>
```

```html
<!-- Third-party app: third-party-app.example.com -->
<input id="app-input" placeholder="Message..." autofocus>
<!-- This autofocus steals focus from the host's search bar -->
```

The user is mid-sentence in the search bar when the iframe loads and steals focus to its own input. Keystrokes are silently redirected. This is uncomfortable for users and from an accessibility point of view it can cause bigger issues as well if the user is moving around the app using a keyboard or is visually impaired.

### Scenario 3: Nested iframes in complex apps

In large web applications (Outlook, Teams, Office), the host page embeds an app iframe, which itself embeds further content. Any of these nested frames might call `element.focus()` on load, creating unpredictable focus jumps that confuse users:

```html
<!-- Host: outlook.com -->
<input id="compose" placeholder="Write your email...">

<iframe src="https://todo-app.example.com/">
  <!-- Inside todo-app.example.com: -->
  <!-- <input id="new-task" placeholder="Add a task" autofocus> -->
  <!-- This autofocus steals focus from the email compose field -->
</iframe>
```

## Current Workarounds and Their Limitations

Today, **developers have no clean mechanism to prevent embedded content from stealing focus**. The workarounds are fragile, incomplete, or break legitimate functionality.

### Workaround 1: Sandbox attribute

Using the `sandbox` attribute without `allow-scripts` would prevent scripted focus, but it also disables JavaScript entirely, breaking most embedded content:

```html
<!-- This blocks focus stealing, but also breaks the entire widget -->
<iframe src="https://partner-widget.example.com/" sandbox></iframe>
```

If you re-enable scripts with `sandbox="allow-scripts"`, the iframe can steal focus again. This mechanism gives no fine-grained control over focus behavior.

### Workaround 2: JavaScript focus-reclaiming hacks

Some developers use `blur` event listeners or polling to fight back against focus theft:

```js
// Fragile workaround: reclaim focus whenever an iframe steals it
const searchInput = document.getElementById('search');

window.addEventListener('blur', () => {
  // Heuristic: if we lost focus, try to take it back
  setTimeout(() => {
    if (document.activeElement === document.body) {
      searchInput.focus();
    }
  }, 100);
});
```

This approach is unreliable: it creates visible focus flickering, races with legitimate user interactions (what if the user intentionally clicked the iframe?), and doesn't scale across multiple iframes.

## Proposed Solution: `focus-without-user-activation` Permissions Policy

The `focus-without-user-activation` permissions policy gives developers a declarative way to control whether embedded content can take focus programmatically without user interaction.

When the policy is **disabled** for a document:
- `autofocus` attributes are ignored (unless the element is inserted as a result of a user gesture)
- `element.focus()` calls have no effect unless triggered by user activation
- `window.focus()` calls are similarly gated
- `dialog.showModal()` and popover focusing are also restricted

User-initiated focus (clicking into an iframe, tabbing with keyboard) is **never blocked** — this policy only restricts programmatic focus changes.

### Solving Scenario 1: Blocking ad focus theft with an HTTP header

The publisher adds an HTTP response header to deny automatic focus for the entire page and all embedded content:

```http
Permissions-Policy: focus-without-user-activation=()
```

Now the ad's `document.getElementById('steal').focus()` call silently does nothing. The user's focus stays on the article. If the user clicks on the ad, focus moves normally.

### Solving Scenario 2: Restricting a specific iframe

The host app restricts focus on the third-party app iframe specifically using the `allow` attribute:

```html
<input id="search" placeholder="Search..." autofocus>

<iframe src="https://third-party-app.example.com/"
        allow="focus-without-user-activation 'none'">
</iframe>
```

The third-party app's `autofocus` is blocked. The user's focus remains in the search bar. If the user clicks into the app, focus moves naturally.

### Allowing same-origin iframes to focus

If you embed same-origin iframes that you trust, you can allow them to take focus while blocking cross-origin ones:

```html
<!-- Same-origin widget: allowed to autofocus -->
<iframe src="/widgets/chat.html"
        allow="focus-without-user-activation 'self'">
</iframe>

<!-- Cross-origin iframe: blocked from stealing focus -->
<iframe src="https://third-party.example.com/"
        allow="focus-without-user-activation 'none'">
</iframe>
```

## Focus Delegation

A common pattern in app platforms is for the host page to explicitly pass focus to an embedded app after it loads. For example, in Microsoft Teams, when a user opens the Copilot sidebar (loaded in an iframe), Teams wants to focus the iframe, and the iframe then moves focus to its own input field.

```html
<!-- Teams host page -->
<button id="open-copilot">Open Copilot</button>
<iframe id="copilot-frame" src="https://copilot.example.com/"
        allow="focus-without-user-activation 'none'">
</iframe>

<script>
  // User clicks "Open Copilot" — Teams explicitly delegates focus to the iframe.
  // This works because the parent's focus call is triggered by user activation.
  document.getElementById('open-copilot').addEventListener('click', () => {
    document.getElementById('copilot-frame').focus();
  });
</script>
```

```html
<!-- Copilot app inside the iframe -->
<input id="prompt-input" placeholder="Message Copilot">
<script>
  // Once the iframe receives focus (delegated by the parent via user gesture),
  // it can move focus to the prompt input within itself.
  window.addEventListener('focus', () => {
    document.getElementById('prompt-input').focus();
  });
</script>
```

A similar pattern applies in Outlook when the user opens the To Do sidebar — the host delegates focus to the iframe, which then focuses the "Add a task" input.

The [TPAC 2024 resolution](https://github.com/w3c/webappsec-permissions-policy/issues/273#issuecomment-2384287101) established that **focus delegation should be allowed**, meaning a parent frame should be able to programmatically set focus into a child iframe. Once a frame has focus, it should be able to move focus within itself. The precise semantics of this behavior are still being refined (see [Open Questions](#open-questions)).

## Alternative Solutions Considered

### 1. `disallowprogrammaticfocus` boolean attribute on `<iframe>`

A new boolean attribute on `HTMLIFrameElement` was explored:

```html
<iframe src="https://ads.example.com/banner.html"
        disallowprogrammaticfocus>
</iframe>
```

The iframe would be unable to steal focus unless the user explicitly interacts with it.

**Why rejected:** This is a heavier-weight solution than a permissions policy. Sites already using permissions policies to control other behaviors (camera, microphone, geolocation) can more naturally adopt a new policy than a one-off HTML attribute. A permissions policy also allows server-side control via HTTP headers, which an HTML attribute cannot provide.

### 2. Alternative policy naming: `disallow-programmatic-focus`

An alternative name was considered:

```http
Permissions-Policy: disallow-programmatic-focus=()
```

**Why rejected:** Existing permissions policies use positive polarity — the policy name describes the *capability* being controlled, and denying the policy disables it. Using negative polarity (`disallow-*`) would be inconsistent with the rest of the permissions policy ecosystem and confuse developers familiar with the existing naming pattern.

### 3. Sandbox flag approach

Using the existing `sandbox` mechanism with a new flag:

```html
<!-- Sandboxed with a new hypothetical flag to allow focus -->
<iframe src="https://ads.example.com/banner.html"
        sandbox="allow-scripts allow-same-origin allow-focus-calls">
</iframe>
```

Or conversely, omitting the flag would block programmatic focus:

```html
<!-- Sandboxed without focus permission — focus() calls are blocked -->
<iframe src="https://ads.example.com/banner.html"
        sandbox="allow-scripts allow-same-origin">
</iframe>
```

**Why rejected:** Adding this to the sandbox would be **potentially breaking** — it would immediately affect every sandboxed frame on the web and require all sites to add the new flag to restore existing focus behavior. A permissions policy with a default allowlist of `'*'` is non-breaking: focus works everywhere by default, and sites opt in to restricting it when needed.

## Implementation Status

<!-- | Engine | Status | Link |
|--------|--------|------|
| Chromium | Shipped | [Chrome Status](https://chromestatus.com/feature/5179186249465856) |
| Gecko (Firefox) | Positive standards position, implementation in progress | [mozilla/standards-positions#1080](https://github.com/mozilla/standards-positions/issues/1080) |
| WebKit (Safari) | Positive standards position | [WebKit/standards-positions#406](https://github.com/WebKit/standards-positions/issues/406) | -->

**Specification:** Part of it has been merged into the [WHATWG HTML Standard](https://html.spec.whatwg.org/#allow-focus-steps) via [whatwg/html#10672](https://github.com/whatwg/html/pull/10672) (March 2025).

**W3C TAG Review:** [Satisfied](https://github.com/w3ctag/design-reviews/issues/1066) (July 2025).

## Open Questions

There are several active discussions refining the precise semantics of this policy:

1. **Focus delegation from parent to policy-denied child:** When a parent frame (with the policy allowed) calls `element.focus()` on an element in a child iframe that has the policy denied, should this be allowed? The current spec checks the *target's* policy, which blocks this. The proposed fix is to check the *caller's* policy instead, aligning with the TPAC 2024 resolution. See [whatwg/html#12032](https://github.com/whatwg/html/issues/12032).

2. **Descendant focus behavior:** If a frame has the policy denied but already has focus (e.g., because the parent delegated focus to it), can it move focus within itself and its child iframes? Real-world use cases (Teams embedding Copilot, Outlook embedding ToDo) depend on this working. See [whatwg/html#11519](https://github.com/whatwg/html/pull/11519) and [w3c/webappsec-permissions-policy#576](https://github.com/w3c/webappsec-permissions-policy/issues/576).

<!-- 3. **Fullscreen interaction:** If a child frame is in fullscreen and focused, should a parent frame be able to move focus away from the fullscreen element without user activation? This may warrant special handling. See discussion in [whatwg/html#11519](https://github.com/whatwg/html/pull/11519).

4. **Cross-browsing-context focus behavior:** Clarifying how the policy interacts with popups and opener relationships. See [whatwg/html#11839](https://github.com/whatwg/html/issues/11839). -->

## Appendix

### A1. Policy Pseudo-Algorithm

This describes the proposed core logic of the `allow focus steps` as specified in the [WHATWG HTML Standard](https://html.spec.whatwg.org/#allow-focus-steps):

```python
def allow_focus(focus_setter_frame, target_frame, currently_focused_frame):
    # If the frame initiating focus has the policy allowed, always permit.
    if focus_setter_frame.has_policy_allowed():
        return True

    # If the user initiated the action, always permit.
    if target_frame.has_transient_activation():
        return True

    # If the frame already has focus (or a descendant does),
    # allow it to move focus within its subtree.
    if currently_focused_frame.is_inclusive_descendant_of(focus_setter_frame):
        return True

    return False
```

An [inclusive descendant](https://html.spec.whatwg.org/#inclusive-descendant-navigables) frame is a frame that is either the same frame or a descendant frame in the frame tree hierarchy.

### A2. Spec Integration Points

The policy integrates with the following spec algorithms:
- **autofocus:** Around step 4 of the [autofocus processing](https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#attr-fe-autofocus), the algorithm returns early if the policy is disabled and the action is not [triggered by user activation](https://html.spec.whatwg.org/multipage/interaction.html#triggered-by-user-activation).
- **element.focus():** Before running the [focusing steps](https://html.spec.whatwg.org/multipage/interaction.html#focusing-steps), the same policy and user activation check is performed.
- **window.focus():** Around step 2 of the [window focus algorithm](https://html.spec.whatwg.org/multipage/interaction.html#dom-window-focus), the same enforcement applies.
- **dialog.showModal() and popover focusing:** The [dialog focusing steps](https://html.spec.whatwg.org/multipage/interactive-elements.html#the-dialog-element) and [popover focusing steps](https://html.spec.whatwg.org/multipage/popover.html#the-popover-attribute) also respect the policy.

### A3. Focus Behavior Cases

Example cases showing how focus setting works with the policy:

| Case | Policy Allowed on Setter | Setter Frame | Currently Focused Frame | Allowed? | Reason |
|------|--------------------------|--------------|-------------------------|----------|--------|
| 1 | No | Parent | Child | Yes | Focus is on a descendant of the setter — moving within subtree is allowed. |
| 2 | No | Child | Parent | No | Child cannot steal focus from parent without policy or user activation. |
| 3 | No | Grandparent | Grandchild | Yes | Focus is on a descendant of the setter. |
| 4 | No | Grandchild | Grandparent | No | Cannot steal focus upward without policy or user activation. |
| 5 | No | Same frame | Same frame | Yes | Frame already has focus, can move focus within itself. |
| 6 | Yes | Parent | Child | Yes | Policy explicitly allows. |
| 7 | Yes | Child | Parent | Yes | Policy explicitly allows. |
| 8 | Yes | Grandparent | Grandchild | Yes | Policy explicitly allows. |
| 9 | Yes | Grandchild | Grandparent | Yes | Policy explicitly allows. |
| 10 | Yes | Same frame | Same frame | Yes | Policy explicitly allows. |