# The story of building my deadline tracker (raw notes for a LinkedIn post)

## The problem
I'm a junior CS student and I kept missing or scrambling for assignment
deadlines because my university spreads everything across Colaraz
and Moodle, and I never remember to check either. 
Also in such cases, I have memory of a goldfish. I check WhatsApp
constantly though since often texts arrive and if I am not wrong, Whatsapp
works a lot in Pakistan, and alone as an app it has an open rate of above 90% 
(gotta fact-check from latest reliable source for the figure), so I decided: what if I could just text a bot my
deadlines like I'd text a friend, and have it nag me automatically as they
approach? Not a to-do app I'd have to open — something that lives where I
already am and I receive the notification conveniently. Leveraging an app I already use kinda.

## The stack decision
I built it in n8n (a visual workflow automation tool) connected to
WhatsApp's official Cloud API, an LLM (OpenAI) for understanding my messages, and
Google Sheets as the actual database. I'd never used plugged in a Whatsapp node or API in n8n before this
project — I learned it by building this, which means almost every step
involved hitting a wall I didn't know existed yet. Especially the Meta Developers Space.

## Decision #1: Agent vs. simple pipeline
Early on I had to decide how the "brain" of the bot would work: should I
use an AI Agent node that autonomously calls tools (look up pending tasks,
add a task, mark one done) and reasons through multi-step decisions on its
own? Or should I do one single structured LLM call that returns JSON, and
handle all the branching logic myself with explicit if/else-style nodes?

I actually asked Claude to argue both sides with me. The agent version
sounded cooler and more "AI" — better for a portfolio flex. But I was going
to depend on this thing every single day once the semester started, and I
didn't want to be debugging an opaque agent reasoning trace at 11pm before
a deadline. So I deliberately chose the boring, more reliable option: one
LLM call, strict JSON schema, and a Switch node doing explicit routing.
Slower to build, way easier to debug — and I stand by it, even though it
meant I had to hand-build logic an agent would've done "for free."

## Getting WhatsApp actually connected (the invisible-wall phase)
This is where things got genuinely frustrating before I'd even written any
real logic. Meta's WhatsApp Cloud API doesn't just work once you plug in a
token — I had to:
- Manually subscribe my app to my WhatsApp Business Account via a raw POST
  request in Graph API Explorer, because the setup UI never mentioned this
  step existed. and all Youtube videos I could find were before Meta updated
  their platform, so in my try, it was different navigating.
- Deal with n8n and Meta fighting over "test" vs "production" webhook URLs
  — Meta only allows ONE registered webhook, and it silently flips between
  test and production depending on whether you're using n8n's "listen for
  test event" or actually publishing. I lost real time not understanding
  why my trigger just... stopped receiving messages, before realizing I
  had to republish every time.
- Set up a Meta System User to get a long-lived access token, because the
  default token expires every 24 hours — which would've meant my
  "automation" quietly dying every single day.

None of this was hard, conceptually. It was hard because nothing told me
it was necessary until it broke. And I learned plainly from trial and error.

## Building the actual brain
I wrote a system prompt that classifies every incoming WhatsApp message
into one of four intents — adding a new task, marking something submitted,
answering a question about my deadlines, or "unclear, ask for
clarification" — and returns strict JSON. Getting the LLM node's output
correctly wired into the rest of the workflow was its own multi-round
debugging saga: field paths that didn't match the model's actual response
shape, a Switch node comparing against a literal "0" I forgot to delete,
node names not matching exactly so n8n couldn't find data it needed. A lot
of the debugging was just: something says "undefined," trace backward
through five nodes to find which one silently returned nothing.

## The phantom rows saga (the most maddening bug of the whole build)
This one deserves its own paragraph because it ate hours and made me
genuinely yell at my laptop. I added a checkbox for "Submitted" in Google
Sheets so it'd look clean. Turns out an unchecked checkbox isn't
"empty" — Sheets stores it as a literal FALSE value, even in cells you've
never touched. That meant my workflow's "find the next empty row" logic
kept skipping past hundreds (at one point, a thousand) of "invisible"
pre-filled rows and appending my real data way down at row 1000+. I tried
several increasingly complicated fixes before realizing the actual root
cause and untangling it properly. There was a solid 20 minutes where I was
just typing frustrated all-caps messages at my AI assistant because
nothing made sense and every fix seemed to create a new version of the
same problem.

## Teaching it to handle how I actually text
Once the core pipeline worked, I kept finding gaps between "how the bot
was built" and "how I actually talk":
- I'd text three deadlines in one message ("comp networks, oop, and data
  structures due wednesday") and it would only save one. Had to rebuild
  the schema to return an array of tasks and split them into separate rows.
- I'd say "all done" or "all my quizzes are submitted" and expected
  everything relevant to get marked — but the bot could only handle one
  task at a time. Extended the matching logic to handle "everything," "by
  type," and "by name," with a fallback to "the soonest-due one" if I was
  vague.
- That fix then caused a new bug: marking 5 tasks done in one go fired 5
  duplicate WhatsApp confirmation messages back at me. Had to add a
  collapsing step so multiple sheet updates still produce one clean reply.

## The reminder engine
The second half of the system — the part that actually proactively nags
me — runs on its own schedule, completely separate from the WhatsApp-input
side. It checks my sheet on an interval, and for every pending task
decides: silence if it's more than 2 days out, roughly daily inside that
window, every ~7 hours once inside 24 hours, and exactly ONE reminder if
something goes overdue (so it nags, not tortures). Getting the actual
sending to work required setting up and getting Meta's approval for an
official WhatsApp Message Template, since proactive/business-initiated
messages outside a 24-hour reply window aren't allowed as plain text on
their platform.

## The last debugging twist
Testing this part meant writing fake due dates directly into my sheet and
manually walking through each time-window boundary. I found a properly
sneaky bug this way: some of my hand-typed test times were single-digit
hours ("5:00 PM") and some were double-digit ("12:00 AM") — and the date
parser I was using required strictly zero-padded two-digit hours, so it
silently failed to parse the single-digit ones and skipped those rows
entirely. One test row worked and two didn't, for a reason that had
nothing to do with my actual reminder logic.

## Where I am now
Both halves work end to end: I can text the bot naturally and it logs,
updates, and answers questions about my deadlines, and separately it
proactively reminds me on a schedule with escalating urgency, stopping
once I've submitted something. Built entirely by learning n8n from
scratch, mid-project, through a lot of very real "why is this undefined"
moments.

## Tone notes for whoever turns this into a post
I want the post to feel honest about the struggle, not just a highlight
reel of "look what I built." I want my experience to come through without
sounding falsely humble or overly polished or as if I am shipping a new search
engine. I don;t want to undersell or oversell. Although I would love being concise
and direct because let's be honest, no one is reading an article. I want
it to be a genuine portayal of how I identified a problem and with that
humor of mine, solved it too. P.S, my uni starts in less than 2 weeks so
perfect time to ship.
