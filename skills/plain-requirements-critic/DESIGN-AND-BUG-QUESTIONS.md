# Design and bug questions

Ask these one at a time, each with its example. Ask only the ones no earlier answer covers.

- **Design:** ask these once per screen, on top of the story and the four checks.
- **Bug:** ask these once per bug, instead of the story. Then run the four checks on the fix the person expects.

## Design questions (once per screen)

Open the file or picture. Describe the screen back in plain words first: "I see a page with a list of bookings and a blue *New booking* button." Then ask:

- What does the screen show when **there is nothing yet**? A new client with zero bookings.
- What does it show when there is **a lot**? 500 bookings. A name with 60 letters.
- What does the person see when **something goes wrong**? The payment fails. The internet drops.
- What does the person see **while they wait**? Saving takes five seconds.
- Who **sees this screen**, and does anyone see a different version? A manager versus a receptionist.
- Does it need to work **on a phone**?
- Where does each button **lead**, and what is on that next screen?
- Where does every piece of text and every number **come from**? Who types it, and when?

## Bug questions (once per bug)

- What did you **expect** to happen? What **actually** happened?
- Can you list the **exact steps**, click by click? Example: "Log in as a clinic manager, open Bookings, press Export."
- **Who** is affected? One person, one clinic, everyone?
- **Since when?** Did it ever work?
- Does it happen **every time**, or only sometimes?
- Do people have **a way to get around it** today?
- How **bad** is it? Lost money, a missed appointment, a patient at risk, or only annoying?
- After the fix, what about the people **already affected**? Example: 30 patients never got their email. Should they get it now?
