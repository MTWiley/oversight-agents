# WAI-ARIA Patterns Reference Checklist

Comprehensive reference for evaluating ARIA usage and widget patterns against the
WAI-ARIA Authoring Practices. Complements the inlined criteria in
`review-accessibility.md` with detailed role/state/property requirements, keyboard
interaction models, focus management rules, common mistakes, and framework-specific
code examples.

---

## Foundational ARIA Rules

Before applying ARIA, understand these governing principles. Misused ARIA is worse
than no ARIA at all -- it creates a false accessibility layer that misleads assistive
technology users.

### First Rule: Prefer Native HTML

If a native HTML element or attribute provides the semantics and behavior you need,
use it instead of adding ARIA. Native elements come with built-in keyboard handling,
focus management, and assistive technology support.

| Instead of | Use |
|-----------|-----|
| `<div role="button" tabindex="0">` | `<button>` |
| `<span role="link">` | `<a href="...">` |
| `<div role="checkbox" aria-checked="true">` | `<input type="checkbox" checked>` |
| `<div role="navigation">` | `<nav>` |
| `<div role="main">` | `<main>` |
| `<div role="heading" aria-level="2">` | `<h2>` |
| `<div role="list"><div role="listitem">` | `<ul><li>` |
| `<span role="img" aria-label="...">` | `<img alt="...">` |
| `<div role="progressbar">` | `<progress>` |
| `<div role="textbox" contenteditable>` | `<input>` or `<textarea>` |

**Detection pattern**:
```regex
# ARIA roles that duplicate native HTML semantics
role\s*=\s*["'](button|link|checkbox|radio|textbox|heading|navigation|main|banner|contentinfo|complementary|list|listitem|img|progressbar|slider)["']
```

**Severity**: MEDIUM when a native element would work identically.

### Second Rule: Do Not Change Native Semantics

Do not change the native semantics of an element unless absolutely necessary. Adding
`role="button"` to a heading, for example, removes the heading semantics.

```html
<!-- Violation: heading semantics overridden -->
<h2 role="button" tabindex="0" onclick="toggle()">Section Title</h2>

<!-- Fix: wrap a button inside the heading -->
<h2><button onclick="toggle()">Section Title</button></h2>
```

**Severity**: HIGH

### Third Rule: All Interactive ARIA Elements Must Be Keyboard Operable

If you give an element an interactive role (`button`, `link`, `tab`, `menuitem`,
etc.), you must ensure it can be operated with the keyboard.

```html
<!-- Violation: role="button" but no keyboard handling -->
<div role="button" onclick="submit()">Submit</div>

<!-- Fix: add tabindex and keyboard handler (or just use <button>) -->
<div role="button" tabindex="0" onclick="submit()"
     onkeydown="if(event.key==='Enter'||event.key===' ')submit()">
  Submit
</div>
```

**Severity**: CRITICAL

### Fourth Rule: Do Not Hide Focusable Elements with aria-hidden

Never use `aria-hidden="true"` on an element that can receive focus or that
contains focusable descendants.

```html
<!-- Violation: focusable element hidden from AT -->
<div aria-hidden="true">
  <button>Close</button>  <!-- Sighted users see it, AT users cannot -->
</div>

<!-- Fix: either remove aria-hidden or make children inert -->
<div aria-hidden="true" inert>
  <button>Close</button>
</div>
<!-- Or remove aria-hidden if the button should be accessible -->
```

**Detection pattern**:
```regex
aria-hidden\s*=\s*["']true["'][^>]*>[\s\S]*?<(button|a\s|input|select|textarea|[^>]*tabindex)
```

**Severity**: CRITICAL

### Fifth Rule: All Interactive Elements Must Have an Accessible Name

Every interactive element must have a name that assistive technology can present
to users. This can come from content, `aria-label`, `aria-labelledby`, `<label>`,
`alt`, or `title`.

**Detection pattern**:
```regex
# Buttons/links without text content or accessible name
<(button|a)\s[^>]*>\s*<(svg|img|i|span)\s[^>]*/>\s*</(button|a)>
# Check: does the element have aria-label, aria-labelledby, or visible text?
```

**Severity**: CRITICAL

---

## Naming and Describing

### aria-label

Provides an accessible name as a string. Use when there is no visible text label
for the element.

```html
<!-- Icon button: aria-label provides the name -->
<button aria-label="Close" onclick="close()">
  <svg aria-hidden="true">...</svg>
</button>

<!-- Search input with visible label elsewhere handled differently -->
<input type="search" aria-label="Search products" />
```

**Rules**:
- Use only when no visible label exists or can be referenced via `aria-labelledby`.
- The value must include the visible text if any is present (SC 2.5.3 Label in Name).
- Do not use on elements that do not support naming (e.g., `<div>` without a role, `<span>` without a role).

**Common mistakes**:
```html
<!-- Mistake: aria-label on non-interactive element without role -->
<div aria-label="Sidebar">...</div>
<!-- Fix: add role or use semantic element -->
<aside aria-label="Sidebar">...</aside>
<!-- Or -->
<div role="complementary" aria-label="Sidebar">...</div>

<!-- Mistake: aria-label doesn't match visible text -->
<button aria-label="Submit form data">Send</button>
<!-- Fix: include visible text -->
<button aria-label="Send">Send</button>
```

**Severity**: MEDIUM (incorrect usage) / CRITICAL (missing name on interactive element)

### aria-labelledby

References other elements by ID to compose an accessible name. Takes precedence
over `aria-label` and native labeling mechanisms.

```html
<!-- Multiple elements compose the label -->
<h2 id="dialog-title">Delete Account</h2>
<p id="dialog-desc">This action cannot be undone.</p>
<div role="dialog" aria-labelledby="dialog-title" aria-describedby="dialog-desc">
  ...
</div>

<!-- Concatenating multiple label sources -->
<span id="billing">Billing</span>
<span id="name">Name</span>
<input aria-labelledby="billing name" />
<!-- AT reads: "Billing Name" -->
```

**Rules**:
- Can reference multiple IDs separated by spaces (names are concatenated).
- Referenced IDs must exist in the same document.
- Takes precedence over `aria-label`, `<label>`, and element content for naming.

**Common mistakes**:
```html
<!-- Mistake: referencing non-existent ID -->
<button aria-labelledby="btn-label">Click</button>
<!-- No element with id="btn-label" exists -->

<!-- Mistake: referencing hidden element that doesn't contribute useful text -->
<span id="lbl" style="display:none"></span>
<input aria-labelledby="lbl" />
```

**Severity**: HIGH (broken references)

### aria-describedby

Links an element to descriptions that provide additional context. Unlike
`aria-labelledby`, the description is announced after the name and role.

```html
<!-- Help text for a form field -->
<label for="password">Password</label>
<input type="password" id="password" aria-describedby="pwd-help pwd-error" />
<span id="pwd-help">Must be at least 8 characters with a number.</span>
<span id="pwd-error" class="error">Password is too short.</span>
```

**Rules**:
- Use for supplementary information, not the primary name.
- Can reference hidden elements (`display:none` content is still read).
- Multiple IDs separated by spaces are concatenated.

**Common mistakes**:
```html
<!-- Mistake: using aria-describedby as the primary label -->
<input type="email" aria-describedby="email-label" />
<label id="email-label">Email</label>
<!-- Fix: use aria-labelledby or for/id association for the label -->
```

**Severity**: MEDIUM

---

## Hiding Mechanisms

Understanding the differences between hiding approaches is critical for proper
ARIA implementation.

| Mechanism | Visible | In Accessibility Tree | Focusable |
|-----------|---------|----------------------|-----------|
| `display: none` | No | No | No |
| `visibility: hidden` | No | No | No |
| `aria-hidden="true"` | Yes | No | Yes (danger!) |
| `role="presentation"` / `role="none"` | Yes | Semantics removed | Depends |
| `.sr-only` (visually hidden) | No | Yes | Yes |
| `hidden` attribute | No | No | No |
| `inert` attribute | Yes | No | No |
| `clip-path / clip` (off-screen) | No | Yes | Yes |

### aria-hidden="true"

Removes the element and its children from the accessibility tree while keeping
them visible. Used for decorative or redundant visual content.

```html
<!-- Correct: decorative icon next to text -->
<button>
  <svg aria-hidden="true"><!-- icon --></svg>
  Delete
</button>

<!-- Correct: visual decoration -->
<div aria-hidden="true" class="background-pattern"></div>

<!-- Violation: hiding content that has no visible alternative -->
<div aria-hidden="true">
  <p>Important instructions for completing the form.</p>
</div>
```

### role="presentation" / role="none"

Removes the semantic meaning of an element but does not hide its content. Used
when the element is used purely for layout.

```html
<!-- Layout table: remove table semantics -->
<table role="presentation">
  <tr><td>Column 1</td><td>Column 2</td></tr>
</table>

<!-- Note: role="none" is a synonym for role="presentation" -->
```

**Important**: `role="presentation"` on an element with required children (like
`<table>` requiring `<tr>`) removes semantics from the required children as well.
It does NOT remove semantics from interactive children like links or buttons.

### Visually Hidden (Screen Reader Only)

Content that should be in the accessibility tree but not visible on screen.

```css
/* Standard visually-hidden class */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

/* Allow focus for skip links */
.sr-only-focusable:focus {
  position: static;
  width: auto;
  height: auto;
  margin: 0;
  overflow: visible;
  clip: auto;
  white-space: normal;
}
```

**Severity**: CRITICAL (aria-hidden on focusable elements) / HIGH (incorrect hiding mechanism choice)

---

## Live Regions

Live regions announce dynamic content changes to screen reader users without
moving focus. They are essential for real-time updates, notifications, and
asynchronous operations.

### aria-live

Defines a live region whose content changes should be announced.

| Value | Behavior | Use For |
|-------|----------|---------|
| `off` | No announcements (default) | Content that updates but should not interrupt |
| `polite` | Announces at next pause in speech | Search results count, saved confirmation, chat messages |
| `assertive` | Interrupts current speech immediately | Error messages, urgent alerts, time-critical warnings |

### aria-atomic

Determines whether the entire live region is announced or only the changed portion.

| Value | Behavior | Use For |
|-------|----------|---------|
| `true` | Entire region re-announced | Short status: "3 items in cart", timestamps |
| `false` (default) | Only changed nodes announced | Chat log, activity feed |

### aria-relevant

Specifies which changes trigger announcements.

| Value | Triggers On |
|-------|------------|
| `additions` | New nodes added |
| `removals` | Nodes removed |
| `text` | Text content changed |
| `all` | All changes |
| `additions text` (default) | New nodes or text changes |

### Live Region Patterns

**Status message (polite)**:
```html
<div role="status" aria-live="polite" aria-atomic="true">
  <!-- Content updated dynamically -->
  3 results found
</div>
```

**Error alert (assertive)**:
```html
<div role="alert" aria-live="assertive">
  <!-- role="alert" implies aria-live="assertive" and aria-atomic="true" -->
  Session expired. Please log in again.
</div>
```

**Progress indicator**:
```html
<div aria-live="polite" aria-atomic="true">
  Uploading: 45% complete
</div>
```

**Chat/Log (additions only)**:
```html
<div aria-live="polite" aria-atomic="false" aria-relevant="additions">
  <p>User A: Hello</p>
  <p>User B: Hi there</p>
  <!-- New messages announced individually -->
</div>
```

### Common Live Region Mistakes

```html
<!-- Mistake: live region added to DOM already containing content -->
<!-- The content on initial render is NOT announced -->
<div aria-live="polite">Already here</div>

<!-- Fix: add content after the live region is in the DOM -->
<div aria-live="polite"></div>
<!-- Then dynamically: liveRegion.textContent = "Update" -->
```

```jsx
// Mistake: conditionally rendering the live region itself
{showMessage && <div role="status">Saved!</div>}
// The region appears and disappears -- AT may not catch it

// Fix: keep the region in DOM, change its content
<div role="status" aria-live="polite">
  {showMessage ? 'Saved!' : ''}
</div>
```

```vue
<!-- Mistake: using aria-live="assertive" for non-urgent info -->
<div aria-live="assertive">{{ resultsCount }} results found</div>

<!-- Fix: use "polite" for non-urgent updates -->
<div aria-live="polite" aria-atomic="true">{{ resultsCount }} results found</div>
```

**Severity**: HIGH (missing live region on critical updates) / MEDIUM (wrong politeness level, incorrect atomic setting)

---

## Component Patterns

Each pattern below follows the WAI-ARIA Authoring Practices specification. For
each component: required ARIA roles/states/properties, keyboard interaction model,
focus management rules, common mistakes, and code examples.

---

### Modal Dialog

A window overlaid on the primary content. When open, interaction with content
outside the dialog is blocked.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="dialog"` | Dialog container | Required |
| `aria-modal="true"` | Dialog container | Required |
| `aria-labelledby` | Dialog container | ID of the dialog title |
| `aria-describedby` | Dialog container | ID of dialog description (optional) |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Tab | Move focus to next focusable element inside dialog; wrap from last to first |
| Shift+Tab | Move focus to previous focusable element; wrap from first to last |
| Escape | Close the dialog |

#### Focus Management

1. When dialog opens: move focus to the first focusable element, or the dialog
   itself if it has `tabindex="-1"`, or a specific element like the close button.
2. Trap focus inside the dialog (Tab cycles within dialog boundaries).
3. When dialog closes: return focus to the element that triggered the dialog.

#### Common Mistakes

```html
<!-- Mistake 1: no focus trap -->
<div role="dialog" aria-modal="true">
  <button>OK</button>
</div>
<!-- User can Tab out of the dialog to background content -->

<!-- Mistake 2: focus not returned to trigger -->
<!-- Dialog closes but focus goes to <body> or top of page -->

<!-- Mistake 3: background not inert -->
<!-- Screen readers can still navigate to content behind the dialog -->

<!-- Mistake 4: missing aria-modal -->
<div role="dialog">
  <!-- Without aria-modal, AT may still read background content -->
</div>
```

#### Code Examples

**HTML + JavaScript**:
```html
<button id="open-dialog" onclick="openDialog()">Delete Account</button>

<div id="confirm-dialog" role="dialog" aria-modal="true"
     aria-labelledby="dialog-title" aria-describedby="dialog-desc"
     hidden>
  <h2 id="dialog-title">Delete Account</h2>
  <p id="dialog-desc">This will permanently delete your account and all data.</p>
  <button id="confirm-btn" onclick="confirmDelete()">Delete</button>
  <button id="cancel-btn" onclick="closeDialog()">Cancel</button>
</div>
```

```javascript
let triggerElement = null;

function openDialog() {
  triggerElement = document.activeElement;
  const dialog = document.getElementById('confirm-dialog');
  dialog.hidden = false;
  dialog.querySelector('#cancel-btn').focus();
  trapFocus(dialog);
}

function closeDialog() {
  const dialog = document.getElementById('confirm-dialog');
  dialog.hidden = true;
  if (triggerElement) triggerElement.focus();
}

function trapFocus(element) {
  const focusable = element.querySelectorAll(
    'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
  );
  const first = focusable[0];
  const last = focusable[focusable.length - 1];

  element.addEventListener('keydown', (e) => {
    if (e.key === 'Escape') { closeDialog(); return; }
    if (e.key !== 'Tab') return;

    if (e.shiftKey) {
      if (document.activeElement === first) { last.focus(); e.preventDefault(); }
    } else {
      if (document.activeElement === last) { first.focus(); e.preventDefault(); }
    }
  });
}
```

**React/JSX**:
```jsx
import { useRef, useEffect, useCallback } from 'react';

function ConfirmDialog({ isOpen, onClose, onConfirm, title, description }) {
  const dialogRef = useRef(null);
  const triggerRef = useRef(null);

  useEffect(() => {
    if (isOpen) {
      triggerRef.current = document.activeElement;
      // Focus first focusable element in dialog
      const firstFocusable = dialogRef.current?.querySelector('button, [href], input');
      firstFocusable?.focus();
    } else if (triggerRef.current) {
      triggerRef.current.focus();
    }
  }, [isOpen]);

  const handleKeyDown = useCallback((e) => {
    if (e.key === 'Escape') { onClose(); return; }
    if (e.key !== 'Tab') return;

    const focusable = dialogRef.current.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    const first = focusable[0];
    const last = focusable[focusable.length - 1];

    if (e.shiftKey && document.activeElement === first) {
      last.focus(); e.preventDefault();
    } else if (!e.shiftKey && document.activeElement === last) {
      first.focus(); e.preventDefault();
    }
  }, [onClose]);

  if (!isOpen) return null;

  return (
    <>
      <div className="dialog-backdrop" onClick={onClose} />
      <div
        ref={dialogRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby="dialog-title"
        aria-describedby="dialog-desc"
        onKeyDown={handleKeyDown}
      >
        <h2 id="dialog-title">{title}</h2>
        <p id="dialog-desc">{description}</p>
        <button onClick={onConfirm}>Delete</button>
        <button onClick={onClose}>Cancel</button>
      </div>
    </>
  );
}
```

#### Severity: **CRITICAL** (missing focus trap, focus not returned) / **HIGH** (missing aria-modal, no Escape handler)

---

### Tabs

A set of layered panels where only one panel is displayed at a time, selected
by its corresponding tab.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="tablist"` | Container of tabs | Required |
| `role="tab"` | Each tab button | Required |
| `role="tabpanel"` | Each panel | Required |
| `aria-selected="true"` | Active tab | Required |
| `aria-selected="false"` | Inactive tabs | Required |
| `aria-controls` | Each tab | ID of its associated tabpanel |
| `aria-labelledby` | Each tabpanel | ID of its associated tab |
| `tabindex="0"` | Active tab | One tab in the tablist is focusable |
| `tabindex="-1"` | Inactive tabs | Removed from tab order |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Arrow Right | Move focus to next tab (wraps to first) |
| Arrow Left | Move focus to previous tab (wraps to last) |
| Home | Move focus to first tab |
| End | Move focus to last tab |
| Space/Enter | Activate focused tab (if not auto-activation) |
| Tab | Move focus into the active tabpanel |

**Activation model**: Tabs can use automatic activation (panel changes on arrow
key focus) or manual activation (panel changes only on Space/Enter). Automatic
activation is preferred when panel content loads quickly.

#### Focus Management

- Only one tab in the tablist is in the Tab order (`tabindex="0"`).
- Arrow keys move focus between tabs within the tablist.
- Tab key moves focus from the tablist into the active panel.

#### Common Mistakes

```html
<!-- Mistake 1: all tabs in tab order -->
<div role="tablist">
  <button role="tab" tabindex="0">Tab 1</button>
  <button role="tab" tabindex="0">Tab 2</button>  <!-- Should be -1 -->
  <button role="tab" tabindex="0">Tab 3</button>  <!-- Should be -1 -->
</div>

<!-- Mistake 2: using links instead of buttons for tabs -->
<div role="tablist">
  <a role="tab" href="#panel1">Tab 1</a>  <!-- Links navigate; tabs switch -->
</div>

<!-- Mistake 3: missing aria-selected -->
<button role="tab" class="active">Tab 1</button>
<!-- Visual "active" class but no aria-selected -->

<!-- Mistake 4: panels not associated with tabs -->
<div role="tabpanel">Content 1</div>
<!-- Missing aria-labelledby pointing to its tab -->
```

#### Code Examples

**React/JSX**:
```jsx
function Tabs({ tabs }) {
  const [activeIndex, setActiveIndex] = useState(0);

  const handleKeyDown = (e, index) => {
    let newIndex = index;
    if (e.key === 'ArrowRight') newIndex = (index + 1) % tabs.length;
    else if (e.key === 'ArrowLeft') newIndex = (index - 1 + tabs.length) % tabs.length;
    else if (e.key === 'Home') newIndex = 0;
    else if (e.key === 'End') newIndex = tabs.length - 1;
    else return;

    e.preventDefault();
    setActiveIndex(newIndex);
    document.getElementById(`tab-${newIndex}`).focus();
  };

  return (
    <div>
      <div role="tablist" aria-label="Feature tabs">
        {tabs.map((tab, i) => (
          <button
            key={tab.id}
            id={`tab-${i}`}
            role="tab"
            aria-selected={i === activeIndex}
            aria-controls={`panel-${i}`}
            tabIndex={i === activeIndex ? 0 : -1}
            onKeyDown={(e) => handleKeyDown(e, i)}
            onClick={() => setActiveIndex(i)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {tabs.map((tab, i) => (
        <div
          key={tab.id}
          id={`panel-${i}`}
          role="tabpanel"
          aria-labelledby={`tab-${i}`}
          hidden={i !== activeIndex}
          tabIndex={0}
        >
          {tab.content}
        </div>
      ))}
    </div>
  );
}
```

#### Severity: **HIGH** (missing roles/states, no keyboard navigation) / **MEDIUM** (incorrect tabindex management)

---

### Menu and Menubar

A widget that offers a list of choices, like an application menu. Not for
navigation links (use `<nav>` with a list for navigation).

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="menu"` | Menu container | Required |
| `role="menubar"` | Horizontal menu bar | If applicable |
| `role="menuitem"` | Each item | Required |
| `role="menuitemcheckbox"` | Checkable item | When applicable |
| `role="menuitemradio"` | Radio-select item | When applicable |
| `aria-haspopup="true"` | Item that opens submenu | Required |
| `aria-expanded` | Item with submenu | `"true"` or `"false"` |
| `aria-checked` | Checkbox/radio items | `"true"`, `"false"`, or `"mixed"` |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Enter/Space | Activate focused menuitem |
| Arrow Down | Move to next menuitem (in vertical menu) |
| Arrow Up | Move to previous menuitem (in vertical menu) |
| Arrow Right | Open submenu, or next menu in menubar |
| Arrow Left | Close submenu, or previous menu in menubar |
| Home | Move to first menuitem |
| End | Move to last menuitem |
| Escape | Close menu, return focus to trigger |
| Character key | Move to next item starting with that character |

#### Focus Management

- Menu trigger (button) has `aria-haspopup="true"` and `aria-expanded`.
- When menu opens, focus moves to the first menuitem.
- When menu closes (Escape or selection), focus returns to the trigger.
- Menuitems use roving tabindex (`tabindex="0"` on focused item, `-1` on others).

#### Common Mistakes

```html
<!-- Mistake: using role="menu" for navigation -->
<nav role="menu">
  <a role="menuitem" href="/about">About</a>
  <a role="menuitem" href="/contact">Contact</a>
</nav>
<!-- role="menu" is for application-style menus, not page navigation -->
<!-- Fix: use <nav><ul><li><a> for navigation -->

<!-- Mistake: no Escape key handling -->
<!-- Menu stays open when Escape is pressed -->

<!-- Mistake: items in tab order -->
<ul role="menu">
  <li role="menuitem" tabindex="0">Cut</li>
  <li role="menuitem" tabindex="0">Copy</li>
  <!-- Only the focused item should have tabindex="0" -->
</ul>
```

#### Code Examples

**HTML + JavaScript**:
```html
<button id="menu-btn" aria-haspopup="true" aria-expanded="false"
        aria-controls="action-menu">
  Actions
</button>
<ul id="action-menu" role="menu" aria-labelledby="menu-btn" hidden>
  <li role="menuitem" tabindex="-1">Cut</li>
  <li role="menuitem" tabindex="-1">Copy</li>
  <li role="menuitem" tabindex="-1">Paste</li>
  <li role="separator"></li>
  <li role="menuitem" tabindex="-1">Delete</li>
</ul>
```

**React/JSX**:
```jsx
function ActionMenu() {
  const [isOpen, setIsOpen] = useState(false);
  const [focusIndex, setFocusIndex] = useState(0);
  const triggerRef = useRef(null);
  const itemRefs = useRef([]);

  const items = ['Cut', 'Copy', 'Paste', 'Delete'];

  useEffect(() => {
    if (isOpen && itemRefs.current[focusIndex]) {
      itemRefs.current[focusIndex].focus();
    }
  }, [isOpen, focusIndex]);

  const handleTriggerKeyDown = (e) => {
    if (e.key === 'ArrowDown' || e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      setIsOpen(true);
      setFocusIndex(0);
    }
  };

  const handleMenuKeyDown = (e) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setFocusIndex((prev) => (prev + 1) % items.length);
        break;
      case 'ArrowUp':
        e.preventDefault();
        setFocusIndex((prev) => (prev - 1 + items.length) % items.length);
        break;
      case 'Home':
        e.preventDefault();
        setFocusIndex(0);
        break;
      case 'End':
        e.preventDefault();
        setFocusIndex(items.length - 1);
        break;
      case 'Escape':
        setIsOpen(false);
        triggerRef.current?.focus();
        break;
    }
  };

  return (
    <div>
      <button
        ref={triggerRef}
        aria-haspopup="true"
        aria-expanded={isOpen}
        aria-controls="action-menu"
        onClick={() => setIsOpen(!isOpen)}
        onKeyDown={handleTriggerKeyDown}
      >
        Actions
      </button>
      {isOpen && (
        <ul id="action-menu" role="menu" onKeyDown={handleMenuKeyDown}>
          {items.map((item, i) => (
            <li
              key={item}
              role="menuitem"
              tabIndex={i === focusIndex ? 0 : -1}
              ref={(el) => (itemRefs.current[i] = el)}
              onClick={() => { handleAction(item); setIsOpen(false); triggerRef.current?.focus(); }}
            >
              {item}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

#### Severity: **HIGH** (missing keyboard navigation, menu for navigation) / **MEDIUM** (missing aria-expanded)

---

### Combobox / Autocomplete

A composite widget combining a text input with a popup providing suggested values.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="combobox"` | Input element | Required |
| `aria-expanded` | Input element | `"true"` when popup is visible, `"false"` otherwise |
| `aria-controls` | Input element | ID of the popup listbox |
| `aria-activedescendant` | Input element | ID of the currently highlighted option |
| `aria-autocomplete` | Input element | `"list"`, `"inline"`, or `"both"` |
| `role="listbox"` | Popup container | Required |
| `role="option"` | Each suggestion | Required |
| `aria-selected="true"` | Highlighted option | Required |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Arrow Down | Open popup (if closed); move highlight to next option |
| Arrow Up | Move highlight to previous option |
| Enter | Accept highlighted option, close popup |
| Escape | Clear selection and close popup, or clear input |
| Home/End | Move cursor within input (standard text editing) |

#### Focus Management

- Focus stays on the input at all times.
- `aria-activedescendant` changes to reflect the visually highlighted option.
- The popup listbox is NOT in the Tab order.

#### Common Mistakes

```html
<!-- Mistake 1: focus moves to the listbox -->
<!-- The input should maintain focus; use aria-activedescendant -->

<!-- Mistake 2: missing aria-expanded -->
<input role="combobox" aria-controls="suggestions" />
<!-- No indication whether the popup is open or closed -->

<!-- Mistake 3: using role="menu" for suggestions -->
<div role="menu">...</div>
<!-- Fix: use role="listbox" with role="option" children -->

<!-- Mistake 4: not updating aria-activedescendant -->
<!-- Highlighted option changes visually but AT is unaware -->
```

#### Code Examples

**React/JSX**:
```jsx
function Combobox({ options, onSelect }) {
  const [query, setQuery] = useState('');
  const [isOpen, setIsOpen] = useState(false);
  const [activeIndex, setActiveIndex] = useState(-1);

  const filtered = options.filter(o =>
    o.label.toLowerCase().includes(query.toLowerCase())
  );

  const handleKeyDown = (e) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        if (!isOpen) { setIsOpen(true); setActiveIndex(0); }
        else setActiveIndex((prev) => Math.min(prev + 1, filtered.length - 1));
        break;
      case 'ArrowUp':
        e.preventDefault();
        setActiveIndex((prev) => Math.max(prev - 1, 0));
        break;
      case 'Enter':
        if (activeIndex >= 0 && filtered[activeIndex]) {
          onSelect(filtered[activeIndex]);
          setQuery(filtered[activeIndex].label);
          setIsOpen(false);
          setActiveIndex(-1);
        }
        break;
      case 'Escape':
        setIsOpen(false);
        setActiveIndex(-1);
        break;
    }
  };

  return (
    <div className="combobox-container">
      <label htmlFor="search-input">Search</label>
      <input
        id="search-input"
        role="combobox"
        aria-expanded={isOpen}
        aria-controls="suggestion-list"
        aria-activedescendant={activeIndex >= 0 ? `option-${activeIndex}` : undefined}
        aria-autocomplete="list"
        value={query}
        onChange={(e) => { setQuery(e.target.value); setIsOpen(true); setActiveIndex(-1); }}
        onKeyDown={handleKeyDown}
        onFocus={() => query && setIsOpen(true)}
      />
      {isOpen && filtered.length > 0 && (
        <ul id="suggestion-list" role="listbox">
          {filtered.map((option, i) => (
            <li
              key={option.id}
              id={`option-${i}`}
              role="option"
              aria-selected={i === activeIndex}
              onClick={() => { onSelect(option); setQuery(option.label); setIsOpen(false); }}
            >
              {option.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

#### Severity: **HIGH** (missing combobox role, focus moves to popup) / **MEDIUM** (missing aria-activedescendant)

---

### Accordion

A vertically stacked set of interactive headings that each contain a title and
content section. Each heading acts as a toggle for its associated content.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `aria-expanded` | Header button | `"true"` or `"false"` |
| `aria-controls` | Header button | ID of the associated panel |
| `role="region"` | Panel | Optional, for important panels |
| `aria-labelledby` | Panel | ID of the header button |

The header button should be inside a heading element (`<h3>`, etc.) to ensure
proper document structure.

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Enter/Space | Toggle the accordion panel |
| Arrow Down | Move focus to next accordion header (optional) |
| Arrow Up | Move focus to previous accordion header (optional) |
| Home | Move focus to first accordion header (optional) |
| End | Move focus to last accordion header (optional) |

Arrow key navigation between headers is recommended but optional. At minimum,
Enter/Space must toggle the panel.

#### Common Mistakes

```html
<!-- Mistake 1: div as toggle without button role -->
<div class="accordion-header" onclick="toggle()">Section Title</div>
<!-- Fix: use a button -->
<h3><button aria-expanded="false" aria-controls="panel-1">Section Title</button></h3>

<!-- Mistake 2: missing aria-expanded -->
<button class="accordion-trigger">Section Title</button>
<!-- No aria-expanded means AT cannot tell if the panel is open -->

<!-- Mistake 3: using display:none on panel but not updating aria-expanded -->
<!-- Visual state and ARIA state diverge -->
```

#### Code Examples

**React/JSX**:
```jsx
function Accordion({ items }) {
  const [openItems, setOpenItems] = useState(new Set());

  const toggle = (id) => {
    setOpenItems((prev) => {
      const next = new Set(prev);
      if (next.has(id)) next.delete(id);
      else next.add(id);
      return next;
    });
  };

  return (
    <div className="accordion">
      {items.map((item) => {
        const isOpen = openItems.has(item.id);
        return (
          <div key={item.id}>
            <h3>
              <button
                aria-expanded={isOpen}
                aria-controls={`panel-${item.id}`}
                id={`header-${item.id}`}
                onClick={() => toggle(item.id)}
              >
                {item.title}
              </button>
            </h3>
            <div
              id={`panel-${item.id}`}
              role="region"
              aria-labelledby={`header-${item.id}`}
              hidden={!isOpen}
            >
              {item.content}
            </div>
          </div>
        );
      })}
    </div>
  );
}
```

#### Severity: **HIGH** (non-button trigger, missing aria-expanded) / **MEDIUM** (missing region role)

---

### Toast / Notification

A non-modal message that appears temporarily to provide feedback. Toasts should
not require user action and should not move focus.

#### Required ARIA

| Attribute | Element | Use Case |
|-----------|---------|----------|
| `role="status"` | Container | Non-urgent success/info messages |
| `role="alert"` | Container | Urgent errors, warnings |
| `aria-live="polite"` | Container | Implied by `role="status"` |
| `aria-live="assertive"` | Container | Implied by `role="alert"` |
| `aria-atomic="true"` | Container | Ensure whole message is read |

#### Focus Management

- Do NOT move focus to the toast.
- The live region handles announcement to screen readers.
- If the toast contains an action (e.g., "Undo"), the action must be reachable
  via keyboard, but focus should not automatically move to it.

#### Common Mistakes

```jsx
// Mistake 1: moving focus to the toast
toast.focus(); // Disrupts user's workflow

// Mistake 2: conditionally rendering the live region
{showToast && <div role="status">Saved!</div>}
// The region may not be announced because it appears and disappears

// Fix: keep the live region in DOM, change content
<div role="status" aria-live="polite" aria-atomic="true">
  {showToast ? message : ''}
</div>

// Mistake 3: using role="alert" for success messages
<div role="alert">Item saved successfully!</div>
// role="alert" interrupts; use role="status" for non-urgent

// Mistake 4: toast disappears too quickly for screen reader
// Minimum display time should be ~5 seconds
```

#### Code Examples

**React/JSX**:
```jsx
function ToastContainer({ toasts }) {
  return (
    <div className="toast-container" aria-live="polite" aria-atomic="true" role="status">
      {toasts.map((toast) => (
        <div key={toast.id} className={`toast toast-${toast.type}`}>
          <span>{toast.message}</span>
          {toast.action && (
            <button onClick={toast.action.handler}>
              {toast.action.label}
            </button>
          )}
        </div>
      ))}
    </div>
  );
}
```

#### Severity: **MEDIUM** (missing live region) / **HIGH** (focus stolen by toast, urgent info not using alert)

---

### Carousel / Slider

A set of content items displayed one or a few at a time, with controls to
navigate between items.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="region"` | Carousel container | Required |
| `aria-roledescription="carousel"` | Carousel container | Recommended |
| `aria-label` | Carousel container | Descriptive label |
| `role="group"` | Each slide | Required |
| `aria-roledescription="slide"` | Each slide | Recommended |
| `aria-label` | Each slide | "N of M" or descriptive |
| `aria-live="off"` | Slide container | When auto-rotating |
| `aria-live="polite"` | Slide container | When user-controlled |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Arrow Right/Next button | Go to next slide |
| Arrow Left/Previous button | Go to previous slide |
| Space/Enter on pagination | Go to that slide |
| Tab | Move between carousel controls |

#### Focus Management

- Auto-rotating carousels must have a pause button.
- If `prefers-reduced-motion` is set, auto-rotation should be disabled.
- Focus should not be trapped inside the carousel.
- When navigation controls are used, announce the new slide content.

#### Common Mistakes

```html
<!-- Mistake 1: auto-play with no pause control -->
<div class="carousel" data-autoplay="true">
  <!-- No pause button -->
</div>

<!-- Mistake 2: no indication of current slide -->
<!-- Slide pagination dots that are purely visual -->

<!-- Mistake 3: content hidden with display:none not announced on change -->
<!-- Use aria-live to announce slide changes -->

<!-- Mistake 4: ignoring prefers-reduced-motion -->
```

#### Code Examples

**HTML structure**:
```html
<div role="region" aria-roledescription="carousel" aria-label="Product highlights">
  <button aria-label="Pause auto-rotation" id="pause-btn">Pause</button>
  <div aria-live="polite" aria-atomic="true">
    <div role="group" aria-roledescription="slide" aria-label="1 of 3">
      <!-- Slide content -->
    </div>
  </div>
  <button aria-label="Previous slide">Previous</button>
  <button aria-label="Next slide">Next</button>
  <div role="tablist" aria-label="Choose slide">
    <button role="tab" aria-selected="true" aria-label="Slide 1">1</button>
    <button role="tab" aria-selected="false" aria-label="Slide 2">2</button>
    <button role="tab" aria-selected="false" aria-label="Slide 3">3</button>
  </div>
</div>
```

#### Severity: **HIGH** (no pause control on auto-rotation, no keyboard navigation) / **MEDIUM** (missing ARIA attributes)

---

### Tree View

A hierarchical list where items can be expanded to reveal nested children.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="tree"` | Root container | Required |
| `role="treeitem"` | Each item | Required |
| `role="group"` | Container of child items | Required |
| `aria-expanded` | Items with children | `"true"` or `"false"` |
| `aria-selected` | Selectable items | `"true"` or `"false"` |
| `aria-label` or `aria-labelledby` | Root tree | Required |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Arrow Down | Move to next visible treeitem |
| Arrow Up | Move to previous visible treeitem |
| Arrow Right | Expand closed node; if open, move to first child |
| Arrow Left | Collapse open node; if closed, move to parent |
| Home | Move to first treeitem |
| End | Move to last visible treeitem |
| Enter | Activate item (perform default action) |
| Space | Toggle selection (in multi-select trees) |
| * (asterisk) | Expand all siblings at current level |
| Type-ahead | Move to next item matching typed characters |

#### Code Examples

**React/JSX**:
```jsx
function TreeItem({ item, level = 1 }) {
  const [expanded, setExpanded] = useState(false);
  const hasChildren = item.children && item.children.length > 0;

  const handleKeyDown = (e) => {
    switch (e.key) {
      case 'ArrowRight':
        if (hasChildren && !expanded) setExpanded(true);
        break;
      case 'ArrowLeft':
        if (hasChildren && expanded) setExpanded(false);
        break;
      case 'Enter':
        item.onActivate?.();
        break;
    }
  };

  return (
    <li
      role="treeitem"
      aria-expanded={hasChildren ? expanded : undefined}
      aria-level={level}
      aria-setsize={item.setSize}
      aria-posinset={item.posInSet}
    >
      <span
        tabIndex={item.isFocused ? 0 : -1}
        onClick={() => hasChildren && setExpanded(!expanded)}
        onKeyDown={handleKeyDown}
      >
        {hasChildren && (expanded ? '▼' : '▶')}
        {item.label}
      </span>
      {hasChildren && expanded && (
        <ul role="group">
          {item.children.map((child) => (
            <TreeItem key={child.id} item={child} level={level + 1} />
          ))}
        </ul>
      )}
    </li>
  );
}

function Tree({ items, label }) {
  return (
    <ul role="tree" aria-label={label}>
      {items.map((item) => (
        <TreeItem key={item.id} item={item} />
      ))}
    </ul>
  );
}
```

#### Severity: **HIGH** (missing keyboard navigation, wrong roles) / **MEDIUM** (missing aria-expanded)

---

### Disclosure (Show/Hide)

A button that controls the visibility of a section of content. Simpler than
an accordion (no heading structure or mutual exclusion required).

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `aria-expanded` | Toggle button | `"true"` or `"false"` |
| `aria-controls` | Toggle button | ID of the content section |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Enter/Space | Toggle content visibility |

#### Code Examples

**HTML**:
```html
<button aria-expanded="false" aria-controls="details-panel">
  Show details
</button>
<div id="details-panel" hidden>
  <p>Additional details here.</p>
</div>
```

**HTML with native `<details>` (preferred)**:
```html
<!-- Native HTML disclosure -- no ARIA or JS needed -->
<details>
  <summary>Show details</summary>
  <p>Additional details here.</p>
</details>
```

**React/JSX**:
```jsx
function Disclosure({ label, children }) {
  const [isOpen, setIsOpen] = useState(false);
  const panelId = useId();

  return (
    <div>
      <button
        aria-expanded={isOpen}
        aria-controls={panelId}
        onClick={() => setIsOpen(!isOpen)}
      >
        {label}
      </button>
      <div id={panelId} hidden={!isOpen}>
        {children}
      </div>
    </div>
  );
}
```

#### Severity: **HIGH** (missing aria-expanded) / **LOW** (using ARIA when `<details>` would work)

---

### Toolbar

A container for a group of related controls. Provides a grouping label and
streamlined keyboard navigation.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="toolbar"` | Container | Required |
| `aria-label` or `aria-labelledby` | Container | Required |
| `aria-orientation` | Container | `"horizontal"` (default) or `"vertical"` |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Arrow Right/Left (horizontal) | Move focus between toolbar items |
| Arrow Down/Up (vertical) | Move focus between toolbar items |
| Home | Move to first item |
| End | Move to last item |
| Tab | Move focus out of toolbar to next element in page |

#### Focus Management

- Use roving tabindex: only one item in the toolbar has `tabindex="0"`.
- Arrow keys move focus between items.
- Tab leaves the toolbar entirely.

#### Code Examples

**HTML**:
```html
<div role="toolbar" aria-label="Text formatting" aria-orientation="horizontal">
  <button tabindex="0" aria-pressed="false">Bold</button>
  <button tabindex="-1" aria-pressed="false">Italic</button>
  <button tabindex="-1" aria-pressed="false">Underline</button>
  <span role="separator" aria-orientation="vertical"></span>
  <button tabindex="-1">Align Left</button>
  <button tabindex="-1">Align Center</button>
  <button tabindex="-1">Align Right</button>
</div>
```

#### Severity: **MEDIUM** (missing toolbar role or keyboard navigation)

---

### Listbox

A widget that allows the user to select one or more items from a list of options.
Used for custom-styled select alternatives.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="listbox"` | Container | Required |
| `role="option"` | Each option | Required |
| `aria-selected` | Each option | `"true"` or `"false"` |
| `aria-multiselectable` | Container | `"true"` if multi-select |
| `aria-label` or `aria-labelledby` | Container | Required |
| `aria-activedescendant` | Container (if focus stays on container) | ID of focused option |

#### Keyboard Interaction (single select)

| Key | Behavior |
|-----|----------|
| Arrow Down | Move focus and selection to next option |
| Arrow Up | Move focus and selection to previous option |
| Home | Move to first option |
| End | Move to last option |
| Type-ahead | Jump to option matching typed characters |

#### Keyboard Interaction (multi-select additions)

| Key | Behavior |
|-----|----------|
| Space | Toggle selection of focused option |
| Shift+Arrow | Extend selection |
| Ctrl+A | Select all |

#### Common Mistakes

```html
<!-- Mistake: using a <div> list without ARIA roles -->
<div class="custom-select">
  <div class="option selected">Option 1</div>
  <div class="option">Option 2</div>
</div>
<!-- No roles, no keyboard navigation, not accessible -->

<!-- Fix: proper listbox roles -->
<div role="listbox" aria-label="Choose option" tabindex="0">
  <div role="option" aria-selected="true" id="opt-1">Option 1</div>
  <div role="option" aria-selected="false" id="opt-2">Option 2</div>
</div>

<!-- Better fix: use native <select> when possible -->
<select>
  <option>Option 1</option>
  <option>Option 2</option>
</select>
```

#### Severity: **HIGH** (custom select without ARIA) / **INFO** (using native `<select>` -- positive pattern)

---

### Grid

An interactive table-like structure where cells can be navigated and activated.
Not for data tables (use `<table>` with proper headers). Grids are for interactive
spreadsheet-like interfaces.

#### Required ARIA

| Attribute | Element | Value |
|-----------|---------|-------|
| `role="grid"` | Container | Required |
| `role="row"` | Each row | Required |
| `role="gridcell"` | Each cell | Required |
| `role="rowheader"` | Header cells in rows | When applicable |
| `role="columnheader"` | Header cells in columns | When applicable |
| `aria-label` or `aria-labelledby` | Grid | Required |
| `aria-colcount` | Grid | Total column count (if virtualized) |
| `aria-rowcount` | Grid | Total row count (if virtualized) |

#### Keyboard Interaction

| Key | Behavior |
|-----|----------|
| Arrow Right | Move to next cell in row |
| Arrow Left | Move to previous cell in row |
| Arrow Down | Move to same column in next row |
| Arrow Up | Move to same column in previous row |
| Home | Move to first cell in row |
| End | Move to last cell in row |
| Ctrl+Home | Move to first cell in grid |
| Ctrl+End | Move to last cell in grid |
| Enter | Enter edit mode on cell / activate cell |
| Escape | Exit edit mode |
| Tab | Move to next interactive element outside grid (or within cell if in edit mode) |

#### Focus Management

- One cell in the grid has `tabindex="0"`, all others have `tabindex="-1"`.
- Arrow keys move focus between cells using roving tabindex.
- Tab exits the grid to the next page element.
- If a cell contains interactive elements, Enter enters the cell and Tab cycles
  within the cell; Escape returns to cell navigation mode.

#### Severity: **HIGH** (grid pattern without keyboard navigation) / **MEDIUM** (missing aria attributes)

---

## Detection Pattern Summary

Quick reference for searching codebases for potential ARIA issues.

### Patterns That Suggest Missing ARIA

```regex
# Custom interactive elements without roles
<(div|span)\s[^>]*(onclick|onClick|@click)[^>]*(?!\brole\s*=)[^>]*>

# Toggle/expand elements without aria-expanded
(?i)(toggle|expand|collapse|accordion|disclosure)(?![\s\S]{0,200}aria-expanded)

# Popup/dropdown elements without aria-haspopup
(?i)(dropdown|popup|menu-trigger|popover)(?![\s\S]{0,200}aria-haspopup)

# Dynamic content areas without aria-live
(?i)(toast|notification|alert|status|snackbar|message-area|flash)(?![\s\S]{0,200}aria-live|role="(alert|status)")

# Custom select/combobox without ARIA
(?i)(custom-select|autocomplete|combobox|typeahead|search-dropdown)(?![\s\S]{0,200}role="(combobox|listbox)")

# Tab interfaces without tab roles
(?i)(tab-list|tab-container|tab-nav|tabs-wrapper)(?![\s\S]{0,200}role="tablist")
```

### Patterns That Suggest Incorrect ARIA

```regex
# aria-hidden on focusable elements
aria-hidden\s*=\s*["']true["'][^>]*tabindex\s*=\s*["'](?!-1)
aria-hidden\s*=\s*["']true["'][^>]*>\s*<(button|a\s|input|select|textarea)

# Positive tabindex (breaks natural tab order)
tabindex\s*=\s*["']?[1-9]

# aria-label on non-labelable elements without role
<(div|span|p)\s(?![^>]*role\s*=)[^>]*aria-label\s*=

# role="menu" used for navigation (should be <nav>)
role\s*=\s*["']menu["'][^>]*>[\s\S]*?<a\s[^>]*href\s*=

# Redundant role on semantic element
<nav\s[^>]*role\s*=\s*["']navigation["']
<main\s[^>]*role\s*=\s*["']main["']
<button\s[^>]*role\s*=\s*["']button["']
<header\s[^>]*role\s*=\s*["']banner["']
<footer\s[^>]*role\s*=\s*["']contentinfo["']

# Invalid ARIA attribute values
aria-expanded\s*=\s*["'](yes|no|1|0)["']
aria-hidden\s*=\s*["'](yes|no|1|0)["']
aria-selected\s*=\s*["'](yes|no|1|0)["']
aria-checked\s*=\s*["'](yes|no|1|0)["']
```

---

## Quick-Reference Summary

| Pattern | Key ARIA | Key Keyboard | Severity When Missing |
|---------|----------|-------------|----------------------|
| Modal Dialog | `role="dialog"`, `aria-modal` | Tab trap, Escape closes | CRITICAL |
| Tabs | `role="tablist/tab/tabpanel"`, `aria-selected` | Arrow keys between tabs | HIGH |
| Menu | `role="menu/menuitem"`, `aria-haspopup`, `aria-expanded` | Arrows, Escape, type-ahead | HIGH |
| Combobox | `role="combobox"`, `aria-expanded`, `aria-activedescendant` | Arrows, Enter, Escape | HIGH |
| Accordion | `aria-expanded`, `aria-controls` | Enter/Space toggle | HIGH |
| Toast | `role="status"` or `role="alert"` | No focus movement | MEDIUM |
| Carousel | `role="region"`, `aria-roledescription`, `aria-live` | Arrows, pause control | HIGH |
| Tree View | `role="tree/treeitem"`, `aria-expanded` | Arrows (4-direction), Home/End | HIGH |
| Disclosure | `aria-expanded`, `aria-controls` | Enter/Space | HIGH |
| Toolbar | `role="toolbar"`, `aria-label` | Arrow keys, roving tabindex | MEDIUM |
| Listbox | `role="listbox/option"`, `aria-selected` | Arrows, type-ahead | HIGH |
| Grid | `role="grid/row/gridcell"` | 4-direction arrows, Enter/Escape | HIGH |

### Severity Classification

| Condition | Severity |
|-----------|----------|
| Interactive element with no accessible name | CRITICAL |
| `aria-hidden="true"` on focusable element | CRITICAL |
| Keyboard trap (no way to exit component) | CRITICAL |
| Focus not managed on dialog open/close | CRITICAL |
| ARIA role used instead of native HTML element | MEDIUM |
| Missing `aria-expanded` on toggle controls | HIGH |
| Missing `aria-live` on dynamic content | HIGH (if content is important) |
| Incorrect `aria-live` politeness level | MEDIUM |
| Missing keyboard navigation on widget | HIGH |
| Redundant ARIA on semantic HTML | LOW |
| Invalid ARIA attribute values | HIGH |
| ARIA reference to non-existent ID | HIGH |
| Missing `aria-label` on icon-only button | CRITICAL |
| Using `role="menu"` for page navigation | HIGH |
