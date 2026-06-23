# BUILD_SPEC.md — "Python, One Day at a Time"

> Hand this whole file to Claude Code. Instruction to give it:
> **"Read BUILD_SPEC.md and build the entire project exactly to spec. Create `index.html`, `.github/workflows/daily-reminder.yml`, and `README.md`. Commit when done."**

---

## 1. What we're building

A tiny, beautiful, beginner-friendly site that teaches **one Python concept a day** with hands-on practice, hosted on **GitHub Pages**. A daily **WhatsApp reminder** (via a scheduled GitHub Action + CallMeBot) nudges the learner to open the site, where they choose to *review yesterday* or *learn today's concept*.

Audience: a complete beginner, new to IT. Tone everywhere: warm, encouraging, never intimidating. Keep each lesson to **one idea, not too much**.

## 2. Repo structure (exactly these files)

```
/index.html                          # the entire app, self-contained
/.github/workflows/daily-reminder.yml # the daily WhatsApp nudge
/README.md                           # short setup notes
/BUILD_SPEC.md                       # this file (leave it in the repo)
```

## 3. Hard technical constraints

- **One self-contained `index.html`.** No build step, no bundler, no framework. Plain HTML + CSS + vanilla JS, all in one file.
- Only two external sources allowed: **Google Fonts** (Fraunces, Inter, JetBrains Mono) and the **Pyodide CDN** (official jsDelivr). Nothing else.
- Must run correctly when opened directly and when served from GitHub Pages.
- **Progress persists via `localStorage`**, wrapped in `try/catch` with an in-memory fallback so it never throws if storage is unavailable.
- Mobile-first. Must look and work great on a phone.
- Accessibility floor: visible keyboard focus, AA text contrast, 44px min tap targets, `prefers-reduced-motion` respected, the code editor is a real `<textarea>`.

## 4. Design system (cozy-dark editorial)

A warm dark theme — softer than pure black so it's friendly for a nervous beginner. The signature motif is the Python REPL prompt `>>>` and a real, live terminal as the hero of every lesson.

**Fonts**
- Display / headings: **Fraunces** (weights 400 & 600; prefer a high optical size for warmth).
- Body / UI: **Inter** (400/500/600).
- Code / terminal: **JetBrains Mono** (400/500).

**Color tokens (use CSS variables):**
```css
--bg:        #15131E;  /* deep plum-charcoal page background */
--bg-soft:   #1E1B2E;  /* lifted cards / panels */
--bg-code:   #0E0D16;  /* the terminal/editor, darker than bg */
--ink:       #ECEAF5;  /* primary text */
--ink-dim:   #A29DB8;  /* muted lavender-grey */
--line:      rgba(255,255,255,0.08); /* hairline borders */
--accent:    #3DDC97;  /* fresh mint-green — the single accent (nods to the owner's teal brand, brightened for dark-bg legibility) */
--accent-ink:#08291C;  /* dark text on accent buttons */
--flame:     #FFB020;  /* amber — ONLY for the streak flame, the one playful spark */
--error:     #FF7B72;  /* soft coral for tracebacks / wrong answers */
```

**Shape & spacing:** cards radius 16px, buttons 12px, code panel 14px. Generous padding. On dark, create depth with the lifted `--bg-soft` + faint `--line` borders and very soft shadows — avoid heavy drop shadows.

**Motion (tasteful, CSS only):**
- Screen transitions: ~220ms fade + slight `translateY(6px)`.
- Terminal output lines fade in as printed.
- The streak flame has a gentle ambient flicker (subtle opacity/scale keyframe).
- On completing a lesson: a drawn SVG checkmark (stroke animation) + the day's dot fills with `--accent`.
- All of the above disabled under `prefers-reduced-motion: reduce`.

**Signature elements:**
- `>>>` as the brand mark in the header (JetBrains Mono, `--accent`).
- The greeting is prompt-styled: `>>> Hey {name}`.
- The Run button reads **`Run ▸`** and is styled like a prompt action.
- A calm **30-day journey strip**: a row/grid of 30 small dots that fill with `--accent` as lessons complete. The amber streak flame + streak number sit beside it as the only playful flourish.

**Copy rules:** sentence case, plain verbs, no filler. Buttons say exactly what happens ("Start learning", "Mark complete", "Got it"). Empty/failure states give direction, not mood (e.g. a wrong answer shows the hint, not an apology).

## 5. Screens & flow

A single-page app with JS-rendered screens. One thing on screen at a time.

### 5.1 Welcome (first visit only — no saved name)
- Fraunces headline: **"Learn Python, one day at a time."**
- One-line subtext: learn a single concept a day, with real code you run yourself.
- A text input: **"What should I call you?"** + button **"Start learning →"**.
- On submit: save `name` + `startDate`, go to Home.

### 5.2 Home / choice (returning visit)
- Header: `>>>` brand mark (left); streak flame + count and "Day N of 30" (right).
- Greeting: `>>> Hey {name}`.
- Two large choice cards:
  1. **Warm up** — "Quick review of Day {prevDay}: {title}". Only show if at least one lesson is completed; otherwise show it disabled with subtext "Finish your first lesson to unlock reviews."
  2. **Today's concept** — "Day {todayDay}: {title}" → opens that lesson.
- Below the cards: the 30-day journey strip + "`{completed} done · 🔥 {streak} day streak`".
- If `lastDate === today`: show a gentle banner above the cards — "You've done today's concept 🎉 Come back tomorrow — or keep exploring below." (Do **not** block; an eager learner may continue to the next lesson.)
- If all 30 complete: Home becomes a celebration — "You finished all 30 🎉" — with the journey strip fully lit and the ability to open any lesson to review.

### 5.3 Lesson
- Back arrow → Home.
- Eyebrow "Day N" + emoji + title (Fraunces).
- **Idea**: the explanation text, readable, with inline `code` spans where useful.
- **Example** (only if the lesson has one): a small code block. If `example.runnable` is true, show a `Run ▸` button and a terminal showing its output. If false (predict-only days), render it as static highlighted code — **do not load Pyodide**.
- **Practice** (see §6 for per-type behavior): `predict`, `run`, or `both`.
- Footer: **"Mark complete ✓"**. On a `run`/`both` lesson it enables after a successful run; the learner may also proceed via the success state. On `predict` it enables after answering. Completing returns to Home with the checkmark + dot-fill + streak update animation.

### 5.4 Review (the "warm up" path)
- Compact 30-second refresher card: "Day {prevDay}: {title}", the one-line `review` text, and the lesson's example code (read-only; runnable only if it was a runnable example). Button **"Got it →"** returns to Home.

### 5.5 Footer / settings (small, every screen)
- A small **"Reset progress"** link (with a confirm dialog) that clears saved state and returns to Welcome. For restarting / testing.

## 6. State, storage & streak logic

Store everything under a single localStorage key `pyjourney_v1` as JSON:
```js
{ name: string, startDate: "YYYY-MM-DD", completed: number[], streak: number, lastDate: "YYYY-MM-DD" | null }
```
Wrap all storage access in `try/catch`; if it throws, fall back to an in-memory object for the session.

Derived values:
- `todayDay` = smallest day in 1..30 **not** in `completed` (or `null` if all done).
- `prevDay` = `completed.length ? Math.max(...completed) : null`.

`completeDay(day)`:
1. If `day` already in `completed`, just return Home (no streak change).
2. Otherwise push `day` into `completed`.
3. Streak update (once per calendar day):
   - if `lastDate === todayStr()` → leave streak unchanged;
   - else if `lastDate === yesterdayStr()` → `streak += 1`;
   - else → `streak = 1`.
4. Set `lastDate = todayStr()`. Save.

Use **local** dates (`YYYY-MM-DD`) for `todayStr()` / `yesterdayStr()`.

## 7. Practice behavior by type

**`predict`** (no code execution, instant — never loads Pyodide):
- Show the code snippet (static, highlighted), the `question`, and the `options` as tappable buttons.
- On tap: mark the chosen option correct/incorrect (accent for correct, coral for wrong), reveal the `explain` text, then show **"Continue"** → enables Mark complete.

**`run`** (real Python via Pyodide):
- Show the `prompt`, a code **editor** (`<textarea>` prefilled with `starter`, JetBrains Mono), a **`Run ▸`** button, and a **terminal** output area.
- On Run: execute the code (see §8), print stdout/stderr into the terminal, then evaluate the `check` (see §9). On pass: success state (green check + encouraging line, e.g. "Nice — that's it!"). On fail: show the `hint` in the terminal area, let them edit and retry.

**`both`**:
- Run the `predict` step first. After "Continue", reveal the `run` editor. Success on the run step completes the lesson.

## 8. Pyodide integration

- Include the **latest stable Pyodide** from the official jsDelivr CDN. Use the current version's URL pattern: `https://cdn.jsdelivr.net/pyodide/v<latest>/full/pyodide.js`, with `indexURL` pointing at the same folder. (Pick the current stable version at build time.)
- **Lazy-load**: call `loadPyodide()` only on the **first** `Run ▸` click anywhere — never on page load, never on predict-only lessons. While loading, show a status like "Booting Python… (a few seconds the first time)" and disable Run. Cache the single instance for the rest of the session.
- **Run routine** for user code:
  1. Reset the target terminal element.
  2. Expose an input bridge on the window:
     ```js
     window._agInput = (msg) => { const r = window.prompt(msg || "Input:"); return r === null ? "" : r; };
     ```
  3. Set batched stdout/stderr handlers that append to the terminal (stderr in `--error` color).
  4. Prepend this preamble so `input()` works in the browser, then run the user's code with `runPythonAsync`:
     ```python
     import builtins as _b
     from js import _agInput as _ag
     _b.input = lambda *a: str(_ag(a[0] if a else ""))
     ```
  5. Wrap in `try/catch`; on error, print the message/traceback to the terminal in `--error` color (do not crash the page).
- **Graceful failure**: if `loadPyodide` is undefined or loading fails, print a friendly note in the terminal: *"Couldn't start Python here. Once this site is live on GitHub Pages the code runner works. (If you're previewing inside another app, that may be blocking it.)"* — everything else on the page must still work.

## 9. Answer checker (for `run` lessons)

Implement a small checker that runs the user's code, captures **stdout** (as `out`), and evaluates a declarative `check` object. A lesson passes only if **all** specified conditions are true:

- `producedOutput: true` → `out.trim().length > 0`
- `contains: ["a","b"]` → `out` (case-insensitive) includes each string
- `minLines: N` → `out.trim()` split on `\n`, count of non-empty lines `>= N`
- `noPlaceholder: true` → `out` does **not** contain `"___"`
- `containsDigit: true` → `/\d/.test(out)`

On pass → success state. On fail → show the lesson's `hint`. Be lenient and encouraging; never block retries.

## 10. Lesson data (the 30-day curriculum)

Transcribe this array faithfully into `index.html`. Fields:
`day, emoji, title, idea, example ({code, note, runnable} or null), practice ({type, predict?, run?}), review`.
Predict-only days (13, 16, 17, 25) have `example: null` and never load Pyodide. "Both" days (4, 15, 19, 23, 28) have a predict step then a run step.

```js
const LESSONS = [
  {
    day: 1, emoji: "👋", title: "Saying hello with print()",
    idea: "Every program starts by telling the computer to do something. `print()` shows text on the screen — whatever you put inside the quotes, Python shows back to you. Think of it as Python repeating after you.",
    example: { code: `print("Hello, world!")`, note: "Press Run to see what happens.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Make Python introduce you. Put your own name inside the quotes.",
      starter: `print("Hi, I'm ___")`,
      check: { producedOutput: true, noPlaceholder: true },
      hint: "Replace the ___ (keep the quotes) with your name, then press Run."
    }},
    review: "print() shows text on screen. The text goes inside quotes."
  },
  {
    day: 2, emoji: "💬", title: "Printing more than one line",
    idea: "You can print as many lines as you like. Python runs your code top to bottom, one line at a time, and each `print()` starts a new line.",
    example: { code: `print("Line one")\nprint("Line two")`, note: "Two prints, two lines.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Print three things about yourself — one per line (your name, your city, your favourite food).",
      starter: `print("My name is ___")\nprint("I live in ___")\nprint("I love ___")`,
      check: { minLines: 3 },
      hint: "Add a print() line for each thing. Three print lines make three lines of output."
    }},
    review: "Code runs top to bottom. Each print() is a new line."
  },
  {
    day: 3, emoji: "📦", title: "Variables: naming a value",
    idea: "A variable is a labelled box that holds a value so you can use it later by name. `name = \"Sam\"` puts the text Sam into a box called name. After that, name means \"Sam\".",
    example: { code: `name = "Sam"\nprint(name)`, note: "Store it, then print it.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Make a variable called city holding your city, then print it.",
      starter: `city = "___"\nprint(city)`,
      check: { producedOutput: true, noPlaceholder: true },
      hint: "Put your city in the quotes; keep print(city) as it is."
    }},
    review: "A variable is a named box. name = value stores it; use the name to get it back."
  },
  {
    day: 4, emoji: "✏️", title: "Changing a variable",
    idea: "A variable can change. Whatever you store last is what it holds — like erasing the box and putting something new in.",
    example: { code: `score = 0\nscore = 10\nprint(score)`, note: "The second value replaces the first.", runnable: true },
    practice: { type: "both",
      predict: {
        code: `mood = "tired"\nmood = "happy"\nprint(mood)`,
        question: "What does this print?",
        options: ["tired", "happy", "tired happy", "Error"],
        answer: "happy",
        explain: "The last value you store wins, so mood is \"happy\" by the time it prints."
      },
      run: {
        prompt: "Now try it yourself: make mood \"tired\", then change it to \"happy\", then print it.",
        starter: `mood = "tired"\nmood = "happy"\nprint(mood)`,
        check: { contains: ["happy"] },
        hint: "Store \"tired\" first, then \"happy\", then print(mood)."
      }
    },
    review: "Re-assigning replaces the value. The last assignment is what's stored."
  },
  {
    day: 5, emoji: "🔤", title: "Text (strings)",
    idea: "Text in Python is called a string — characters inside quotes, like \"hello\". The quotes tell Python \"this is text, not a command\".",
    example: { code: `greeting = "Good morning!"\nprint(greeting)`, note: "A whole sentence is fine.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Store your favourite saying in a variable called saying, and print it.",
      starter: `saying = "___"\nprint(saying)`,
      check: { producedOutput: true, noPlaceholder: true },
      hint: "Anything inside the quotes works — a full sentence too."
    }},
    review: "A string is text inside quotes. Quotes mark where the text starts and ends."
  },
  {
    day: 6, emoji: "🔢", title: "Numbers",
    idea: "Python knows numbers too, and numbers need no quotes. Whole numbers like 7 are ints; numbers with a dot like 3.5 are floats. You can do math with `+ - * /`.",
    example: { code: `print(2 + 3)\nprint(10 / 4)`, note: "Let Python do the math.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Print the answer to 15 times 8 — let Python calculate it, don't type the answer.",
      starter: `print(15 * 8)`,
      check: { contains: ["120"] },
      hint: "Use * to multiply: print(15 * 8)."
    }},
    review: "Numbers need no quotes. Use + - * / for math. Whole = int, decimal = float."
  },
  {
    day: 7, emoji: "🧮", title: "Math with variables",
    idea: "Variables that hold numbers can be used in math, just like the numbers themselves. This is how programs calculate things.",
    example: { code: `price = 5\nquantity = 3\ntotal = price * quantity\nprint(total)`, note: "Store the result, then use it.", runnable: true },
    practice: { type: "run", run: {
      prompt: "You buy 4 items at 12 each. Use variables to calculate and print the total.",
      starter: `items = 4\ncost = 12\ntotal = ___\nprint(total)`,
      check: { contains: ["48"] },
      hint: "total = items * cost, then print(total)."
    }},
    review: "Number variables work in math. Store the result in a new variable, then use it."
  },
  {
    day: 8, emoji: "🔗", title: "Joining text together",
    idea: "Stick strings together with `+`. This is called joining. \"Hello, \" + \"Sam\" becomes \"Hello, Sam\".",
    example: { code: `first = "Ada"\nprint("Hello, " + first)`, note: "Mind the space after the comma.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Make a variable name with your name, then print \"Welcome, \" joined with your name.",
      starter: `name = "___"\nprint("Welcome, " + name)`,
      check: { contains: ["welcome"], noPlaceholder: true },
      hint: "Use + to join: \"Welcome, \" + name. Keep the space after the comma."
    }},
    review: "+ joins strings. Numbers add, but strings glue together."
  },
  {
    day: 9, emoji: "✨", title: "f-strings (text + values, the easy way)",
    idea: "Put an `f` before the quotes and you can drop variables right inside the text using {curly braces}. Much easier than +. `f\"Hi {name}\"` fills in the name automatically.",
    example: { code: `name = "Lee"\nage = 20\nprint(f"{name} is {age}")`, note: "Braces get replaced by the values.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Use an f-string to print \"I am [your name] and I love [something]\" with two variables filled in.",
      starter: `name = "___"\nlikes = "___"\nprint(f"I am {name} and I love {likes}")`,
      check: { producedOutput: true, noPlaceholder: true },
      hint: "Keep the f before the quotes, and put variable names inside {curly braces}."
    }},
    review: "f\"...\" lets you put {variables} inside text. Cleaner than joining with +."
  },
  {
    day: 10, emoji: "⌨️", title: "Asking the user a question",
    idea: "`input()` pauses and waits for the person to type something, then hands you what they typed (always as text).",
    example: { code: `name = input("What's your name? ")\nprint(f"Hi {name}!")`, note: "A box will pop up — type and press OK.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Ask the user for their favourite colour, then print \"Nice, [colour] is a great colour!\"",
      starter: `colour = input("Favourite colour? ")\nprint(f"Nice, {colour} is a great colour!")`,
      check: { producedOutput: true },
      hint: "Use input(\"...\") to ask, then an f-string to reply. Type an answer in the popup."
    }},
    review: "input() waits for the user to type, and gives you their text."
  },
  {
    day: 11, emoji: "🔁", title: "Turning input into numbers",
    idea: "`input()` always gives text — even if the person types 5, you get the string \"5\", which you can't do math with. Wrap it in `int()` to turn it into a real number.",
    example: { code: `age = int(input("Your age? "))\nprint(f"Next year you'll be {age + 1}")`, note: "int() makes it a number.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Ask for a number, then print that number doubled.",
      starter: `n = int(input("Pick a number: "))\nprint(n * 2)`,
      check: { producedOutput: true, containsDigit: true },
      hint: "Wrap input in int(): int(input(...)). Then print(n * 2)."
    }},
    review: "input() gives text. int(...) turns \"5\" into 5 so you can do math."
  },
  {
    day: 12, emoji: "✅", title: "True and False (booleans)",
    idea: "Some values are just yes or no — Python calls them `True` and `False` (capital first letter, no quotes). They're the foundation of decisions.",
    example: { code: `is_raining = True\nprint(is_raining)`, note: "No quotes around True.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Make a variable is_weekend, set it to True, then print it.",
      starter: `is_weekend = True\nprint(is_weekend)`,
      check: { contains: ["true"] },
      hint: "True has a capital T and no quotes."
    }},
    review: "True / False are booleans — yes/no values. No quotes, capital first letter."
  },
  {
    day: 13, emoji: "⚖️", title: "Comparing things",
    idea: "Python can compare values and answer with True/False. `==` means equal to (two equals!), and you also have `>`, `<`, `>=`, `<=`, `!=` (not equal).",
    example: null,
    practice: { type: "predict", predict: {
      code: `print(7 > 12)\nprint(4 == 4)`,
      question: "What does this print?",
      options: ["False\nTrue", "True\nFalse", "True\nTrue", "False\nFalse"],
      answer: "False\nTrue",
      explain: "7 is not greater than 12 → False. 4 equals 4 → True."
    }},
    review: "Comparisons give True/False. == is equal (two!), != is not equal."
  },
  {
    day: 14, emoji: "🔀", title: "Making decisions with if",
    idea: "`if` lets your program choose. If the condition is True, the indented code under it runs; if not, it's skipped. The indent (spaces) is how Python knows what's inside the if.",
    example: { code: `age = 20\nif age >= 18:\n    print("You're an adult")`, note: "Note the colon and the indent.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Set temperature = 35. If it's greater than 30, print \"It's hot!\"",
      starter: `temperature = 35\nif temperature > 30:\n    print("It's hot!")`,
      check: { contains: ["hot"] },
      hint: "End the if line with a colon, and indent the print under it (4 spaces)."
    }},
    review: "if condition: runs the indented block only when the condition is True. Indentation matters."
  },
  {
    day: 15, emoji: "🔂", title: "if / else",
    idea: "`else` is the \"otherwise\" partner of `if`. If the condition is False, the else block runs instead. One branch or the other always runs.",
    example: { code: `age = 12\nif age >= 18:\n    print("Adult")\nelse:\n    print("Not yet")`, note: "else has no condition.", runnable: true },
    practice: { type: "both",
      predict: {
        code: `score = 40\nif score >= 50:\n    print("Pass")\nelse:\n    print("Try again")`,
        question: "What does this print?",
        options: ["Pass", "Try again", "Both", "Nothing"],
        answer: "Try again",
        explain: "40 is not >= 50, so the if is False and the else runs."
      },
      run: {
        prompt: "Now write it: set score = 40. If score >= 50 print \"Pass\", otherwise print \"Try again\".",
        starter: `score = 40\nif score >= 50:\n    print("Pass")\nelse:\n    print("Try again")`,
        check: { contains: ["try again"] },
        hint: "else: has no condition. Indent its print the same as the if's print."
      }
    },
    review: "else runs when the if is False. Exactly one branch runs."
  },
  {
    day: 16, emoji: "🪜", title: "elif (more than two choices)",
    idea: "`elif` (\"else if\") checks another condition when the first was False. You can chain several. Python tries them top to bottom and runs the first True one.",
    example: null,
    practice: { type: "predict", predict: {
      code: `light = "yellow"\nif light == "red":\n    print("Stop")\nelif light == "green":\n    print("Go")\nelse:\n    print("Slow down")`,
      question: "What does this print?",
      options: ["Stop", "Go", "Slow down", "Nothing"],
      answer: "Slow down",
      explain: "light isn't \"red\" or \"green\", so both checks fail and the else runs."
    }},
    review: "elif adds more conditions between if and else. The first True one wins."
  },
  {
    day: 17, emoji: "🔗", title: "and / or / not",
    idea: "Combine conditions: `and` needs both to be True; `or` needs at least one; `not` flips True/False. Just like everyday logic: \"sunny and warm\".",
    example: null,
    practice: { type: "predict", predict: {
      code: `temp = 22\nif temp > 15 and temp < 30:\n    print("Nice day")`,
      question: "What does this print?",
      options: ["Nice day", "Nothing", "True", "Error"],
      answer: "Nice day",
      explain: "Both parts are True (22 is above 15 and below 30), so 'and' is True and it prints."
    }},
    review: "and = both true, or = at least one true, not = flip it."
  },
  {
    day: 18, emoji: "📋", title: "Lists (a collection)",
    idea: "A list holds many values in order, inside square brackets, separated by commas — like a shopping list stored in one variable.",
    example: { code: `fruits = ["apple", "banana", "cherry"]\nprint(fruits)`, note: "One variable, many values.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Make a list called hobbies with three of your hobbies, and print it.",
      starter: `hobbies = ["___", "___", "___"]\nprint(hobbies)`,
      check: { contains: ["["], noPlaceholder: true },
      hint: "Put each item in quotes, separated by commas, inside [ ]."
    }},
    review: "A list = [ ] holds many values in order, separated by commas."
  },
  {
    day: 19, emoji: "🔢", title: "Getting items out of a list",
    idea: "Each item has a position number called an index, and Python starts counting at 0. So `list[0]` is the first item, `list[1]` the second.",
    example: { code: `colors = ["red", "green", "blue"]\nprint(colors[0])\nprint(colors[2])`, note: "Counting starts at 0.", runnable: true },
    practice: { type: "both",
      predict: {
        code: `colors = ["red", "green", "blue"]\nprint(colors[0])`,
        question: "What does this print?",
        options: ["red", "green", "blue", "0"],
        answer: "red",
        explain: "Index 0 is the first item, which is \"red\"."
      },
      run: {
        prompt: "Make a list of three animals and print only the second one.",
        starter: `animals = ["cat", "dog", "fox"]\nprint(animals[___])`,
        check: { producedOutput: true, noPlaceholder: true },
        hint: "Counting starts at 0, so the second item is index 1: animals[1]."
      }
    },
    review: "Items have positions starting at 0. list[0] is first, list[1] is second."
  },
  {
    day: 20, emoji: "➕", title: "Adding to a list",
    idea: "`.append()` adds an item to the end of a list. Lists can grow and shrink while your program runs.",
    example: { code: `tasks = ["wash", "cook"]\ntasks.append("clean")\nprint(tasks)`, note: "\"clean\" gets added to the end.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Start with shopping = [\"milk\", \"eggs\"], append \"bread\", then print the list.",
      starter: `shopping = ["milk", "eggs"]\nshopping.append("bread")\nprint(shopping)`,
      check: { contains: ["bread"] },
      hint: ".append(\"bread\") adds it to the end, then print the list."
    }},
    review: ".append() adds an item to the end of a list. Lists can change."
  },
  {
    day: 21, emoji: "📏", title: "How many? len()",
    idea: "`len()` tells you how many items are in a list (or how many characters in a string). Handy for counting.",
    example: { code: `names = ["Sam", "Lee", "Kai"]\nprint(len(names))`, note: "Three items → 3.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Make a list of your favourite movies and print how many you listed.",
      starter: `movies = ["___", "___"]\nprint(len(movies))`,
      check: { containsDigit: true, noPlaceholder: true },
      hint: "len(movies) gives the count. Add as many movies as you like."
    }},
    review: "len(x) counts items in a list (or characters in a string)."
  },
  {
    day: 22, emoji: "🔁", title: "Looping through a list (for)",
    idea: "A `for` loop repeats code once for each item in a list, handing you one item at a time. No more copy-pasting print() for every item.",
    example: { code: `for fruit in ["apple", "banana"]:\n    print(fruit)`, note: "The body runs once per item.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Loop through a list of three friends and print \"Hi\" and each name.",
      starter: `friends = ["Ana", "Ben", "Cy"]\nfor name in friends:\n    print(f"Hi {name}")`,
      check: { minLines: 3 },
      hint: "Indent the print inside the loop; it runs once per name."
    }},
    review: "for item in list: runs the indented block once for each item."
  },
  {
    day: 23, emoji: "#️⃣", title: "Counting with range()",
    idea: "`range(n)` gives the numbers 0 up to (but not including) n. Great for repeating something a set number of times.",
    example: { code: `for i in range(3):\n    print(i)`, note: "0, 1, 2 — not 3.", runnable: true },
    practice: { type: "both",
      predict: {
        code: `for i in range(3):\n    print(i)`,
        question: "What does this print?",
        options: ["0, 1, 2 (each on its own line)", "1, 2, 3", "0, 1, 2, 3", "3"],
        answer: "0, 1, 2 (each on its own line)",
        explain: "range(3) gives 0, 1, 2 — it starts at 0 and stops before 3."
      },
      run: {
        prompt: "Use range to print \"Hello\" five times.",
        starter: `for i in range(5):\n    print("Hello")`,
        check: { minLines: 5 },
        hint: "range(5) loops 5 times. Print \"Hello\" inside the loop."
      }
    },
    review: "range(n) counts 0 to n-1. Use it to repeat a set number of times."
  },
  {
    day: 24, emoji: "🔄", title: "while loops",
    idea: "A `while` loop keeps repeating as long as its condition stays True — useful when you don't know how many times in advance. Make sure something changes, or it never stops.",
    example: { code: `count = 1\nwhile count <= 3:\n    print(count)\n    count = count + 1`, note: "count grows until the condition fails.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Print the numbers 1 to 5 using a while loop.",
      starter: `n = 1\nwhile n <= 5:\n    print(n)\n    n = n + 1`,
      check: { contains: ["5"] },
      hint: "Increase n each time (n = n + 1) so the loop eventually stops."
    }},
    review: "while condition: repeats while it's True. Change something inside or it loops forever."
  },
  {
    day: 25, emoji: "🛑", title: "break (stopping a loop early)",
    idea: "Inside a loop, `break` stops it completely. (Its partner `continue` skips to the next round.) They give you fine control over loops.",
    example: null,
    practice: { type: "predict", predict: {
      code: `for n in range(10):\n    if n == 5:\n        break\n    print(n)`,
      question: "What's the LAST number printed?",
      options: ["4", "5", "9", "10"],
      answer: "4",
      explain: "When n becomes 5, break stops the loop before printing, so 4 is the last number shown."
    }},
    review: "break stops a loop; continue skips to the next iteration."
  },
  {
    day: 26, emoji: "🛠️", title: "Functions: your own command",
    idea: "A function is a reusable chunk of code with a name. You define it once with `def`, then call it by name whenever you need it — like saving a recipe to reuse.",
    example: { code: `def greet():\n    print("Hello!")\n\ngreet()`, note: "Define, then call it.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Define a function called cheer that prints \"Go team!\", then call it.",
      starter: `def cheer():\n    print("Go team!")\n\ncheer()`,
      check: { contains: ["go team"] },
      hint: "Define with def cheer():, indent the print, then call it on a new line: cheer()."
    }},
    review: "def name(): defines a function. Call it with name() to run it."
  },
  {
    day: 27, emoji: "🎁", title: "Functions with inputs (parameters)",
    idea: "Functions can take inputs, called parameters, so they work with different values each time. `def greet(name)` lets you greet anyone.",
    example: { code: `def greet(name):\n    print(f"Hi {name}!")\n\ngreet("Sam")\ngreet("Lee")`, note: "Same function, different values.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Write a function welcome(place) that prints \"Welcome to [place]\", and call it with your city.",
      starter: `def welcome(place):\n    print(f"Welcome to {place}")\n\nwelcome("___")`,
      check: { contains: ["welcome to"], noPlaceholder: true },
      hint: "Put place in the (parentheses), use it inside, then call welcome(\"YourCity\")."
    }},
    review: "Parameters let a function work with different values: def f(x): ... then f(5)."
  },
  {
    day: 28, emoji: "↩️", title: "Functions that give back a value",
    idea: "`return` sends a value back out of a function so you can use the result. Unlike print (which only shows it), return hands it to you to store or reuse.",
    example: { code: `def add(a, b):\n    return a + b\n\nresult = add(2, 3)\nprint(result)`, note: "return gives the answer back.", runnable: true },
    practice: { type: "both",
      predict: {
        code: `def double(n):\n    return n * 2\n\nprint(double(8))`,
        question: "What does this print?",
        options: ["16", "8", "double(8)", "Error"],
        answer: "16",
        explain: "double(8) returns 8 * 2 = 16, and print shows it."
      },
      run: {
        prompt: "Write a function double(n) that returns n times 2, then print double(8).",
        starter: `def double(n):\n    return ___\n\nprint(double(8))`,
        check: { contains: ["16"] },
        hint: "return n * 2. Then double(8) gives 16, which print shows."
      }
    },
    review: "return hands a value back. Store it (x = f()) or print it."
  },
  {
    day: 29, emoji: "📒", title: "Dictionaries (labels and values)",
    idea: "A dictionary stores pairs: a key (a label) and its value. You look things up by name instead of by position — like a real dictionary: word → meaning.",
    example: { code: `person = {"name": "Ada", "age": 30}\nprint(person["name"])`, note: "Look up by key.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Make a dictionary car with keys \"brand\" and \"year\", then print the brand.",
      starter: `car = {"brand": "___", "year": ___}\nprint(car["brand"])`,
      check: { producedOutput: true, noPlaceholder: true },
      hint: "Use { \"key\": value }. Look up with car[\"brand\"]."
    }},
    review: "A dict = { key: value } pairs. Get a value with dict[\"key\"]."
  },
  {
    day: 30, emoji: "🚀", title: "Your first mini program",
    idea: "Time to combine everything — variables, input, f-strings and a decision make a real program. You've learned the building blocks; here they work together.",
    example: { code: `name = input("Your name? ")\nage = int(input("Your age? "))\nif age >= 18:\n    print(f"{name}, you can vote!")\nelse:\n    print(f"{name}, {18 - age} years to go.")`, note: "Everything so far, working together.", runnable: true },
    practice: { type: "run", run: {
      prompt: "Build a tiny program: ask for a name and one more thing, then print a friendly message using both. Make it yours — there's no wrong answer here.",
      starter: `name = input("Your name? ")\nfav = input("Favourite thing? ")\nprint(f"Nice to meet you {name}! {fav} is awesome.")`,
      check: { producedOutput: true },
      hint: "Use input() to ask, and an f-string to build your message."
    }},
    review: "Real programs combine variables, input, decisions and functions. You can build them now."
  }
];
```

## 11. The daily WhatsApp reminder — `.github/workflows/daily-reminder.yml`

A scheduled GitHub Action that sends one WhatsApp message a day via **CallMeBot**, plus a manual trigger for testing and a light monthly keep-alive so the schedule isn't auto-disabled.

It reads these from the repo (the owner sets them in **Settings → Secrets and variables → Actions**):
- **Secret** `CALLMEBOT_APIKEY` — the API key the learner received from CallMeBot.
- **Secret** `LEARNER_PHONE` — the learner's number with country code (e.g. `+1404...`).
- **Variable** `LEARNER_NAME` — the learner's first name.
- **Variable** `SITE_URL` — the live GitHub Pages URL (filled in after Pages is enabled).

Create the file exactly like this:

```yaml
name: Daily Python reminder

on:
  schedule:
    # 22:00 UTC ≈ 6:00 PM in Atlanta during EDT (summer).
    # Cron is always UTC and does NOT follow daylight saving, so in winter (EST)
    # this lands at 5:00 PM. Adjust the hour here to taste.
    - cron: "0 22 * * *"
  workflow_dispatch: {}        # lets the owner run a test from the Actions tab

permissions:
  contents: write              # needed only for the monthly keep-alive commit

jobs:
  remind:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Send WhatsApp reminder via CallMeBot
        env:
          PHONE: ${{ secrets.LEARNER_PHONE }}
          APIKEY: ${{ secrets.CALLMEBOT_APIKEY }}
          NAME: ${{ vars.LEARNER_NAME }}
          URL: ${{ vars.SITE_URL }}
        run: |
          MESSAGE="Hey ${NAME:-there}! 👋 Time for today's Python concept. Open your lab: ${URL} — warm up with a quick review of yesterday, or jump into today's new idea. 5 minutes, one concept. You've got this 💪"
          curl -sS -G "https://api.callmebot.com/whatsapp.php" \
            --data-urlencode "phone=${PHONE}" \
            --data-urlencode "text=${MESSAGE}" \
            --data-urlencode "apikey=${APIKEY}"

      - name: Monthly keep-alive (prevents 60-day auto-disable)
        run: |
          if [ "$(date -u +%d)" = "01" ]; then
            git config user.name "github-actions"
            git config user.email "actions@github.com"
            date -u > .last-reminder
            git add .last-reminder
            git commit -m "keep-alive: $(date -u)" || echo "nothing to commit"
            git push || echo "push skipped"
          else
            echo "Not the 1st — skipping keep-alive commit."
          fi
```

Notes for the README: GitHub's scheduled runs can be delayed by several minutes at busy times, which is fine for a daily nudge. The free CallMeBot tier only delivers to the number that authorised it, so the learner must do the one-time activation on their own phone.

## 12. `README.md` (write something like this)

A short README covering: what the project is; that the site is one self-contained `index.html` you can open locally or serve via GitHub Pages; how to deploy (Settings → Pages → deploy from `main`, root); how the reminder works and the four secrets/variables to set; how to change the daily time (edit the cron line) and the lesson list (edit the `LESSONS` array); and a one-line note that the code runner uses Pyodide (real Python in the browser).

## 13. Definition of done (check each)

1. Opening `index.html` shows the Welcome screen on first load; entering a name proceeds to Home.
2. Home shows the greeting, both choice cards (Warm up disabled until a lesson is done), the 30-day strip, and the streak.
3. "Today's concept" opens the correct next lesson; completing it fills its dot, advances `todayDay`, and updates the streak.
4. Predict lessons (13, 16, 17, 25) work with tappable options + explanation and **never load Pyodide**.
5. Run lessons load Pyodide on first Run, execute real Python, capture output, and evaluate the check with the right pass/fail + hint.
6. `input()` works in run lessons (via the prompt bridge).
7. "Both" lessons (4, 15, 19, 23, 28) do predict → then reveal the editor.
8. Progress survives a page refresh (localStorage); "Reset progress" clears it.
9. Looks polished on a phone; keyboard focus visible; reduced-motion respected.
10. If Pyodide can't load, the page still works and shows the friendly fallback message.
11. The workflow file is valid YAML, sends via CallMeBot using the four secrets/variables, supports manual `workflow_dispatch`, and includes the monthly keep-alive.
12. README explains deployment, the reminder, and how to change the time and lessons.
