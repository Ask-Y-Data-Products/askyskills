---
name: timesheet
description: "Automate weekly timesheet submission on remote.com using Chrome MCP browser tools. Use when the user wants to fill, submit, or complete their hourly report, timesheet, time tracking, or weekly hours on remote.com. Triggers on: timesheet, hourly report, time tracking, remote.com hours, submit hours, fill timesheet."
---

# Remote.com Timesheet Automation

Fast browser automation for submitting weekly timesheets on employ.remote.com.

## PREREQUISITES

1. **Get tab context**: Call `tabs_context_mcp` with `createIfEmpty: true`.
2. **Chrome must have saved credentials** for remote.com (autofill).
3. **User must have authenticator app ready** for OTP.

---

## STEP 1: NAVIGATE & LOGIN

```
navigate → https://employ.remote.com/sign-in
wait 3s
```

The login page auto-fills credentials from Chrome. Click the Log In button using `find` tool to locate it, then click via `ref`.

**Wait 5s** for the OTP page to appear.

---

## STEP 2: OTP ENTRY

**Ask the user:** "Please give me the OTP code from your authenticator app."

The OTP page has 6 individual `input[type="tel"]` fields. Use JS to fill them with React-compatible value setting:

```javascript
// Fill OTP — remote.com uses 6 individual tel inputs
(() => {
  const otp = 'OTP_VALUE_HERE'; // Replace with actual OTP
  const inputs = document.querySelectorAll('input[type="tel"]');
  if (inputs.length < 6) return 'NO_OTP_INPUTS_FOUND';
  const nativeInputValueSetter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
  inputs.forEach((input, i) => {
    nativeInputValueSetter.call(input, otp[i]);
    input.dispatchEvent(new Event('input', { bubbles: true }));
    input.dispatchEvent(new Event('change', { bubbles: true }));
  });
  return 'filled ' + inputs.length + ' OTP digits';
})()
```

Then click "Verify code":

```javascript
(() => {
  const btn = [...document.querySelectorAll('button')].find(b => b.textContent.includes('Verify code'));
  if (btn) { btn.click(); return 'clicked Verify code'; }
  return 'NO_VERIFY_BUTTON';
})()
```

**Wait 5s** for login to complete. Check tab title — should show "Dashboard | Remote".

---

## STEP 3: NAVIGATE TO TIME TRACKING

**Important**: The URL is `https://employ.remote.com/dashboard/time-tracking/` (NOT `employ.remote.com/time-tracking` which gives Access Denied).

Option A — click sidebar link (more reliable):
```javascript
(() => {
  const links = document.querySelectorAll('a');
  for (const link of links) {
    if (link.textContent?.trim() === 'Time tracking') {
      link.click();
      return 'clicked Time tracking link';
    }
  }
  return 'NO_TIME_TRACKING_LINK';
})()
```

Option B — direct navigation:
```
navigate → https://employ.remote.com/dashboard/time-tracking/
```

**Wait 5s** for page to load.

---

## STEP 4: LOOP — FILL & SUBMIT UNFILLED WEEKS

### 4a. Check if current week needs action

```javascript
(() => {
  const body = document.body.innerText;
  const hasApproved = body.includes('APPROVED');
  const hasProcessed = body.includes('PROCESSED');
  const hasUnsubmitted = body.includes('Unsubmitted timesheet');
  const hasSubmitBtn = !!([...document.querySelectorAll('button')].find(b => b.textContent.includes('Submit timesheet')));
  const hasApplyBtn = !!([...document.querySelectorAll('button')].find(b => b.textContent.includes('Apply default schedule')));
  const hoursText = body.match(/(\d+h)\s*worked this week/)?.[1] || 'unknown';

  if (hasApproved || hasProcessed) return 'ALREADY_DONE|' + hoursText;
  if (hasUnsubmitted && hasApplyBtn) return 'NEEDS_FILLING';
  if (hasSubmitBtn && !hasApplyBtn) return 'NEEDS_SUBMIT|' + hoursText;
  if (hasSubmitBtn && hasApplyBtn) return 'NEEDS_FILLING';
  return 'UNKNOWN_STATE';
})()
```

### 4b. Apply Default Schedule

```javascript
(() => {
  const btn = [...document.querySelectorAll('button')].find(b => b.textContent.includes('Apply default schedule'));
  if (btn) { btn.click(); return 'clicked Apply default schedule'; }
  return 'NO_APPLY_DEFAULT_BUTTON';
})()
```

**Wait 3s** for the schedule to populate.

### 4c. Submit Timesheet

```javascript
(() => {
  const btn = [...document.querySelectorAll('button')].find(b => b.textContent.includes('Submit timesheet'));
  if (btn) { btn.click(); return 'clicked Submit timesheet'; }
  return 'NO_SUBMIT_BUTTON';
})()
```

**Wait 2s** — a confirmation dialog appears showing breakdown.

### 4d. Confirm Submission — click "Submit hours" in the dialog

```javascript
(() => {
  const btn = [...document.querySelectorAll('button')].find(b => b.textContent.trim() === 'Submit hours');
  if (btn) { btn.click(); return 'clicked Submit hours'; }
  return 'NO_SUBMIT_HOURS_BUTTON';
})()
```

**Wait 3s** for submission to process.

### 4e. Go to Previous Week — click the back arrow `<`

The back arrow is a `<` button to the left of the week date heading.

```javascript
(() => {
  // The back button contains a < or left arrow SVG, it's the first nav button before the date
  const buttons = [...document.querySelectorAll('button')];
  for (const btn of buttons) {
    const ariaLabel = btn.getAttribute('aria-label')?.toLowerCase() || '';
    if (ariaLabel.includes('previous') || ariaLabel.includes('back')) {
      btn.click();
      return 'clicked previous week';
    }
  }
  // Fallback: find buttons near the date heading with SVG arrows
  const svgBtns = document.querySelectorAll('button svg');
  for (const svg of svgBtns) {
    const btn = svg.closest('button');
    if (btn) {
      const siblings = btn.parentElement?.querySelectorAll('button');
      if (siblings?.length >= 2 && btn === siblings[0]) {
        btn.click();
        return 'clicked left arrow button';
      }
    }
  }
  return 'NO_PREVIOUS_WEEK_BUTTON';
})()
```

If JS fails, use `computer` tool to click coordinates `(219, 135)` — the back arrow location.

**Wait 2s** for week to load.

### 4f. Loop Logic

Repeat steps 4a-4e:
1. Check week status (4a)
2. If `ALREADY_DONE` → **STOP, all caught up**
3. If `NEEDS_FILLING` → Apply default schedule (4b), wait 3s, then Submit (4c), wait 2s, Confirm (4d), wait 3s
4. If `NEEDS_SUBMIT` → Submit (4c), wait 2s, Confirm (4d), wait 3s
5. Go to previous week (4e), wait 2s
6. Repeat from step 1

**Max iterations: 20 weeks** (safety limit).

---

## EXECUTION NOTES

- **Speed**: Use `javascript_tool` for all DOM interactions — no screenshots needed between steps.
- **Only take a screenshot** if a step returns an unexpected result (e.g., `NO_*` or `UNKNOWN_STATE`).
- **Waits**: Use the `wait` action from the computer tool.
- **If OTP fails**: The code was likely expired. Ask user for a new one immediately.
- **Key URLs**:
  - Login: `https://employ.remote.com/sign-in`
  - Dashboard: `https://employ.remote.com/dashboard`
  - Time tracking: `https://employ.remote.com/dashboard/time-tracking/`
  - Do NOT use `employ.remote.com/time-tracking` (gives Access Denied)

## QUICK REFERENCE — EXECUTION SEQUENCE

```
1. tabs_context_mcp (createIfEmpty: true)
2. navigate → https://employ.remote.com/sign-in
3. wait 3s
4. find "Log in button" → click ref
5. wait 5s
6. ASK USER FOR OTP
7. JS: fill OTP into input[type="tel"] fields using nativeInputValueSetter
8. JS: click "Verify code" button
9. wait 5s
10. JS: click "Time tracking" sidebar link (or navigate to /dashboard/time-tracking/)
11. wait 5s
12. LOOP:
    a. JS: check week status
    b. if ALREADY_DONE → done
    c. JS: apply default schedule (if needed), wait 3s
    d. JS: click "Submit timesheet", wait 2s
    e. JS: click "Submit hours" (confirm dialog), wait 3s
    f. click back arrow (coordinates 219,135 or JS), wait 2s
    g. goto (a)
```
