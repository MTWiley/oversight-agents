# WCAG 2.1 Standards Reference Checklist

Comprehensive reference for evaluating web content against WCAG 2.1 Level A and AA
success criteria. Complements the inlined criteria in `review-accessibility.md` with
expanded explanations, severity classifications, detection patterns, code examples,
and automated testing coverage for each success criterion.

Organized by the four WCAG principles: Perceivable, Operable, Understandable, Robust.

---

## Principle 1: Perceivable

Information and user interface components must be presentable to users in ways they
can perceive. Content must not be invisible to all of a user's senses.

---

### 1.1.1 Non-text Content (Level A)

All non-text content has a text alternative that serves the equivalent purpose.

**What this covers**: images, icons, image buttons, image maps, charts, diagrams,
audio/video, CAPTCHA, decorative elements, `<canvas>`, `<svg>`, emoji used as content.

#### Detection Patterns

```regex
# Images without alt attribute
<img\s+(?![^>]*\balt\s*=)[^>]*>

# SVGs without accessible name
<svg\s+(?![^>]*\b(aria-label|aria-labelledby|role)\s*=)[^>]*>

# Icon-only buttons/links without accessible label
<(button|a)\s[^>]*>\s*<(i|span|svg)\s[^>]*(class="[^"]*icon)[^>]*>\s*</(i|span|svg)>\s*</(button|a)>

# Canvas without fallback
<canvas\s[^>]*>\s*</canvas>

# Input type="image" without alt
<input\s[^>]*type\s*=\s*["']image["'][^>]*(?!\balt\s*=)[^>]*>
```

#### Violation Examples

**Missing alt on functional image (HTML)**:
```html
<!-- Violation: no alt text on linked image -->
<a href="/home"><img src="logo.png"></a>

<!-- Fix: describe the image's function -->
<a href="/home"><img src="logo.png" alt="Acme Corp home page"></a>
```

**Decorative image not marked decorative (React/JSX)**:
```jsx
{/* Violation: screen reader announces file name */}
<img src="/decorative-border.svg" />

{/* Fix: empty alt marks it as decorative */}
<img src="/decorative-border.svg" alt="" />
```

**Icon button without label (React/JSX)**:
```jsx
// Violation: button has no accessible name
<button onClick={handleClose}>
  <CloseIcon />
</button>

// Fix: add aria-label
<button onClick={handleClose} aria-label="Close dialog">
  <CloseIcon aria-hidden="true" />
</button>
```

**SVG without accessible name (Vue)**:
```vue
<!-- Violation: SVG has no accessible name -->
<template>
  <svg viewBox="0 0 24 24">
    <path d="M12 2L2 22h20L12 2z" />
  </svg>
</template>

<!-- Fix: add title element and aria-labelledby -->
<template>
  <svg viewBox="0 0 24 24" aria-labelledby="warning-icon-title" role="img">
    <title id="warning-icon-title">Warning</title>
    <path d="M12 2L2 22h20L12 2z" />
  </svg>
</template>
```

**Canvas without fallback (HTML)**:
```html
<!-- Violation: no fallback content -->
<canvas id="chart"></canvas>

<!-- Fix: provide text alternative -->
<canvas id="chart" aria-label="Sales chart showing Q1-Q4 revenue">
  <p>Sales revenue by quarter: Q1: $120k, Q2: $145k, Q3: $132k, Q4: $178k</p>
</canvas>
```

#### Severity: **CRITICAL** (functional images, form controls) / **MEDIUM** (informational images) / **LOW** (decorative images not marked)

#### Automated Testing Coverage

| Tool | Coverage |
|------|----------|
| axe-core | Detects missing `alt`, empty `alt` on functional images, `<svg>` without name |
| pa11y | Detects missing `alt` attributes |
| eslint-plugin-jsx-a11y | `alt-text`, `img-redundant-alt`, `anchor-has-content` rules |
| vue-a11y | `alt-text` rule |

Manual testing required for: verifying alt text quality and accuracy, complex image descriptions, chart/diagram alternatives.

---

### 1.2.1 Audio-only and Video-only (Prerecorded) (Level A)

Prerecorded audio-only content has a text transcript. Prerecorded video-only content
has either a text alternative or an audio track describing the visual content.

#### Detection Patterns

```regex
# Audio elements without associated transcript
<audio\s[^>]*src\s*=

# Video elements -- check for track elements
<video\s[^>]*>(?![\s\S]*<track\s)
```

#### Violation Examples

```html
<!-- Violation: audio with no transcript -->
<audio src="/podcast-ep1.mp3" controls></audio>

<!-- Fix: provide transcript link -->
<audio src="/podcast-ep1.mp3" controls></audio>
<a href="/transcripts/podcast-ep1.html">Read transcript</a>
```

#### Severity: **HIGH**

#### Automated Testing Coverage

Automated tools cannot verify transcript content quality. They can detect the absence
of `<track>` elements on `<video>` and flag `<audio>` elements for manual review.

---

### 1.2.2 Captions (Prerecorded) (Level A)

Captions are provided for all prerecorded audio content in synchronized media.

#### Detection Patterns

```regex
# Video without captions track
<video\s[^>]*>(?![\s\S]*<track\s[^>]*kind\s*=\s*["']captions["'])
```

#### Violation Examples

```html
<!-- Violation: video without captions -->
<video src="/training.mp4" controls></video>

<!-- Fix: add captions track -->
<video src="/training.mp4" controls>
  <track kind="captions" src="/captions/training.vtt" srclang="en" label="English" default>
</video>
```

#### Severity: **HIGH**

#### Automated Testing Coverage

axe-core and pa11y can detect missing `<track>` elements. Caption quality requires
manual review.

---

### 1.2.3 Audio Description or Media Alternative (Prerecorded) (Level A)

An alternative for time-based media or audio description is provided for prerecorded
video content.

#### Severity: **HIGH**

#### Automated Testing Coverage

Cannot be automated. Requires manual review to determine whether visual information
in video is conveyed through the audio track or a separate audio description track.

---

### 1.2.5 Audio Description (Prerecorded) (Level AA)

Audio description is provided for all prerecorded video content in synchronized media.

#### Severity: **HIGH**

#### Automated Testing Coverage

Cannot be automated. Requires manual review.

---

### 1.3.1 Info and Relationships (Level A)

Information, structure, and relationships conveyed through presentation are
programmatically determinable or available in text.

**What this covers**: headings, lists, tables, form labels, landmarks, grouping.

#### Detection Patterns

```regex
# Headings by visual styling only (div/span with heading-like classes)
<(div|span|p)\s[^>]*class\s*=\s*["'][^"']*(heading|title|h[1-6])[^"']*["']

# Lists faked with divs/spans
<div\s[^>]*class\s*=\s*["'][^"']*(list-item|bullet)[^"']*["']

# Data tables without headers
<table\s[^>]*>[\s\S]*?<td[\s>](?![\s\S]*<th[\s>])

# Form inputs without labels
<input\s+(?![^>]*type\s*=\s*["'](hidden|submit|button|reset|image)["'])(?![^>]*aria-label)[^>]*(?<!id\s*=\s*["'][^"']*["'][^>]*)>

# Missing landmark regions
# Check: page has <main>, <nav>, <header>, <footer> or equivalent ARIA roles
```

#### Violation Examples

**Fake heading (HTML)**:
```html
<!-- Violation: visual heading without semantic meaning -->
<div class="heading-large" style="font-size: 24px; font-weight: bold;">
  Product Features
</div>

<!-- Fix: use proper heading element -->
<h2>Product Features</h2>
```

**Data table without headers (HTML)**:
```html
<!-- Violation: table cells only, no headers -->
<table>
  <tr><td>Name</td><td>Email</td><td>Role</td></tr>
  <tr><td>Alice</td><td>alice@example.com</td><td>Admin</td></tr>
</table>

<!-- Fix: use th elements with scope -->
<table>
  <caption>Team members</caption>
  <thead>
    <tr><th scope="col">Name</th><th scope="col">Email</th><th scope="col">Role</th></tr>
  </thead>
  <tbody>
    <tr><td>Alice</td><td>alice@example.com</td><td>Admin</td></tr>
  </tbody>
</table>
```

**Form input without label (React/JSX)**:
```jsx
// Violation: input not associated with label
<div>
  <span>Email</span>
  <input type="email" name="email" />
</div>

// Fix: use htmlFor and id
<div>
  <label htmlFor="email-input">Email</label>
  <input type="email" name="email" id="email-input" />
</div>
```

**Missing landmark regions (Vue)**:
```vue
<!-- Violation: no landmarks -->
<template>
  <div id="app">
    <div class="nav">...</div>
    <div class="content">...</div>
    <div class="footer">...</div>
  </div>
</template>

<!-- Fix: semantic landmarks -->
<template>
  <div id="app">
    <nav aria-label="Main navigation">...</nav>
    <main>...</main>
    <footer>...</footer>
  </div>
</template>
```

#### Severity: **HIGH** (missing form labels, fake headings) / **MEDIUM** (missing landmarks, table structure)

#### Automated Testing Coverage

| Tool | Coverage |
|------|----------|
| axe-core | Missing form labels, table headers, duplicate IDs, landmark regions |
| pa11y | Missing form labels, heading structure |
| eslint-plugin-jsx-a11y | `label-has-associated-control`, `heading-has-content` |
| vue-a11y | `label-has-for`, `form-control-has-label` |

Manual testing required for: verifying heading hierarchy logic, confirming labels
describe their fields accurately, checking reading order.

---

### 1.3.2 Meaningful Sequence (Level A)

When the sequence in which content is presented affects its meaning, a correct
reading sequence can be programmatically determined.

#### Detection Patterns

```regex
# CSS that reorders content visually (may break reading order)
order\s*:\s*-?\d+
flex-direction\s*:\s*row-reverse
flex-direction\s*:\s*column-reverse
grid-row\s*:.*\/
grid-column\s*:.*\/

# Absolute/fixed positioning that may change visual order
position\s*:\s*(absolute|fixed)
```

#### Violation Examples

```css
/* Violation: visual order differs from DOM order */
.sidebar { order: 2; }
.main-content { order: 1; }
/* If sidebar appears first in DOM but visually second,
   screen reader users get content in wrong order */

/* Fix: match DOM order to visual order, or use aria-flowto if needed */
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Automated tools generally cannot verify reading order. axe-core detects some
`tabindex` ordering issues. Manual screen reader testing is required.

---

### 1.3.3 Sensory Characteristics (Level A)

Instructions do not rely solely on sensory characteristics (shape, color, size,
visual location, orientation, or sound).

#### Detection Patterns

```regex
# Instructions referencing visual properties only
(?i)(click the (red|green|blue|round|square) button)
(?i)(see the (left|right|top|bottom) (panel|sidebar|section))
(?i)(the (circular|triangular|square) icon)
```

#### Violation Examples

```html
<!-- Violation: instruction relies on visual position -->
<p>Click the button on the right to continue.</p>

<!-- Fix: use the button's label -->
<p>Click "Continue" to proceed.</p>
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be fully automated. Requires manual review of instructional text.

---

### 1.3.4 Orientation (Level AA)

Content does not restrict its view and operation to a single display orientation
unless essential.

#### Detection Patterns

```regex
# CSS or JS that locks orientation
orientation\s*:\s*(portrait|landscape)
screen\.orientation\.lock
screen\.lockOrientation
```

#### Violation Examples

```css
/* Violation: hiding content in landscape */
@media (orientation: landscape) {
  .main-content { display: none; }
  .rotate-message { display: block; }
}

/* Fix: support both orientations */
@media (orientation: landscape) {
  .main-content { flex-direction: row; }
}
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Automated tools cannot reliably detect orientation restrictions. Manual testing
on mobile devices required.

---

### 1.3.5 Identify Input Purpose (Level AA)

The purpose of form fields collecting user information can be programmatically
determined using `autocomplete` attributes.

#### Detection Patterns

```regex
# Common personal info fields without autocomplete
<input\s[^>]*(?:name|id)\s*=\s*["'](?:name|email|phone|address|city|zip|cc-|postal)[^"']*["'][^>]*(?!\bautocomplete\s*=)[^>]*>
```

#### Violation Examples

```html
<!-- Violation: personal info fields without autocomplete -->
<input type="text" name="full-name" />
<input type="email" name="email" />
<input type="tel" name="phone" />

<!-- Fix: add autocomplete attributes -->
<input type="text" name="full-name" autocomplete="name" />
<input type="email" name="email" autocomplete="email" />
<input type="tel" name="phone" autocomplete="tel" />
```

```jsx
// React equivalent
<input type="text" name="full-name" autoComplete="name" />
<input type="email" name="email" autoComplete="email" />
```

#### Severity: **LOW** to **MEDIUM**

#### Automated Testing Coverage

axe-core does not currently check autocomplete values. Manual review or custom
linting rules required.

---

### 1.4.1 Use of Color (Level A)

Color is not used as the only visual means of conveying information, indicating an
action, prompting a response, or distinguishing a visual element.

#### Detection Patterns

```regex
# Error states using only color
(?i)class\s*=\s*["'][^"']*(error|invalid|danger)[^"']*["']
# Check: is there also an icon, text, or pattern to convey the state?

# Form validation relying on color alone
(?i)border-color\s*:\s*(red|#f00|#ff0000|rgb\(255)
# Check: is error text also displayed?
```

#### Violation Examples

```html
<!-- Violation: error indicated only by red border -->
<input type="email" style="border-color: red;" />

<!-- Fix: add error text and icon -->
<input type="email" style="border-color: red;" aria-describedby="email-error" aria-invalid="true" />
<span id="email-error" class="error-message">
  <svg aria-hidden="true"><!-- error icon --></svg>
  Please enter a valid email address.
</span>
```

```jsx
// Violation: status indicated only by color
<span className={status === 'active' ? 'text-green' : 'text-red'}>
  {status}
</span>

// Fix: add icon or text indicator
<span className={status === 'active' ? 'text-green' : 'text-red'}>
  {status === 'active' ? '● Active' : '○ Inactive'}
</span>
```

#### Severity: **HIGH**

#### Automated Testing Coverage

axe-core can detect some color-only indicators in links (link-in-text-block rule).
Most color-as-information issues require manual review.

---

### 1.4.2 Audio Control (Level A)

If audio plays automatically for more than 3 seconds, there is a mechanism to
pause/stop it or control volume independently.

#### Detection Patterns

```regex
# Autoplay audio/video
<(audio|video)\s[^>]*autoplay[^>]*>
autoplay\s*:\s*true
\.play\(\)
```

#### Severity: **HIGH**

#### Automated Testing Coverage

axe-core detects `autoplay` on `<video>` and `<audio>` elements.

---

### 1.4.3 Contrast (Minimum) (Level AA)

Text and images of text have a contrast ratio of at least 4.5:1 (normal text) or
3:1 (large text, 18pt or 14pt bold).

#### Contrast Ratio Calculations

The contrast ratio formula is:

```
(L1 + 0.05) / (L2 + 0.05)
```

Where L1 is the relative luminance of the lighter color and L2 is the relative
luminance of the darker color. Relative luminance is calculated from sRGB values.

**Common failing combinations**:

| Text Color | Background | Ratio | Pass AA? |
|-----------|------------|-------|----------|
| `#767676` | `#ffffff` | 4.54:1 | Yes (barely) |
| `#777777` | `#ffffff` | 4.48:1 | No |
| `#808080` | `#ffffff` | 3.95:1 | No (large text only) |
| `#999999` | `#ffffff` | 2.85:1 | No |
| `#ffffff` | `#59a3fc` | 2.44:1 | No |
| `#ffffff` | `#0066cc` | 5.07:1 | Yes |

**Large text threshold**: 18pt (24px) regular weight, or 14pt (18.66px) bold weight.

#### Detection Patterns

```regex
# Light gray text (potential contrast issue)
color\s*:\s*#[89a-fA-F]{6}
color\s*:\s*#[89a-fA-F]{3}\b
color\s*:\s*rgb\(\s*[1-2]\d{2}

# Placeholder text (often insufficient contrast)
::placeholder\s*\{[^}]*color
::-webkit-input-placeholder
```

#### Violation Examples

```css
/* Violation: insufficient contrast (3.95:1) */
.muted-text {
  color: #808080;
  background-color: #ffffff;
}

/* Fix: darken text to meet 4.5:1 */
.muted-text {
  color: #767676; /* 4.54:1 ratio */
  background-color: #ffffff;
}
```

```css
/* Violation: placeholder text too light */
::placeholder {
  color: #cccccc; /* 1.61:1 ratio */
}

/* Fix: use darker placeholder */
::placeholder {
  color: #767676; /* 4.54:1 ratio */
}
```

#### Severity: **MEDIUM** (near misses, non-critical text) / **HIGH** (body text, navigation, form labels)

#### Automated Testing Coverage

| Tool | Coverage |
|------|----------|
| axe-core | Detects contrast failures on rendered elements; highly reliable |
| pa11y | Detects contrast issues via WCAG2AA runner |
| Lighthouse | Contrast audit in accessibility category |

Automated tools analyze computed styles on rendered pages. Static code analysis
can flag suspicious color values but cannot reliably calculate contrast without
rendering. Transparent backgrounds, gradients, and background images require
manual verification.

---

### 1.4.4 Resize Text (Level AA)

Text can be resized up to 200% without assistive technology and without loss of
content or functionality.

#### Detection Patterns

```regex
# Fixed pixel sizes on text (may prevent scaling)
font-size\s*:\s*\d+px
# Check: does the layout break when text is scaled to 200%?

# Overflow hidden that may clip scaled text
overflow\s*:\s*hidden
text-overflow\s*:\s*ellipsis

# Fixed-height containers for text
height\s*:\s*\d+px.*font-size
line-height\s*:\s*\d+px
```

#### Violation Examples

```css
/* Violation: fixed container clips resized text */
.card-title {
  font-size: 14px;
  height: 20px;
  overflow: hidden;
}

/* Fix: use relative units and min-height */
.card-title {
  font-size: 0.875rem;
  min-height: 1.25rem;
  overflow: visible;
}
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be reliably automated. Requires manual testing at 200% browser zoom.

---

### 1.4.5 Images of Text (Level AA)

Text is used to convey information rather than images of text, except where
customizable or essential (logos).

#### Detection Patterns

```regex
# Images with filenames suggesting text content
<img\s[^>]*src\s*=\s*["'][^"']*(banner|heading|title|text|quote|slogan|tagline)[^"']*["']

# Background images on text-containing elements
background-image\s*:\s*url\([^)]*\).*font
```

#### Severity: **HIGH** (body text, navigation, headings as images) / **LOW** (logos, stylistic elements)

#### Automated Testing Coverage

Cannot be automated. Requires visual inspection.

---

### 1.4.10 Reflow (Level AA)

Content can be presented without loss of information or functionality and without
scrolling in two dimensions at 320 CSS pixels wide (vertical scrolling content)
or 256 CSS pixels tall (horizontal scrolling content).

#### Detection Patterns

```regex
# Fixed widths that prevent reflow
width\s*:\s*\d{3,}px
min-width\s*:\s*\d{3,}px

# Horizontal scrolling containers
overflow-x\s*:\s*(scroll|auto)

# Viewport meta preventing zoom
<meta\s[^>]*name\s*=\s*["']viewport["'][^>]*maximum-scale\s*=\s*["']?1
<meta\s[^>]*name\s*=\s*["']viewport["'][^>]*user-scalable\s*=\s*["']?no
```

#### Violation Examples

```html
<!-- Violation: prevents zoom on mobile -->
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">

<!-- Fix: allow user scaling -->
<meta name="viewport" content="width=device-width, initial-scale=1">
```

```css
/* Violation: fixed-width layout */
.container {
  width: 960px;
}

/* Fix: responsive layout */
.container {
  max-width: 960px;
  width: 100%;
}
```

#### Severity: **MEDIUM** to **HIGH**

#### Automated Testing Coverage

axe-core detects `user-scalable=no` and `maximum-scale=1` in viewport meta. Layout
reflow issues require manual testing at narrow viewport widths.

---

### 1.4.11 Non-text Contrast (Level AA)

UI components and graphical objects required to understand the content have a contrast
ratio of at least 3:1 against adjacent colors.

**What this covers**: form field borders, buttons, icons, chart elements, focus
indicators, custom checkboxes/radios.

#### Detection Patterns

```regex
# Light borders on form elements
border-color\s*:\s*#[c-fC-F]{6}
border\s*:\s*\d+px\s+solid\s+#[c-fC-F]{6}

# Light colored icons
fill\s*:\s*#[c-fC-F]{6}
stroke\s*:\s*#[c-fC-F]{6}
```

#### Violation Examples

```css
/* Violation: input border too light (1.98:1 against white) */
.form-input {
  border: 1px solid #cccccc;
  background: #ffffff;
}

/* Fix: darken border to meet 3:1 */
.form-input {
  border: 1px solid #767676;
  background: #ffffff;
}
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Limited automated detection. axe-core checks some component contrast. Most non-text
contrast issues require visual inspection.

---

### 1.4.12 Text Spacing (Level AA)

No loss of content or functionality when users override text spacing to: line height
1.5x font size, paragraph spacing 2x font size, letter spacing 0.12x font size,
word spacing 0.16x font size.

#### Detection Patterns

```regex
# Fixed line heights that prevent override
line-height\s*:\s*\d+px
# Check: does layout break with increased spacing?

# Overflow hidden on text containers
overflow\s*:\s*hidden.*line-height
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be automated with static analysis. Requires testing with a text spacing
bookmarklet or browser extension that overrides spacing properties.

---

### 1.4.13 Content on Hover or Focus (Level AA)

Additional content that appears on hover or focus is dismissible (Escape key),
hoverable (user can move pointer to it), and persistent (stays until dismissed
or trigger removed).

#### Detection Patterns

```regex
# Custom tooltips/popovers
:hover\s*\{[^}]*display\s*:\s*(block|flex|grid|inline)
:hover\s*\{[^}]*visibility\s*:\s*visible
:hover\s*\{[^}]*opacity\s*:\s*1
:focus\s*\{[^}]*display\s*:\s*(block|flex|grid|inline)

# JavaScript hover handlers
onmouse(enter|over)\s*=
@mouse(enter|over)\s*=
```

#### Violation Examples

```css
/* Violation: tooltip disappears when you try to hover it */
.trigger:hover + .tooltip { display: block; }

/* Fix: tooltip stays visible when hovered */
.trigger:hover + .tooltip,
.tooltip:hover { display: block; }
```

```jsx
// Violation: tooltip not dismissible
<Tooltip content="Help text">
  <button>?</button>
</Tooltip>

// Fix: tooltip dismissible with Escape
<Tooltip content="Help text" dismissOnEscape={true} hoverable={true}>
  <button>?</button>
</Tooltip>
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be automated. Requires manual testing of hover/focus interactions.

---

## Principle 2: Operable

User interface components and navigation must be operable by all users regardless
of input method.

---

### 2.1.1 Keyboard (Level A)

All functionality is operable through a keyboard interface without requiring specific
timings for individual keystrokes.

#### Detection Patterns

```regex
# Click handlers without keyboard equivalents
onclick\s*=(?![^>]*onkeydown|onkeypress|onkeyup)
onClick\s*=\s*\{(?![\s\S]*onKeyDown|onKeyPress|onKeyUp)
@click(?![\s\S]*@keydown|@keypress|@keyup)

# Non-interactive elements with click handlers
<(div|span|td|li|p)\s[^>]*(onclick|onClick|@click)\s*=

# Mouse-only event handlers
onmouseover\s*=
onmousedown\s*=
ondblclick\s*=

# Drag-and-drop without keyboard alternative
draggable\s*=\s*["']true["']
ondragstart\s*=
@dragstart\s*=
```

#### Violation Examples

**Non-interactive element as button (HTML)**:
```html
<!-- Violation: div is not keyboard accessible -->
<div class="btn" onclick="submit()">Submit</div>

<!-- Fix: use a real button -->
<button type="submit" onclick="submit()">Submit</button>
```

**Click handler without keyboard support (React/JSX)**:
```jsx
// Violation: div with click handler, not keyboard accessible
<div className="card" onClick={() => selectItem(id)}>
  {item.name}
</div>

// Fix option 1: use a button
<button className="card" onClick={() => selectItem(id)}>
  {item.name}
</button>

// Fix option 2: if button styling is undesirable, add keyboard handling and role
<div
  className="card"
  onClick={() => selectItem(id)}
  onKeyDown={(e) => { if (e.key === 'Enter' || e.key === ' ') selectItem(id); }}
  role="button"
  tabIndex={0}
>
  {item.name}
</div>
```

**Mouse-only interaction (Vue)**:
```vue
<!-- Violation: drag-and-drop with no keyboard alternative -->
<template>
  <div
    v-for="item in items"
    :key="item.id"
    draggable="true"
    @dragstart="onDragStart(item)"
    @drop="onDrop(item)"
  >
    {{ item.name }}
  </div>
</template>

<!-- Fix: add keyboard reorder buttons -->
<template>
  <div v-for="(item, index) in items" :key="item.id">
    <span>{{ item.name }}</span>
    <button @click="moveUp(index)" :disabled="index === 0" aria-label="Move up">
      &uarr;
    </button>
    <button @click="moveDown(index)" :disabled="index === items.length - 1" aria-label="Move down">
      &darr;
    </button>
  </div>
</template>
```

#### Severity: **CRITICAL** (primary functionality unreachable) / **HIGH** (secondary functionality)

#### Automated Testing Coverage

| Tool | Coverage |
|------|----------|
| axe-core | Detects non-interactive elements with handlers missing `role` and `tabindex` |
| eslint-plugin-jsx-a11y | `click-events-have-key-events`, `no-static-element-interactions`, `interactive-supports-focus` |
| vue-a11y | `click-events-have-key-events`, `no-static-element-interactions` |

Manual testing required for: verifying all workflows are completable by keyboard,
testing custom component keyboard interactions.

---

### 2.1.2 No Keyboard Trap (Level A)

If keyboard focus can be moved to a component, focus can be moved away using only
the keyboard. If non-standard keys are needed, the user is advised.

#### Detection Patterns

```regex
# Focus management that may trap
\.focus\(\)
tabIndex\s*=\s*["']?-1
tabindex\s*=\s*["']?-1

# Event listeners that may prevent Tab
e\.preventDefault\(\).*Tab
event\.preventDefault\(\).*Tab
```

#### Violation Examples

```javascript
// Violation: preventing Tab from leaving an element
element.addEventListener('keydown', (e) => {
  if (e.key === 'Tab') {
    e.preventDefault(); // Traps keyboard focus!
  }
});

// Fix: only trap focus in modal dialogs (see ARIA dialog pattern)
// and ensure Escape can close the dialog to release focus
```

#### Severity: **CRITICAL**

#### Automated Testing Coverage

Cannot be reliably detected by automated tools. Requires manual keyboard testing.

---

### 2.1.4 Character Key Shortcuts (Level A)

If keyboard shortcuts use only letter, punctuation, number, or symbol characters,
users can turn them off, remap them, or they only activate when the relevant
component has focus.

#### Detection Patterns

```regex
# Single character keyboard shortcuts
addEventListener\s*\(\s*["']keydown["'][\s\S]*?\.key\s*===?\s*["'][a-zA-Z0-9]["']
# Check: are these only active when the component is focused?
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be automated. Requires manual testing.

---

### 2.2.1 Timing Adjustable (Level A)

Time limits can be turned off, adjusted, or extended (at least 10x the default).

#### Detection Patterns

```regex
# Timers and auto-redirect
setTimeout\s*\(
setInterval\s*\(
<meta\s[^>]*http-equiv\s*=\s*["']refresh["']

# Session timeout
(?i)(session|timeout|expire)\s*[:=]\s*\d+
```

#### Violation Examples

```javascript
// Violation: session timeout with no warning
setTimeout(() => {
  window.location.href = '/logout';
}, 900000); // 15 minutes

// Fix: warn user and allow extension
let timeout;
function resetTimeout() {
  clearTimeout(timeout);
  timeout = setTimeout(() => {
    showTimeoutWarning(); // Give user 60 seconds to extend
  }, 840000); // Warn at 14 minutes
}
```

```html
<!-- Violation: auto-redirect with no user control -->
<meta http-equiv="refresh" content="30; url=/next-page">

<!-- Fix: provide a link instead of auto-redirect -->
<p>Proceed to the <a href="/next-page">next page</a> when ready.</p>
```

#### Severity: **HIGH**

#### Automated Testing Coverage

axe-core detects `<meta http-equiv="refresh">`. JavaScript-based timers require
manual review.

---

### 2.2.2 Pause, Stop, Hide (Level A)

Moving, blinking, scrolling, or auto-updating content can be paused, stopped, or
hidden by the user.

#### Detection Patterns

```regex
# Auto-scrolling / auto-advancing elements
animation\s*:.*infinite
@keyframes
setInterval
requestAnimationFrame
# Check: is there a pause control?

# Carousels / sliders
(?i)(carousel|slider|slideshow|auto-?play|auto-?rotate)
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Limited. axe-core can detect `<marquee>` and `<blink>` elements. JavaScript
animations require manual review.

---

### 2.3.1 Three Flashes or Below Threshold (Level A)

No content flashes more than three times per second, or the flash is below
general flash and red flash thresholds.

#### Detection Patterns

```regex
# Rapid animations
animation-duration\s*:\s*0\.[0-3]s
transition-duration\s*:\s*0\.[0-3]s
# Check: does the animation cause flashing?
```

#### Severity: **CRITICAL** (can cause seizures)

#### Automated Testing Coverage

Cannot be reliably automated. The Photosensitive Epilepsy Analysis Tool (PEAT)
can analyze video content. Code analysis can flag rapid animations for manual review.

---

### 2.4.1 Bypass Blocks (Level A)

A mechanism is available to bypass blocks of content repeated on multiple pages
(skip navigation links).

#### Detection Patterns

```regex
# Skip navigation link
(?i)(skip\s*(to)?\s*(main|content|navigation))
<a\s[^>]*href\s*=\s*["']#(main|content|maincontent)[^"']*["']

# Check: is there a skip link as the first focusable element?
```

#### Violation Examples

```html
<!-- Violation: no skip link -->
<body>
  <nav><!-- 50 navigation links --></nav>
  <main id="main-content">...</main>
</body>

<!-- Fix: add skip link -->
<body>
  <a href="#main-content" class="skip-link">Skip to main content</a>
  <nav><!-- 50 navigation links --></nav>
  <main id="main-content">...</main>
</body>
```

```css
/* Skip link styling: visible on focus */
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  padding: 8px 16px;
  background: #000;
  color: #fff;
  z-index: 100;
}
.skip-link:focus {
  top: 0;
}
```

#### Severity: **HIGH**

#### Automated Testing Coverage

axe-core detects missing skip links (bypass rule). Manual testing needed to verify
the skip link works and is the first focusable element.

---

### 2.4.2 Page Titled (Level A)

Pages have titles that describe topic or purpose.

#### Detection Patterns

```regex
# Missing or empty title
<title>\s*</title>
<title></title>
# Check: does every route/page have a descriptive title?
```

#### Violation Examples

```html
<!-- Violation: generic title -->
<title>App</title>

<!-- Fix: descriptive title -->
<title>Order Summary - Acme Store</title>
```

```jsx
// React: update document title for SPAs
useEffect(() => {
  document.title = `${pageTitle} - Acme Store`;
}, [pageTitle]);

// Or use react-helmet / next/head
<Head><title>Order Summary - Acme Store</title></Head>
```

#### Severity: **LOW** to **MEDIUM**

#### Automated Testing Coverage

axe-core detects missing or empty `<title>` elements. Quality of title text
requires manual review.

---

### 2.4.3 Focus Order (Level A)

Focusable components receive focus in an order that preserves meaning and
operability.

#### Detection Patterns

```regex
# Positive tabindex (almost always wrong)
tabindex\s*=\s*["']?[1-9]
tabIndex\s*=\s*\{?[1-9]

# CSS reordering that may affect focus order
order\s*:\s*-?\d+
flex-direction\s*:\s*row-reverse
```

#### Violation Examples

```html
<!-- Violation: tabindex creates confusing focus order -->
<input tabindex="3" name="email" />
<input tabindex="1" name="name" />
<input tabindex="2" name="phone" />

<!-- Fix: let DOM order determine focus order -->
<input name="name" />
<input name="phone" />
<input name="email" />
```

#### Severity: **HIGH** (positive tabindex) / **MEDIUM** (focus order mismatch from CSS reordering)

#### Automated Testing Coverage

axe-core detects `tabindex` values greater than 0. Focus order verification requires
manual keyboard testing.

---

### 2.4.4 Link Purpose (In Context) (Level A)

The purpose of each link can be determined from the link text alone, or from the
link text together with its programmatically determined context.

#### Detection Patterns

```regex
# Ambiguous link text
(?i)<a\s[^>]*>\s*(click here|here|read more|more|learn more|link|download)\s*</a>
(?i)<a\s[^>]*>\s*(click here|here|read more|more|learn more|link|download)\s*<
```

#### Violation Examples

```html
<!-- Violation: ambiguous link text -->
<p>To view the report, <a href="/report">click here</a>.</p>

<!-- Fix option 1: descriptive link text -->
<p><a href="/report">View the quarterly sales report</a>.</p>

<!-- Fix option 2: use aria-label for context -->
<p>Quarterly sales: <a href="/report" aria-label="View quarterly sales report">read more</a>.</p>
```

```jsx
// Violation: repeated "Learn more" links
{features.map(f => (
  <div key={f.id}>
    <h3>{f.name}</h3>
    <p>{f.description}</p>
    <a href={f.url}>Learn more</a>  {/* All links say "Learn more" */}
  </div>
))}

// Fix: include feature name in link text
{features.map(f => (
  <div key={f.id}>
    <h3>{f.name}</h3>
    <p>{f.description}</p>
    <a href={f.url}>Learn more about {f.name}</a>
  </div>
))}
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

axe-core detects generic link text (`link-name` rule). eslint-plugin-jsx-a11y has
`anchor-is-valid` rule. Quality of link text requires manual review.

---

### 2.4.6 Headings and Labels (Level AA)

Headings and labels describe topic or purpose.

#### Detection Patterns

```regex
# Empty headings
<h[1-6]\s*>\s*</h[1-6]>
<h[1-6]\s[^>]*>\s*</h[1-6]>

# Headings with only icons
<h[1-6]\s*>\s*<(i|span|svg)\s
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

axe-core detects empty headings. Heading quality requires manual review.

---

### 2.4.7 Focus Visible (Level AA)

Any keyboard operable user interface has a mode of operation where the keyboard
focus indicator is visible.

#### Detection Patterns

```regex
# Removing focus outlines
outline\s*:\s*none
outline\s*:\s*0
:focus\s*\{[^}]*outline\s*:\s*(none|0)
\*\s*\{[^}]*outline\s*:\s*(none|0)

# Check: is a custom focus style provided when outline is removed?
```

#### Violation Examples

```css
/* Violation: focus indicator removed globally */
*:focus {
  outline: none;
}

/* Fix: provide visible custom focus style */
*:focus {
  outline: none;
}
*:focus-visible {
  outline: 2px solid #005fcc;
  outline-offset: 2px;
}
```

```css
/* Violation: focus removed on specific elements */
button:focus {
  outline: none;
}

/* Fix: custom focus ring */
button:focus-visible {
  outline: 2px solid #005fcc;
  outline-offset: 2px;
  box-shadow: 0 0 0 4px rgba(0, 95, 204, 0.3);
}
```

#### Severity: **HIGH**

#### Automated Testing Coverage

Static analysis can detect `outline: none` / `outline: 0` in CSS. Verifying that
a replacement focus style exists and is visible requires manual testing.

---

### 2.5.1 Pointer Gestures (Level A)

Functionality that uses multipoint or path-based gestures can also be operated
with a single pointer without a path-based gesture.

#### Detection Patterns

```regex
# Multi-touch gestures
(?i)(pinch|swipe|two.?finger|multi.?touch|gesture)
touchstart.*touchmove
```

#### Severity: **HIGH**

#### Automated Testing Coverage

Cannot be automated. Requires manual testing.

---

### 2.5.2 Pointer Cancellation (Level A)

For functionality operated using a single pointer: the down-event is not used to
execute (use up-event/click instead), or execution can be aborted/undone.

#### Detection Patterns

```regex
# Using mousedown/touchstart for actions (should use click/mouseup)
onmousedown\s*=
ontouchstart\s*=
addEventListener\s*\(\s*["']mousedown["']
addEventListener\s*\(\s*["']touchstart["']
@mousedown\s*=
@touchstart\s*=
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Limited. eslint-plugin-jsx-a11y `no-noninteractive-element-interactions` catches
some cases. Most require manual review.

---

### 2.5.3 Label in Name (Level A)

For UI components with labels that include text or images of text, the accessible
name contains the text that is presented visually.

#### Detection Patterns

```regex
# aria-label that differs from visible text
aria-label\s*=\s*["'][^"']*["']
# Check: does the aria-label include the visible text content?
```

#### Violation Examples

```html
<!-- Violation: aria-label doesn't match visible text -->
<button aria-label="Submit form data">Send</button>

<!-- Fix: include visible text in accessible name -->
<button aria-label="Send">Send</button>
<!-- Or simply: -->
<button>Send</button>
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

axe-core has a `label-content-name-mismatch` rule that detects when the accessible
name does not contain the visible text.

---

### 2.5.4 Motion Actuation (Level A)

Functionality operated by device motion or user motion can also be operated by
UI components and can be disabled.

#### Detection Patterns

```regex
# Device motion events
devicemotion
deviceorientation
accelerometer
gyroscope
```

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be automated. Requires manual review.

---

## Principle 3: Understandable

Information and the operation of the user interface must be understandable.

---

### 3.1.1 Language of Page (Level A)

The default human language of each web page can be programmatically determined.

#### Detection Patterns

```regex
# Missing lang attribute on html element
<html(?!\s[^>]*\blang\s*=)
<html\s[^>]*lang\s*=\s*["']\s*["']
```

#### Violation Examples

```html
<!-- Violation: no lang attribute -->
<html>

<!-- Fix: specify language -->
<html lang="en">
```

```jsx
// React/Next.js
// Violation: _document.js without lang
<Html>

// Fix
<Html lang="en">
```

```vue
<!-- Nuxt: nuxt.config -->
<!-- Violation: missing htmlAttrs -->
export default { head: { title: 'My App' } }

<!-- Fix -->
export default {
  head: {
    htmlAttrs: { lang: 'en' },
    title: 'My App'
  }
}
```

#### Severity: **LOW** to **MEDIUM**

#### Automated Testing Coverage

axe-core reliably detects missing `lang` attribute on `<html>` (`html-has-lang`,
`html-lang-valid` rules).

---

### 3.1.2 Language of Parts (Level AA)

The human language of each passage or phrase can be programmatically determined,
except for proper names, technical terms, or words of indeterminate language.

#### Detection Patterns

```regex
# Inline text in a different language without lang attribute
# Cannot be reliably detected with regex; requires manual review
```

#### Severity: **LOW**

#### Automated Testing Coverage

Cannot be automated. Requires manual review.

---

### 3.2.1 On Focus (Level A)

When a UI component receives focus, it does not initiate a change of context.

#### Detection Patterns

```regex
# Form actions on focus
onfocus\s*=.*submit
onfocus\s*=.*window\.location
onfocus\s*=.*navigate
@focus\s*=.*submit
```

#### Severity: **HIGH**

#### Automated Testing Coverage

Cannot be reliably automated. Requires manual testing.

---

### 3.2.2 On Input (Level A)

Changing the setting of a UI component does not automatically cause a change of
context unless the user has been advised before using the component.

#### Detection Patterns

```regex
# Select/radio that triggers navigation or submit on change
onchange\s*=.*(submit|location|navigate|redirect)
@change\s*=.*(submit|location|navigate|redirect)
onChange\s*=.*\b(submit|navigate|redirect)
```

#### Violation Examples

```html
<!-- Violation: select auto-submits on change -->
<select onchange="this.form.submit()">
  <option>Choose language...</option>
  <option value="en">English</option>
  <option value="es">Spanish</option>
</select>

<!-- Fix: add explicit submit button -->
<select name="language">
  <option>Choose language...</option>
  <option value="en">English</option>
  <option value="es">Spanish</option>
</select>
<button type="submit">Change language</button>
```

#### Severity: **HIGH**

#### Automated Testing Coverage

axe-core detects `onchange` on `<select>` elements that trigger navigation.
Most cases require manual testing.

---

### 3.2.3 Consistent Navigation (Level AA)

Navigational mechanisms repeated on multiple pages occur in the same relative order
each time they are repeated.

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be automated with single-page tools. Requires cross-page comparison.

---

### 3.2.4 Consistent Identification (Level AA)

Components that have the same functionality are identified consistently across pages.

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be automated. Requires cross-page manual comparison.

---

### 3.3.1 Error Identification (Level A)

If an input error is detected, the item in error is identified and the error is
described to the user in text.

#### Detection Patterns

```regex
# Form validation without visible error messages
(?i)(invalid|error|required)\s*[:=]
aria-invalid\s*=\s*["']true["']
# Check: is there a visible error message associated via aria-describedby?

# Error indicated only by color
class\s*=\s*["'][^"']*(error|invalid)[^"']*["']
# Check: is there also error text?
```

#### Violation Examples

```jsx
// Violation: error shown only via styling
<input
  className={hasError ? 'input-error' : 'input'}
  value={email}
  onChange={handleChange}
/>

// Fix: add visible error message and ARIA attributes
<input
  className={hasError ? 'input-error' : 'input'}
  value={email}
  onChange={handleChange}
  aria-invalid={hasError}
  aria-describedby={hasError ? 'email-error' : undefined}
/>
{hasError && (
  <span id="email-error" className="error-text" role="alert">
    Please enter a valid email address.
  </span>
)}
```

#### Severity: **HIGH**

#### Automated Testing Coverage

axe-core detects `aria-invalid` without associated error messages. Full error
handling patterns require manual testing.

---

### 3.3.2 Labels or Instructions (Level A)

Labels or instructions are provided when content requires user input.

#### Detection Patterns

```regex
# Inputs relying on placeholder only
<input\s[^>]*placeholder\s*=\s*["'][^"']+["'][^>]*(?!\baria-label|\blabel)[^>]*>
placeholder\s*=\s*["'][^"']+["'](?![^>]*aria-label)
```

#### Violation Examples

```html
<!-- Violation: placeholder as sole label -->
<input type="email" placeholder="Email address" />

<!-- Fix: visible label with placeholder as supplement -->
<label for="email">Email address</label>
<input type="email" id="email" placeholder="you@example.com" />
```

#### Severity: **HIGH** (missing labels) / **MEDIUM** (placeholder-only labels)

#### Automated Testing Coverage

axe-core detects inputs without labels. eslint-plugin-jsx-a11y detects placeholder
used as sole label.

---

### 3.3.3 Error Suggestion (Level AA)

If an input error is detected and suggestions are known, the suggestions are
provided to the user.

#### Severity: **MEDIUM**

#### Automated Testing Coverage

Cannot be automated. Requires manual review of error messaging logic.

---

### 3.3.4 Error Prevention (Legal, Financial, Data) (Level AA)

For pages that cause legal commitments or financial transactions: submissions are
reversible, data is checked for errors with opportunity to correct, or a confirmation
mechanism is provided.

#### Detection Patterns

```regex
# Financial/legal form submissions
(?i)(payment|checkout|purchase|order|delete|remove|cancel).*submit
(?i)type\s*=\s*["']submit["']
# Check: is there a confirmation step?
```

#### Severity: **HIGH**

#### Automated Testing Coverage

Cannot be automated. Requires manual review of transaction flows.

---

## Principle 4: Robust

Content must be robust enough to be interpreted by a wide variety of user agents,
including assistive technologies.

---

### 4.1.1 Parsing (Level A)

In content implemented using markup languages, elements have complete start and
end tags, are nested according to spec, do not contain duplicate attributes, and
all IDs are unique.

Note: As of WCAG 2.2, this criterion is considered always satisfied for HTML.
However, duplicate IDs remain a practical problem for ARIA references.

#### Detection Patterns

```regex
# Duplicate IDs
id\s*=\s*["']([^"']+)["']
# Check: same ID value appears more than once in the document

# ARIA references to non-existent IDs
aria-labelledby\s*=\s*["']([^"']+)["']
aria-describedby\s*=\s*["']([^"']+)["']
aria-controls\s*=\s*["']([^"']+)["']
aria-owns\s*=\s*["']([^"']+)["']
# Check: referenced IDs exist in the document
```

#### Violation Examples

```html
<!-- Violation: duplicate IDs break aria-labelledby -->
<label id="name-label">Name</label>
<input aria-labelledby="name-label" />

<!-- Later in the document -->
<label id="name-label">Full Name</label> <!-- Duplicate ID! -->
<input aria-labelledby="name-label" /> <!-- Which label? -->
```

```jsx
// Violation: duplicate IDs in React lists
{items.map(item => (
  <div key={item.id}>
    <label id="item-label">{item.name}</label>  {/* Same ID for every item! */}
    <input aria-labelledby="item-label" />
  </div>
))}

// Fix: unique IDs per item
{items.map(item => (
  <div key={item.id}>
    <label id={`item-label-${item.id}`}>{item.name}</label>
    <input aria-labelledby={`item-label-${item.id}`} />
  </div>
))}
```

#### Severity: **HIGH** (duplicate IDs breaking ARIA) / **LOW** (other parsing issues)

#### Automated Testing Coverage

axe-core detects duplicate IDs (`duplicate-id`, `duplicate-id-aria` rules).
HTML validators catch nesting and attribute issues.

---

### 4.1.2 Name, Role, Value (Level A)

For all UI components, the name and role can be programmatically determined; states,
properties, and values that can be set by the user can be programmatically set;
and notification of changes is available to user agents, including assistive tech.

#### Detection Patterns

```regex
# Custom components without roles
<(div|span)\s[^>]*(onclick|onClick|@click)[^>]*(?!\brole\s*=)[^>]*>

# Custom controls without ARIA states
(?i)(toggle|switch|accordion|tab|dropdown|expand|collapse)
# Check: appropriate aria-expanded, aria-selected, aria-checked, etc.

# Dynamic state changes without ARIA updates
setState.*(?!aria)
\.classList\.(add|remove|toggle)
# Check: are ARIA states updated when visual state changes?
```

#### Violation Examples

```html
<!-- Violation: custom toggle without role or state -->
<div class="toggle active" onclick="toggle(this)">Dark Mode</div>

<!-- Fix: proper role and state -->
<button role="switch" aria-checked="true" onclick="toggle(this)">Dark Mode</button>
```

```jsx
// Violation: custom accordion without ARIA
const Accordion = ({ title, children, isOpen, onToggle }) => (
  <div>
    <div className="accordion-header" onClick={onToggle}>{title}</div>
    {isOpen && <div className="accordion-body">{children}</div>}
  </div>
);

// Fix: ARIA attributes and keyboard support
const Accordion = ({ title, children, isOpen, onToggle, id }) => (
  <div>
    <h3>
      <button
        aria-expanded={isOpen}
        aria-controls={`panel-${id}`}
        onClick={onToggle}
      >
        {title}
      </button>
    </h3>
    <div
      id={`panel-${id}`}
      role="region"
      aria-labelledby={`header-${id}`}
      hidden={!isOpen}
    >
      {children}
    </div>
  </div>
);
```

#### Severity: **CRITICAL** (custom interactive elements without roles) / **HIGH** (missing ARIA states)

#### Automated Testing Coverage

| Tool | Coverage |
|------|----------|
| axe-core | Detects missing roles, invalid ARIA attributes, mismatched roles/states |
| eslint-plugin-jsx-a11y | `role-has-required-aria-props`, `role-supports-aria-props` |
| vue-a11y | `role-has-required-aria-props` |

Manual testing required for: verifying dynamic ARIA state updates, confirming
screen reader announcements match visual state.

---

### 4.1.3 Status Messages (Level AA)

Status messages can be programmatically determined through role or properties so
they can be presented to the user by assistive technologies without receiving focus.

#### Detection Patterns

```regex
# Dynamic status/notification elements
(?i)(toast|notification|alert|status|success|error|warning|message|snackbar)
# Check: is there aria-live, role="status", or role="alert"?

# Loading indicators
(?i)(loading|spinner|progress|fetching)
# Check: is loading state announced?
```

#### Violation Examples

```jsx
// Violation: status message not announced
const [saved, setSaved] = useState(false);
// ...
{saved && <span className="success">Settings saved!</span>}

// Fix: use role="status" for non-urgent or role="alert" for urgent
{saved && <span role="status" className="success">Settings saved!</span>}
```

```vue
<!-- Violation: search results count not announced -->
<template>
  <span class="results-count">{{ count }} results found</span>
</template>

<!-- Fix: live region for dynamic count -->
<template>
  <span class="results-count" aria-live="polite" aria-atomic="true">
    {{ count }} results found
  </span>
</template>
```

#### Severity: **MEDIUM** to **HIGH**

#### Automated Testing Coverage

Limited. axe-core can flag elements with dynamic content that lack live region
attributes, but detection depends on the rendering state at scan time. Manual
testing with a screen reader is the most reliable verification.

---

## Form Accessibility Patterns

Forms are one of the most common sources of accessibility failures. This section
consolidates form-specific requirements across multiple success criteria.

### Complete Form Pattern

```html
<!-- Accessible form pattern -->
<form aria-labelledby="form-title" novalidate>
  <h2 id="form-title">Create Account</h2>

  <!-- Required field with description -->
  <div class="field">
    <label for="full-name">Full name <span aria-hidden="true">*</span></label>
    <input
      type="text"
      id="full-name"
      name="full-name"
      autocomplete="name"
      required
      aria-required="true"
      aria-describedby="name-hint"
    />
    <span id="name-hint" class="hint">As it appears on your ID</span>
  </div>

  <!-- Field with error -->
  <div class="field">
    <label for="email">Email <span aria-hidden="true">*</span></label>
    <input
      type="email"
      id="email"
      name="email"
      autocomplete="email"
      required
      aria-required="true"
      aria-invalid="true"
      aria-describedby="email-error"
    />
    <span id="email-error" class="error" role="alert">
      Please enter a valid email address.
    </span>
  </div>

  <!-- Grouped fields -->
  <fieldset>
    <legend>Notification preferences</legend>
    <div>
      <input type="checkbox" id="notify-email" name="notify" value="email" />
      <label for="notify-email">Email notifications</label>
    </div>
    <div>
      <input type="checkbox" id="notify-sms" name="notify" value="sms" />
      <label for="notify-sms">SMS notifications</label>
    </div>
  </fieldset>

  <!-- Error summary at top of form (on submission) -->
  <div role="alert" aria-labelledby="error-summary-title" class="error-summary">
    <h3 id="error-summary-title">There are 2 errors in this form</h3>
    <ul>
      <li><a href="#email">Email: Please enter a valid email address</a></li>
      <li><a href="#password">Password: Must be at least 8 characters</a></li>
    </ul>
  </div>

  <button type="submit">Create Account</button>
</form>
```

### Form Pattern in React/JSX

```jsx
const Form = () => {
  const [errors, setErrors] = useState({});

  return (
    <form aria-labelledby="form-title" noValidate onSubmit={handleSubmit}>
      <h2 id="form-title">Create Account</h2>

      <div className="field">
        <label htmlFor="email">
          Email <span aria-hidden="true">*</span>
        </label>
        <input
          type="email"
          id="email"
          name="email"
          autoComplete="email"
          required
          aria-required="true"
          aria-invalid={!!errors.email}
          aria-describedby={errors.email ? 'email-error' : undefined}
        />
        {errors.email && (
          <span id="email-error" className="error" role="alert">
            {errors.email}
          </span>
        )}
      </div>

      <button type="submit">Create Account</button>
    </form>
  );
};
```

### Form Checklist

| Requirement | SC | Severity |
|------------|-----|----------|
| Every input has a visible `<label>` | 1.3.1, 3.3.2 | HIGH |
| Labels use `for`/`id` or wrapping | 1.3.1 | HIGH |
| Required fields marked with `aria-required` | 3.3.2 | MEDIUM |
| Error messages linked via `aria-describedby` | 3.3.1 | HIGH |
| `aria-invalid` set on fields with errors | 3.3.1 | MEDIUM |
| Error summary with links to fields | 3.3.1 | MEDIUM |
| `autocomplete` on personal data fields | 1.3.5 | LOW-MEDIUM |
| Grouped controls use `<fieldset>`/`<legend>` | 1.3.1 | MEDIUM |
| Error messages describe how to fix the error | 3.3.3 | MEDIUM |
| Confirmation step for irreversible actions | 3.3.4 | HIGH |

---

## Focus Management Patterns

Focus management is critical for SPAs and dynamic content. Incorrect focus
management causes disorientation for screen reader and keyboard users.

### Common Focus Management Scenarios

**Page navigation in SPA**:
```jsx
// Move focus to main heading after route change
useEffect(() => {
  const heading = document.querySelector('h1');
  if (heading) {
    heading.setAttribute('tabindex', '-1');
    heading.focus();
  }
}, [location.pathname]);
```

**Content loaded dynamically**:
```jsx
// Announce new content, move focus if appropriate
const [results, setResults] = useState([]);
const resultsRef = useRef(null);

useEffect(() => {
  if (results.length > 0 && resultsRef.current) {
    resultsRef.current.focus();
  }
}, [results]);

return (
  <div
    ref={resultsRef}
    tabIndex={-1}
    role="region"
    aria-label="Search results"
    aria-live="polite"
  >
    {results.map(r => <ResultItem key={r.id} result={r} />)}
  </div>
);
```

**Element removed from DOM**:
```jsx
// When a focused element is removed, move focus to a logical place
const handleDelete = (itemId) => {
  const items = items.filter(i => i.id !== itemId);
  setItems(items);
  // Move focus to the list or next item
  if (listRef.current) {
    listRef.current.focus();
  }
};
```

### Focus Management Checklist

| Scenario | Focus Target | Severity |
|----------|-------------|----------|
| Modal dialog opens | First focusable element in dialog | CRITICAL |
| Modal dialog closes | Element that triggered the dialog | CRITICAL |
| Page navigation (SPA) | Main heading or main content area | HIGH |
| Inline error on submit | First field with error, or error summary | HIGH |
| Content deleted | Next item, previous item, or parent container | MEDIUM |
| Drawer/panel opens | First focusable element or heading | HIGH |
| Toast appears | Do not move focus (use `aria-live`) | MEDIUM |

---

## Automated Testing Coverage Summary

### What Automated Tools Catch Well

| Category | Success Criteria | Tools |
|----------|-----------------|-------|
| Missing alt text | 1.1.1 | axe-core, pa11y, eslint-plugin-jsx-a11y |
| Missing form labels | 1.3.1, 3.3.2 | axe-core, pa11y, eslint-plugin-jsx-a11y, vue-a11y |
| Color contrast | 1.4.3 | axe-core, pa11y, Lighthouse |
| Missing lang attribute | 3.1.1 | axe-core, pa11y |
| Duplicate IDs | 4.1.1 | axe-core, HTML validators |
| Missing ARIA attributes | 4.1.2 | axe-core, eslint-plugin-jsx-a11y |
| Viewport zoom restriction | 1.4.10 | axe-core |
| Skip link presence | 2.4.1 | axe-core |
| Positive tabindex | 2.4.3 | axe-core, eslint-plugin-jsx-a11y |
| Invalid ARIA usage | 4.1.2 | axe-core, eslint-plugin-jsx-a11y |

### What Requires Manual Testing

| Category | Success Criteria | Testing Method |
|----------|-----------------|----------------|
| Alt text quality | 1.1.1 | Manual review |
| Caption accuracy | 1.2.2 | Manual review |
| Reading order | 1.3.2 | Screen reader testing |
| Color as sole indicator | 1.4.1 | Visual inspection |
| Text resize to 200% | 1.4.4 | Browser zoom testing |
| Reflow at 320px | 1.4.10 | Narrow viewport testing |
| Keyboard operability | 2.1.1 | Full keyboard walkthrough |
| No keyboard trap | 2.1.2 | Keyboard testing |
| Focus order | 2.4.3 | Keyboard tab-through |
| Focus visible | 2.4.7 | Keyboard navigation |
| Consistent navigation | 3.2.3 | Cross-page comparison |
| Error handling | 3.3.1-3.3.4 | Form submission testing |
| Status messages | 4.1.3 | Screen reader testing |
| Focus management | 2.4.3, 4.1.2 | Keyboard + screen reader |

### Recommended CI Configuration

```json
{
  "axe-core": {
    "rules": {
      "color-contrast": { "enabled": true },
      "image-alt": { "enabled": true },
      "label": { "enabled": true },
      "link-name": { "enabled": true },
      "html-has-lang": { "enabled": true },
      "duplicate-id": { "enabled": true },
      "bypass": { "enabled": true },
      "document-title": { "enabled": true }
    },
    "tags": ["wcag2a", "wcag2aa"]
  }
}
```

```json
{
  "pa11y": {
    "standard": "WCAG2AA",
    "runners": ["axe", "htmlcs"],
    "ignore": [],
    "threshold": 0
  }
}
```

---

## Quick-Reference Summary

| SC | Name | Level | Severity | Automatable |
|----|------|-------|----------|-------------|
| 1.1.1 | Non-text Content | A | CRITICAL-LOW | Partial |
| 1.2.1 | Audio/Video-only | A | HIGH | No |
| 1.2.2 | Captions | A | HIGH | Partial |
| 1.2.3 | Audio Description (alt) | A | HIGH | No |
| 1.2.5 | Audio Description | AA | HIGH | No |
| 1.3.1 | Info and Relationships | A | HIGH-MEDIUM | Partial |
| 1.3.2 | Meaningful Sequence | A | MEDIUM | No |
| 1.3.3 | Sensory Characteristics | A | MEDIUM | No |
| 1.3.4 | Orientation | AA | MEDIUM | No |
| 1.3.5 | Identify Input Purpose | AA | LOW-MEDIUM | No |
| 1.4.1 | Use of Color | A | HIGH | Partial |
| 1.4.2 | Audio Control | A | HIGH | Partial |
| 1.4.3 | Contrast (Minimum) | AA | MEDIUM-HIGH | Yes |
| 1.4.4 | Resize Text | AA | MEDIUM | No |
| 1.4.5 | Images of Text | AA | HIGH-LOW | No |
| 1.4.10 | Reflow | AA | MEDIUM-HIGH | Partial |
| 1.4.11 | Non-text Contrast | AA | MEDIUM | Partial |
| 1.4.12 | Text Spacing | AA | MEDIUM | No |
| 1.4.13 | Content on Hover/Focus | AA | MEDIUM | No |
| 2.1.1 | Keyboard | A | CRITICAL-HIGH | Partial |
| 2.1.2 | No Keyboard Trap | A | CRITICAL | No |
| 2.1.4 | Character Key Shortcuts | A | MEDIUM | No |
| 2.2.1 | Timing Adjustable | A | HIGH | Partial |
| 2.2.2 | Pause, Stop, Hide | A | MEDIUM | Partial |
| 2.3.1 | Three Flashes | A | CRITICAL | No |
| 2.4.1 | Bypass Blocks | A | HIGH | Yes |
| 2.4.2 | Page Titled | A | LOW-MEDIUM | Yes |
| 2.4.3 | Focus Order | A | HIGH-MEDIUM | Partial |
| 2.4.4 | Link Purpose | A | MEDIUM | Partial |
| 2.4.6 | Headings and Labels | AA | MEDIUM | Partial |
| 2.4.7 | Focus Visible | AA | HIGH | Partial |
| 2.5.1 | Pointer Gestures | A | HIGH | No |
| 2.5.2 | Pointer Cancellation | A | MEDIUM | Partial |
| 2.5.3 | Label in Name | A | MEDIUM | Yes |
| 2.5.4 | Motion Actuation | A | MEDIUM | No |
| 3.1.1 | Language of Page | A | LOW-MEDIUM | Yes |
| 3.1.2 | Language of Parts | AA | LOW | No |
| 3.2.1 | On Focus | A | HIGH | No |
| 3.2.2 | On Input | A | HIGH | Partial |
| 3.2.3 | Consistent Navigation | AA | MEDIUM | No |
| 3.2.4 | Consistent Identification | AA | MEDIUM | No |
| 3.3.1 | Error Identification | A | HIGH | Partial |
| 3.3.2 | Labels or Instructions | A | HIGH-MEDIUM | Partial |
| 3.3.3 | Error Suggestion | AA | MEDIUM | No |
| 3.3.4 | Error Prevention | AA | HIGH | No |
| 4.1.1 | Parsing | A | HIGH-LOW | Yes |
| 4.1.2 | Name, Role, Value | A | CRITICAL-HIGH | Partial |
| 4.1.3 | Status Messages | AA | MEDIUM-HIGH | Partial |
