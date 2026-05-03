---
name: google-developer-style-guide-voice-tone
description: >-
  Voice and tone rules from the Google developer documentation style guide.
  Covers conversational tone, active voice, second person, politeness, and
  restrictions on pre-announcing features.
---

# Voice & tone

## Tone

**Target:** Conversational, friendly, respectful — like a knowledgeable friend. Casual and natural, not pedantic or pushy. Not overly entertaining, not bone-dry. Be human, be memorable, but prioritize clarity and usefulness.

**Global audience:** Avoid culturally specific references. Keep writing simple and consistent to aid translation and readers with varying English proficiency.

### Things to avoid

- Buzzwords / technical jargon
- Being too cutesy, wacky, zany, or goofy
- Ableist language or figures of speech
- Placeholder phrases: *please note*, *at this time*
- Choppy or long-winded sentences
- Starting all sentences with the same phrase (*You can …*, *To do …*)
- Current pop-culture references
- Exclamation marks (avoid in general)
- Mixing metaphors or taking a metaphor too far
- Phrasing that denigrates or insults any group
- *Let's* do something
- *Simply*, *It's that simple*, *It's easy*, *quickly* in procedures
- Internet slang / abbreviations (*tl;dr*, *ymmv*)

### Techniques

- Step back and ask "What am I trying to say?" — use the plain answer.
- Read aloud; rewrite awkward sentences to be more conversational.
- Use transitions (*Though*, *This way*) but avoid stilted ones (*However*, *Nonetheless*) when they stiffen prose.
- When in doubt, ask a colleague to review.
- Above all, communicate useful information clearly and directly.

### Tone spectrum — examples

| Too informal | Just right | Too formal |
|---|---|---|
| Dude! This API is totally awesome! | This API lets you collect data about what your users like. | The API documented by this page may enable the acquisition of information pertaining to user preferences. |
| Just like a certain pop star, this call gets your *telephone* number. The easy way to ask for someone's digits! | To get the user's phone number, call `user.phoneNumber.get`. | The telephone number can be retrieved by the developer via the simple expedient of using the `get` method on the `user` object's `phoneNumber` property. |
| Then—BOOM—just garbage-collect, and you're golden. | To clean up, call the `collectGarbage` method. | Please note that completion of the task requires the following prerequisite: executing an automated memory management function. |

---

## Politeness & *please*

**Rule:** Do not use *please* in instructions.

| Do | Don't |
|---|---|
| To view the document, click **View**. | To view the document, please click **View**. |
| For more information, see [link]. | For more information, please see [link]. |

---

## Active voice

**Rule:** Use active voice. The grammatical subject performs the action. Make clear who is acting.

| Do (active) | Don't (passive) |
|---|---|
| Send a query to the service. The server sends an acknowledgment. | The service is queried, and an acknowledgment is sent. |
| Send a query to the service. The server sends an acknowledgment. | The service is queried by you, and an acknowledgment is sent by the server. |

### Exceptions — passive voice is acceptable when

| Reason | Example |
|---|---|
| Emphasize an object over an action | The file is saved. |
| De-emphasize a subject / avoid blame | Over 50 conflicts were found in the file. *(not "You created over 50 conflicts …")* |
| Reader doesn't need to know who acted | The database was purged in January. |

---

## Second person & first person

### Address the reader as *you*

**Rule:** Use *you* / *your* (second person), not *we* / *our* / *us* (first person) when addressing the reader. The reader is the person doing tasks or making decisions.

| Do | Don't |
|---|---|
| The following sections describe how you can create a website. | The following sections describe how we can create a website. |
| Consider adding a description to your table. | Let's add a description to our table. |
| This document shows you how to develop an app for your organization. | This document shows the user how to develop an app for their organization. |

**Use *user* only** to refer to the end user of software the reader is building — not as a synonym for the reader.

### Imperative mood

Use the imperative (implied *you*) when telling the reader to do something:

> ✅ Click **Submit**.

After establishing who is addressed, the imperative is acceptable in running text. Consider whether imperative sequences should be formatted as a procedure.

| Do | Don't |
|---|---|
| You can obtain the IP address from your network administrator. Store the address in a variable for future use in the runbook. | To hold the backup data, create a storage bucket. In the Google Cloud console, go to the **Buckets** page. Click **Create bucket**. *(imperative without establishing context)* |

### Third person for software / end users

Use *you* for what the reader does. Use third person for what the software or an end user does. In API docs, state facts about programming elements in third person; address the reader as *you* when telling them what to do.

### First-person plural (*we*, *our*, *us*)

**Allowed** to represent the authoring organization — but the antecedent must be clear.

| Do |
|---|
| Example Organization provides A and B, but we don't provide C and D. |
| For more information, contact our sales organization. |
| The example.org support team regularly reviews tickets. Expect to hear from us in 2-3 business days. |

### Audience consistency

Identify who *you* is (developer? sysadmin? other?) early — ideally with an explicit audience statement near the top of the document — and maintain that consistently.

---

## Future features

**Rule:** Do not document future features or products, even in innocuous ways. Do not pre-announce anything unless approved by legal counsel.

**Related principles:** Use present tense; write timeless documentation.
