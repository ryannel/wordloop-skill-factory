# Product Design Guide: Word Loop

**Our Approach:** Word Loop is designed to get out of the user's way. We build applications that are fast, intuitive, and highly responsive. Every design decision should reduce friction and help the user complete their tasks efficiently without asking them to manage the system. The visual aesthetic merges the mathematical rigour of high-density interfaces with a physical, "Liquid Glass" feel.

---

## PART 1: UX CORE PRINCIPLES

### 1. Speed and Interaction
The interface should never slow the user down or make them wait unnecessarily.

* **Focus Inputs Immediately:** Ensure the primary input field is active the moment a screen loads. *The Benefit:* Users can start typing instantly, eliminating start-up friction.
* **Build Frictionless Inline Editing:** Treat content creation and administration as the same action. Clicking a text element should turn it into an active input field right there on the page. *The Benefit:* Keeps the user in context and speeds up data entry.
* **Design Optimistic UIs:** Assume user actions (like saves or updates) will succeed. Update the UI instantly and handle the server synchronisation silently in the background. *The Benefit:* Prevents workflow interruptions and makes Word Loop feel lightning-fast.
* **Implement Instant Resets:** When a user submits an entry, clear the form and reset the UI instantly. *The Benefit:* Allows users to input multiple items consecutively without reaching for the mouse.
* **🚫 What We Avoid:** Creating separate "view" and "edit" screens or modals, or blocking workflows with loading spinners for simple data submissions.

### 2. Navigation and Control
We dedicate maximum screen real estate to the user's content by moving global actions into a central hub.

* **Centralise with a Command Palette:** Use a central palette as the core brain of the application for search, navigation, and settings. *The Benefit:* Frees up screen space and flattens the navigation architecture.
* **Prioritise Keyboard Navigation:** Ensure every major action, navigation loop, and setting is accessible via keyboard shortcuts. *The Benefit:* Massively accelerates workflows for power users.
* **Enable Quick Context Switching:** Allow users to open the Palette, run a command, and close it without losing their place on the current screen. *The Benefit:* Reduces cognitive load when pivoting between distinct tasks.
* **🚫 What We Avoid:** Relying on space-consuming, multi-level dropdown menus for core navigation.

### 3. Information Architecture & Visual Hierarchy
In an app like Word Loop, the user's content is the primary interface. We rely on strict structural logic, typography, and opacity to organise information so the user intuitively knows where they are and what matters most.

* **Content-First Architecture:** The user's data always takes centre stage. System controls, sidebars, and chrome must visually recede into the background. *The Benefit:* Reduces cognitive overload and keeps the user focused entirely on their work.
* **Progressive Disclosure of Complexity:** Start with a sparse, simple view. Reveal advanced tools, metadata, or secondary actions only when the user interacts with or hovers over a specific element. *The Benefit:* Prevents the interface from feeling intimidating or cluttered while maintaining power-user capabilities.
* **Establish Depth Through Opacity:** Create visual hierarchy using background blurring and translucency. Elements closer to the user are opaque; background elements are blurred. *The Benefit:* Creates a clear, intuitive physical space.
* **Use Dynamic Stacking Contexts:** When overlays are invoked, dynamically blur the background further to push the main content deep into the background. *The Benefit:* Narrows the user's focus entirely to the active task.
* **Focal Points via Typography:** Utilise the strict typographic scale to establish what matters most. A clear, high-contrast header anchors the page, while muted text pushes metadata down the hierarchy. *The Benefit:* Guides the user's eye naturally down the page without relying on heavy borders.
* **🚫 What We Avoid:** "Flat" architectures where every piece of data and every button competes equally for attention; deep nesting that forces users to click three or four levels down to find a core feature; and using aggressive drop shadows to fake depth.

### 4. Feedback and Motion
Movement and state changes must feel physical, organic, and calm.

* **Use Liquid Transitions:** Employ organic easing curves so layouts flow smoothly into new configurations rather than snapping instantly.
* **Create Tactile Interactions:** Add very subtle scale shifts when a user hovers over or presses a button.
* **Design Calm Alerts:** Use soft, organic Sage Green for success and a sophisticated Muted Rose for errors. 
* **Implement Progressive Disclosure:** Hide secondary action icons (like edit, delete, or share) until the user hovers over the relevant row.
* **🚫 What We Avoid:** Using harsh, vibrating "stoplight" reds and greens that alarm the user, or cluttering lists with permanently visible action buttons.

### 5. Accessibility (A11y)
Speed and flow mean nothing if a segment of our user base cannot use Word Loop.

* **Ensure High Contrast:** Maintain accessible contrast ratios for all text and critical UI elements, regardless of the active theme.
* **Maintain Visible Focus States:** Provide clear, highly visible focus rings for all interactive elements to support keyboard-only navigation.
* **Support Screen Readers:** Use correct semantic HTML and ARIA labels for all custom components and icons.
* **🚫 What We Avoid:** Relying solely on colour to convey critical information, or removing default browser focus outlines without providing a custom alternative.

### 6. Empty States and Error Handling
We design for the edge cases and the first-time user experience just as rigorously as the primary workflow.

* **Design for Day Zero:** Treat empty states as onboarding opportunities. Provide clear explanations of what belongs in an empty space and a primary call-to-action.
* **Write Actionable Errors:** When an error occurs, clearly state what went wrong and exactly how the user can fix it.
* **Preserve User Input:** If a form submission fails, never erase the data the user just typed.
* **🚫 What We Avoid:** Displaying dead-end "No data found" pages, or showing raw system error codes to the end user.

### 7. Copy and Tone of Voice
Good UX relies just as much on clear writing as it does on layout and interaction.

* **Keep it Concise:** Edit interface copy ruthlessly. Use the fewest words necessary to convey the point.
* **Use Active Voice:** Start calls-to-action with strong verbs (e.g., "Create Project" instead of "New Project can be created").
* **Match the Emotion:** Use a supportive tone for onboarding, and a highly direct, clinical tone for warnings or destructive actions.
* **🚫 What We Avoid:** Using inside jokes, overly casual slang, robotic phrasing, or writing long paragraphs in UI elements.

---

## PART 2: VISUAL DESIGN & CSS STANDARDS

### 1. Colour Architecture (The OKLCH System)
All colours must be defined and manipulated using the **OKLCH** colour space. Hexadecimal, RGB, and HSL are strictly forbidden. To ensure accessible contrast scaling and perceptual uniformity. 

**Dynamic Opacity Injection:** To use these variables with opacity in Tailwind v4 or modern CSS, you must leverage the native CSS `from` syntax:
```css
.card-glass {
  background-color: oklch(from var(--color-bg-secondary) l c h / 0.8);
}
```

| Element | CSS Variable | Obsidian (Dark) OKLCH | Milk (Light) OKLCH |
| :--- | :--- | :--- | :--- |
| **Canvas** | `--color-bg-primary` | `0.15 0.01 285` | `0.985 0.005 285` |
| **Surface** | `--color-bg-secondary` | `0.18 0.01 285` | `1 0 0` |
| **Primary Text**| `--color-text-primary` | `0.95 0.01 285` | `0.12 0.01 285` |
| **Muted Text** | `--color-text-secondary`| `0.45 0.02 285` | `0.45 0.02 285` |
| **Accent** | `--color-accent` | `0.75 0.15 300` | `0.75 0.15 300` |
| **Success** | `--color-success` | `0.70 0.10 140` | `0.70 0.10 140` |
| **Error** | `--color-error` | `0.60 0.12 20` | `0.60 0.12 20` |

### 2. Typography (Geist)
Typography utilises the **Geist** font stack. All sizing is executed in `rem` units (`1rem = 16px`). As size increases, tracking (letter-spacing) decreases to maintain optical tightness.

| Level | Size | Weight | Line Height | Tracking | Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Display Header** | `3.5rem` | `600` | `1.0` | `-0.025em` | Hero sections, main view titles. |
| **Section Header** | `2.0rem` | `600` | `1.2` | `-0.025em` | Modal headers, bento card titles. |
| **Body Primary** | `1.0rem` | `400` | `1.5` | `-0.011em` | Standard paragraphs, descriptions. |
| **UI Control** | `0.875rem`| `500` | `1.2` | `-0.011em` | Buttons, navigation links, tabs. |
| **Micro Data** | `0.6875rem`| `500` | `1.2` | `0.05em` | Metadata, badges (always uppercase). |

### 3. Spatial Architecture & Layout
All layouts adhere to an **8-point base grid** to prevent sub-pixel rendering blur on high-density displays.
* **Sub-Grid:** `4px` (`0.25rem`) 
* **Base Step:** `8px` (`0.5rem`) 
* **Standard Gap:** `16px` (`1.0rem`)
* **Macro Gap:** `32px` (`2.0rem`)

**The Bento Grid:** The structural arrangement of components relies heavily on compartmentalised cards adhering to a 12-column CSS Grid. Visual hierarchy is established by the surface area a card occupies.

```css
.bento-grid {
  display: grid;
  gap: 1rem;
  grid-template-columns: repeat(12, minmax(0, 1fr));
  grid-auto-rows: 90px; /* Establishes strict vertical rhythm */
}
```

### 4. Surface Dynamics (Depth & Elevation)
Because the colour palette is heavily constrained, Word Loop relies on optical illusions of depth to separate cards from the background canvas.

* **The Subtle Border Illusion:** Standard borders appear harsh in dark mode. Define edges using heavily transparent white strokes (`oklch(1 0 0 / 0.08)`) to simulate a microscopic physical bevel.
* **Concentric Corner Radii:** Excessive rounding feels consumer-centric; tighter rounding conveys engineering precision. Outer cards use `8px` radii. Internal nested elements use `4px`. **Mathematical Rule:** Inner Radius = Outer Radius - Padding.
* **Multi-Layered Drop Shadows:** Single-layer shadows fail to mimic real-world lighting. Use a 4-layer stack to simulate depth:
    1. *Ambient Occlusion:* Grounding shadow where the card meets the canvas.
    2. *Direct Shadow:* Primary cast light.
    3. *Soft Penumbra:* The blurring edge of the shadow.
    4. *Diffuse Scatter:* Ambient light scatter across the canvas.
* **Inset Lighting:** Always pair a shadow stack with an inset top rim light (`inset 0 1px 0 oklch(1 0 0 / 0.05)`) to simulate overhead lighting hitting the physical edge of the glass.

### 5. Ambient Mesh Textures
To break the starkness of large bento cards, employ "Ambient Mesh Textures" (radial gradients). 
* **No Hard Edges:** Gradients must fade to `transparent` using percentages (`50%` to `70%`) to prevent visual banding.
* **Layering:** The base surface colour (`var(--bg-secondary)`) must always be declared last as the solid fallback layer beneath the translucent gradients.

---

## PART 3: LIQUID GLASS COMPONENT EXECUTION

Do not mix background blurs arbitrarily. To ensure distinct roles, each card uses specific lighting physics built from the surface dynamics outlined above.

### 1. The Standard Glass Surface
**Role:** The architectural baseline. It holds data without competing for attention. It rests comfortably within the grid without distorting the background heavily.

```css
.glass-surface {
  background: var(--glass-bg) !important; /* Base 65% opacity */
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  
  border: 1px solid var(--glass-border) !important; /* 8% white stroke */
  border-radius: 0.5rem; /* 8px rounding */

  /* Standard 4-layer depth stack + top rim light */
  box-shadow:
    0 2px 4px var(--glass-shadow-1),   /* Ambient Occlusion */
    0 4px 8px var(--glass-shadow-2),   /* Direct Shadow */
    0 8px 16px var(--glass-shadow-3),  /* Penumbra */
    0 16px 32px var(--glass-shadow-4), /* Diffuse Scatter */
    inset 0 1px 0 var(--glass-highlight) !important; /* Standard Rim Light */
}
```

### 2. The Elevated Overlay
**Role:** High-priority interruption (Command Palette, Modals). It pulls the user completely out of the bento grid. Uses a heavy blur, a deeper shadow, and a brighter rim light to simulate floating closer to overhead light.

```css
.glass-elevated {
  background: var(--glass-bg) !important;
  backdrop-filter: blur(20px); /* Heavy distortion */
  -webkit-backdrop-filter: blur(20px);
  
  border: 1px solid var(--glass-border) !important;
  border-radius: 0.75rem; /* Slightly larger 12px radius */

  /* High-elevation, wide-scatter shadow stack */
  box-shadow:
    0 4px 8px var(--glass-shadow-1),
    0 8px 16px var(--glass-shadow-2),
    0 16px 32px var(--glass-shadow-3),
    0 24px 48px var(--glass-shadow-4),
    inset 0 1px 0 oklch(1 0 0 / 0.10) !important; /* Brighter Rim Light */
}
```

### 3. The Hero Insight Card
**Role:** The primary atmospheric anchor. It should feel sweeping and premium. Uses a **Diagonal Aurora** mesh texture to create dynamic visual tension.

```css
.card-insight {
  background:
    /* Top-Left Purple wash */
    radial-gradient(120% 120% at 0% 0%, oklch(from var(--glow-insight-1) l c h / 0.25) 0%, transparent 70%),
    /* Bottom-Right Magenta wash */
    radial-gradient(120% 120% at 100% 100%, oklch(from var(--glow-insight-2) l c h / 0.20) 0%, transparent 70%),
    /* Solid base layer */
    var(--bg-secondary) !important;

  border: 1px solid var(--glass-border) !important;
  border-radius: 0.5rem;

  /* Standard shadow stack */
  box-shadow:
    0 2px 4px var(--glass-shadow-1),
    0 4px 8px var(--glass-shadow-2),
    0 8px 16px var(--glass-shadow-3),
    0 16px 32px var(--glass-shadow-4),
    inset 0 1px 0 var(--glass-highlight) !important;
}
```

### 4. The Action Required Card
**Role:** Drive immediate interaction. Uses a **Top-Down Spotlight** to visually guide the user's eye downwards toward the action or metric.

```css
.card-accent-glow {
  background:
    /* Top-Centre Spotlight wash fading quickly at 50% */
    radial-gradient(100% 80% at 50% 0%, oklch(from var(--glow-accent) l c h / 0.25) 0%, transparent 50%),
    var(--bg-secondary) !important;

  border: 1px solid var(--glass-border) !important;
  border-radius: 0.5rem;

  /* Standard shadow stack */
  box-shadow:
    0 2px 4px var(--glass-shadow-1),
    0 4px 8px var(--glass-shadow-2),
    0 8px 16px var(--glass-shadow-3),
    0 16px 32px var(--glass-shadow-4),
    inset 0 1px 0 var(--glass-highlight) !important;
}
```

### 5. The Positive State Card
**Role:** Celebrate completion. It needs to feel restful and grounded. Uses a **Bottom-Up Ambient** glow, resolving tension by drawing the eye to the base of the card.

```css
.card-success-glow {
  background:
    /* Bottom-Centre Grounded wash */
    radial-gradient(120% 100% at 50% 100%, oklch(from var(--glow-success) l c h / 0.20) 0%, transparent 60%),
    var(--bg-secondary) !important;

  border: 1px solid var(--glass-border) !important;
  border-radius: 0.5rem;

  /* Standard shadow stack */
  box-shadow:
    0 2px 4px var(--glass-shadow-1),
    0 4px 8px var(--glass-shadow-2),
    0 8px 16px var(--glass-shadow-3),
    0 16px 32px var(--glass-shadow-4),
    inset 0 1px 0 var(--glass-highlight) !important;
}
```