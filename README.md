# ChatSafetyAI

- Request your free demo [here](https://safetyapp.shinyapps.io/chatsafetyai_demo_access_request/) !
- For our SafetyAI Council Members employees, request your access [here](https://safetyapp.shinyapps.io/chatsafetyai_access_request/) !
- Help us improve ChatSafetyAI by taking this [quick survey](https://forms.gle/E6sK87u7DdoL4Kg86) !

## About
- ChatSafetyAI is built with care by Antoine Tixier at [SafetyAI](https://www.safetyai.io) 🛠️❤️.
- ChatSafetyAI relies on the [OpenAI API](https://platform.openai.com/docs/api-reference/introduction).
- OpenAI [does not](https://platform.openai.com/docs/models/how-we-use-your-data) use the conversations for training their models, and does not store the conversations permanently.
- Screencast demos (not necessarily up-to-date) can be found on our [website](https://www.safetyai.io).

Table of Contents
- [Usage Tiers](https://github.com/safetyAI/ChatSafetyAI/blob/main/README.md#usage-tiers)
- [Version History](https://github.com/safetyAI/ChatSafetyAI/blob/main/README.md#version-history)
- [Contact](https://github.com/safetyAI/ChatSafetyAI/blob/main/README.md#contact)
- [Terms of Use](https://github.com/safetyAI/ChatSafetyAI/blob/main/README.md#terms-of-use)

## Usage Tiers
There is a free [demo](https://safetyapp.shinyapps.io/chatsafetyai_demo_access_request/) with a limited budget, just for quick testing purposes.<br>
Contact us for membership pricing (see contact info at the bottom of this page, or on our [website](https://www.safetyai.io)).<br>
Note: we do not currently offer plans for individuals, SafetyAI membership is for companies only.

<table border=1>
<tr> <th>  </th> <th> Demo </th> <th> SafetyAI Council </th>  </tr>
  <tr> <td align="right"> Engine </td> <td> GPT-4.1, GPT-4.1-mini </td> <td>  GPT-4.1, GPT-4.1-mini </td> </tr>
  <tr> <td align="right"> Conversation History </td> <td> 750k words, 11 turns </td> <td> 750k words, 19 turns </td> </tr>
  <tr> <td align="right"> Max Chats per Session </td> <td> 5 </td> <td> Unlimited </td> </tr>
  <tr> <td align="right"> File Upload </td> <td> max ~110 pages, 10 MB per file </td> <td> max ~890 pages, 20 MB per file </td> </tr>
  <tr> <td align="right"> Budget per Session </td> <td> 300k words </td> <td> Unlimited (within company budget) </td> </tr>
  <tr> <td align="right"> Budget per User </td> <td> 1.5M words </td> <td> Unlimited (within company budget) </td> </tr>
  <tr> <td align="right"> Custom Databases </td> <td> 3 of 10 files max </td> <td> 10 of 100 files max </td> </tr>
  <tr> <td align="right"> Custom Predictive Models </td> <td> No </td> <td> Yes </td> </tr>
  <tr> <td align="right"> Custom Risk Analysis </td> <td> No </td> <td> Yes </td> </tr>
   </table>


## Version History

### Next steps / wish list (not in order, non-exhaustive)
- [ ] incognito mode
- [ ] ability to search the web
- [ ] chat sharing as individual icons  
- [ ] image passed to every call  
- [ ] search engine for chats  
- [ ] input multiple files at once  
- [ ] voice input and output  
- [ ] generate PDF out of anything  
- [ ] image and table width on mobile  
- [ ] image generation  
- [ ] bounding boxes for hazards on photos    

### 10-06-25
- first version of custom database functionality
- SaaS database fully migrated to MS Azure Storage
- priority OpenAI calls for faster responses
- instructions now consume 50% less tokens
- other small improvements

### 09-10-25
- custom database functionality 75% done
- env variable to hide custom database tab and logic while it is still under development

### 09-03-25
- env variable to check user budgets (disabled if not defined)
- env variable to record logs and save them to the database

### 08-16-25
- env variable to disable keepalive JS
- link to terms of use in avatar popup
- raised file size limit to 20 MB

### 08-08-25
- files streamed from new utilities endpoint with managed identities in Azure
- reset rating emojis when switching conv

### 08-05-25
- support for Hindi, Arabic, Portuguese
- support for .heic images
- rating (thumbs up/down) each response, with integration in dashboard

### 07-16-25
- managed identities

### 07-14-25
- new ENV VARs, CHECK_LOCK_FILE and MAX_INACTIVE_TIME

### 07-08-25
- bug fix pillow's image save when 4 channels

### 06-11-25
- Graham's internal documentation (SOPs, guidelines, standards)
- NYC's TBTA agency regulations
- Support for Azure OpenAI in SaaS (faster), tested but not yet activated

### 05-11-25
- Invisible handling of automatic scroll-down
- Allowing file drag and drop in full conversation container
- Greater streaming output speed

### 05-08-25
- Using GPT-4.1
- Increased context size, number of turns, and file size before truncation
- Improved suggestion instructions
- Only using two API keys

### 04-04-25
- logs to debug scaling and load balancing on Azure

### 03-16-25
- bug fix introductory text attribute detection
- fixed Azure paths
- favicon
- company email address update in user messages

### 03-07-25
- azure support for metadata, and new metada (mode counters)
- fixed unwanted line breaks in regulation mode output

### 03-03-25
- support for deployment relying 100% on Azure services
- better handling of unknown language in mode instructions ("cannot tell") to avoid "sorry, I can't help" responses
- pagination in Dropbox file listing function to support directories containing many files
- generated PDFs and presentations now have a unique ID and contain the chat index, so can be deleted w/ the conv
- sensitive info, and is_local, now passed as environment variables
- greater budgets for demo users

### 12-19-24
- fixed bug in XLSX file scraping function
- made PPTX and DOCX file scraping functions more robust

### 12-04-24
- Better description of the SCL model and criteria for high-energy
- Support for new file types: DOCX, XLSX, PPTX (MS Word, Excel, and PowerPoint files)
- Removed dependency on `rdrop2` for communication with the database
- Better wrapping of file content to avoid any confusion with user query, clarified that SCL has nothing to do with safety climate in function description and general instructions (to address TCE bug report)

### 11-10-24
- Image and text anonymization: Face and logo blurred in photos; names of people, places, and organizations redacted in text (optional, TRUE by default).
- LLM switch for vision things, to fix the "Sorry, I can't help with this request" issue with GPT.
  
### 10-18-24
- C3 and D3 JS libraries now local (attempt to fix MIME type issues for Atkins)
- favicon attempt (turns out it works locally but not on shinyapps.io)

### 09-16-24
- fixed vertical space avatar badge mobile
- fixed budget bars in avatar modal (demo)
- added link to survey in avatar modal

### 09-10-24
- initial hazard identification step before HECA call with minimal instructions, to benefit from unconstrained, unbiased eye. Hazards passed to mode thru conversation history.
- refined HECA instructions
- function instructions now reject HECA for photos not depicting job site situations
- knowledge of acronyms to off-topic classifier and better instructions
- no green light for chat deletion while a response is being generated

### 09-03-24
- edited regulation and prediction's function descriptions to reduce false starts on calls
- internal hazard listing in addition to description upon image upload, to improve HECA mode

### 08-30-24
- saving user data after each turn to make onSessionEnded only handle unlocking and cleaning
- disabled parallel function calling

### 08-29-24
- exponential backoff for file deletion
- upload flag if unlocking failed
- regulation mode now aware of original query (enabling compound questions)

### 08-26-24
- take conversation into account for adversarial classifier too
- show reason for rejection in off-topic and adversarial classifiers
- session termination after some time of inactivity
- file upload check on session termination

### 08-22-24
- gpt-4o as the main engine and for the vision microservice
- regulations: improved scraping and chunking, especially for tables, equations, and long chunks. Also now using an approach with larger chunks in combination with the original small-chunk approach, to better balance precision and recall, and because some documents just cannot afford to be split.
- regulations: slightly more neighbors and slightly lower similarity threshold, to improve recall
- regulations: new instructions with unconstrained output format, to better suit generation scenarios. This allows the regulation mode to be more than just a question answering mode (information retrieval with fixed output format), which was too limited.
- regulations: added OSHA 1910 (general industry)
- chat history and log out
- all files saved to database and images embedded into the chats via URLs
- wrappers to gracefully handle Dropbox API errors and retry
- shorter greetings to save time and budget
- bug fix consumption measurement in build_context.R (regulations)
- bug fix name to url mapping for long documents (regulations, original approach for OSHA)
- more budget for CSV and PDF file uploads in terms of amount of text taken into account before truncation
- improved off-topic and adversarial classifers, to reduce false alarms on both text queries and file uploads

### 05-12-24
- support for Canada Occupational Health and Safety Regulations (COHSR) for federally regulated companies

### 05-10-24
- improved "HECA from photo" mode (knowledge, instructions, conversation-awareness, special instructions as optional function call argument)
- knowledge about the SCL model and a few online resources / tools
- EC2 APIs
- bug fix progress bar titles in Demo
- budget increases

### 04-07-24
- updating total number of conversations file on session end
- one additional example on desktops (7 instead of 6)
- added examples in "misc" category
- start to improve HECA knowledge and instructions (work in progress)

### 04-05-24
- fix word wrapping in tables
- fix long URL regulation mode in tables

### 04-02-24
- URLs and long words now wrap in tables
- Bug fix OSHA reference table in regulations mode for non-English languages

### 03-29-24
- private app, public demo
- routines now reply in the language of the original query (except, only in the GPT-4 case, those relying on the `[START]` and `[END]` special tokens for non-English languages, due to a bug: https://community.openai.com/t/gpt-4-1106-preview-is-not-generating-utf-8/482839)
- support for non-English headlines
- refactoring of adversarial and off-topic classifiers to reduce false alarms, especially in non-English languages
- bug fix bad function call handling for follow-up question suggestions
- use of the `0125` GPT installments for follow-up question suggestions, to generate proper accentuated chars in Spanish and French (related to bug above)
- refactoring of supported language classifier to return language instead of just yes/no
- more comprehensive cost measurement
- knowledge of time on session start, with time zone (previously was just date, w/o time zone)
- bug fix: handling tokens that are not character (e.g., list, when fake function call)

### 03-22-24
- Saskatchewan regulations
- regulation embeddings upgrade (`text-embedding-3-large` from Jan 25, 2024)
- query expansion to improve semantic search results

### 03-20-24
- message when file size exceeds max allowed size
- max allowed file size increased from 2 MB to 5 MB

### 03-17-24
- automatically close sidebar on mobile devices when tapping any sidebar button (new chat, switch chats, toggle suggestions, etc.)
- bug fix: hitting enter now just submits query without modifying text. (Note that shift + enter can be used)
- cost recording for file inputs (including photos)

### 03-02-24
- conversation sharing functionality
- errors during file checking and trucation are now handled
- stringi's `stri_read_lines` now used instead of base's `readLines` to increase robustness to encoding issues when reading CSV files
- Graceful handling of API errors for regulations, predictions, attributes
- Model aware of risk assessment and its meaning
- Minimalized info message at the bottom, expanded README (this page)
- Removed "click to populate" from Examples title
- fixed typo in one example
- blue border now does not extend to the right of the fileInput element when drag-and-dropping
- loader when generating follow-up questions

### 02-16-24
- Image, CSV, and PDF inputs.
- File upload area allowing browse and drag and drop
- New "HECA" image mode.
- Now using `gpt-4-vision-preview`
- `gpt-3.5-turbo-0125` and `gpt-4-0125-preview` were tested but not used for now due to breaking changes
- Now using full possible context of 128k everywhere.
- Now 10 turns (9 real, 1 instructions) instead of 7
- NSFW classifiers for text and images
- Content file checking: adversarial, topic, language, file content truncation
- Off-topic classifier for images, handling of API refusals
- Rounded corners for message areas
- UI improvement: placement and alignment of text bar and buttons
- bug fix adversarial input classifier

### 01-16-24
- Expanded knowledge about direct controls, HECA, and the energy method

### 01-15-24
- First version of "prejob safety brief and paperwork filling" mode
- Prejob mode examples
- Number of examples adapts to screen width
- Improved off-topic classifier to reduce false alarms
- Improved presentation instructions to increase figures' inclusion rate
- Replaced traffic safety example by fire prevention example (presentation mode)

### 12-28-23
- Fixed error vectors in if statements of NLP tool (new version of R issues error instead of warning)
- NLP API error handling in `get_attributes.R`

### 12-26-23
- improved presentation instructions
- global knowledge in presentation instructions
- better sanitization of presentation Markdown code (handle triple backquotes)
- added one example of the presentation category
- shorter bottom info message
  
### 12-25-23
- enforce the sampling of examples from different categories (covering different modes)

### 12-24-23
- fixed 'New chat' button placement for mobile devices

### 12-23-23
- regulations: added support for Alberta, British Columbia, Manitoba, New Brunswick, Newfound Land and Labrador, Nova Scotia, Ontario, and Quebec. 
- adapted instructions to prevent Markdown formatting issues
- multiple API keys to increase usage limits
- ability to generate a slideset summary out of any conversation
- now aware of today's date
- adapted instructions and strategy of several modes to halve processing time
- UI improvements (placement and/or appearance of buttons, scrollbar, tables, search bar...)
- ability for the user to disable suggestions of follow-up questions
- ability for the user to disable autoscroll
- better formatting of tables
- plenty of red teaming and legit example testing
- proper handling of line breaks by sanitizing and HTML display functions
- off-topic classifier takes previous turns as context to reduce false alarms
- classifiers have better instructions
- redesign of adversarial classifier
- classifiers do not send green flag when query fails
- mode automatically detected for each query with function calling (taking conversation into account, capable of retrieving arguments across earlier messages). Modes can now be switched within the same chat, removing the need for mode-specific classifiers, and improving acceptance of follow-up questions.
- prettify now handles markdown tables and is called during generation
- updated message at the bottom of the page
- SHIFT + ENTER in text area now does not trigger
- textArea and scrollbar styles consistent with rest of page
- discussion thread slightly wider
- examples now randomly selected from a large pool at the beginning of each chat
- new safety climate survey analysis mode
- calling the latest generic models in prediction mode (SUPER-COMPANY 2023 update)
- showing the latest generic risk assessment in prediction mode (SUPER-COMPANY 2023 update)
- now relying on GPT-4-Turbo from 11/06 (bleeding-edge version)
- follow-up questions now suggested via function calling
- fixed scroll down issues
- added link to changelog (this page)
- reduced user budget and max nb of threads in single session to account for the increased cost of GPT-4
- global parameter to enable privacy selection
- `prettify_text` now handles basic markdown tags
- logo now links to website
- improved several instruction files
- injected knowledge about energy wheel and HECA 

### 07-25-23
- OSHA and prediction modes
- tab names are now automatically renamed as extreme summaries of their conversations
- now using GPT-3.5-turbo 16k

### 05-12-23
- initial release

## Contact
Bug reports, questions: `antoine DOT tixier AT safetyai .com`

# Terms of Use

These Terms of Use ("Terms") govern your use of ChatSafetyAI, along with any associated models, APIs, software applications, websites, and related services (collectively, the **"Services"**). These Terms form a binding agreement between you and SafetyAI, Inc., a Colorado corporation ("**SafetyAI**," "**we**," "**our**," or "**us**"). By accessing or using our Services, you agree to be bound by these Terms.

---

## 1. Who We Are

ChatSafetyAI is developed by SafetyAI, Inc., a Colorado-based company. Our mission is to conduct research and development to improve safety in the construction industry.

---

## 2. Permission to Access and Use the Services

We grant you a limited, non-exclusive, non-transferable, revocable right to access and use the Services, subject to these Terms and all applicable laws.

You acquire **no ownership rights** in the Services. You may not install, download, or save our software except where expressly permitted. Constant internet access is required to use the Services.

---

## 3. What You Can Do

Subject to these Terms, you may:
- Access and use the Services for lawful purposes.
- Use the Services in compliance with our documentation, guidelines, and policies.

---

## 4. What You Cannot Do

You may not:
1. Use the Services in a way that infringes, misappropriates, or violates any third-party rights.
2. Modify, copy, lease, sell, sublicense, or distribute any part of the Services.
3. Reverse engineer, decompile, or attempt to derive the source code or underlying components of the Services, including models, algorithms, or systems.
4. Automatically or programmatically extract data or Output (as defined below).
5. Represent that Output was human-generated when it was not.
6. Interfere with or circumvent any access controls, safety measures, or usage limits.
7. Use Output to develop, train, or improve any model that competes with SafetyAI.
8. Remove or alter any proprietary notices, branding, or logos from the Services.
9. Embed the Services into another system in a way that conceals SafetyAI’s branding or identity.

---

## 5. Token Usage & Metering

- **Company Accounts (Production).** Token allocations are set **at the company level** under the applicable subscription/plan. Only **Client-designated administrative “power users”** may access the usage dashboard and view company-level consumption.

- **Demo Accounts (Non-production).** Individuals using the demo may view their **personal demo token balance** by clicking their avatar in the chatbot app. Demo usage is separate from any company allocation, is **not transferable**, and may be throttled or limited at any time.

- **Definition of “Token.”** A “token” is a unit of input or output text (including intermediate queries) processed by the large language model(s) underlying the Services, **as measured by the applicable LLM provider’s tokenizer**. On average, one token corresponds to **approximately four characters** of text; the exact count may vary with content.

- **Overages & Enforcement (Production).** If company usage exceeds the applicable allocation, **overage charges** apply at the then-current rates per the subscription or service plan and are invoiced as specified therein. We may **throttle, limit, or suspend** access for material over-consumption or unpaid overages after notice.

- **Visibility & Privacy.** Company-level dashboards may display aggregated and per-user usage details. Access to such dashboards must be limited by Client to a **small group of authorized power users** due to the sensitivity of this information.

---

## 6. Content

**Your Content** — You may provide input to the Services ("**Input**") and receive output generated by the Services based on the Input ("**Output**"). Input and Output are collectively "**Content**". You are responsible for ensuring your Content complies with all applicable laws and these Terms. You represent and warrant that you have all rights, licenses, and permissions needed for your Content.

**Similarity of Content** — Due to the nature of AI, Output may not be unique and other users may receive similar output.

**Our Use of Content** — We may use Content to operate, maintain, improve, and develop our Services, comply with law, enforce our terms, and ensure safety.

**Opt-Out** — You may opt out of our use of your Content for improvement purposes by emailing `antoine.tixier@safetyai.io`. Opting out may limit the Services’ ability to address your specific needs.

**Accuracy** — Large Language Models (LLMs) can produce incorrect or misleading Output. You agree:
- Output may not be accurate and should not be your sole source of truth or professional advice.
- You will verify Output for accuracy and appropriateness before use.
- You will not use Output relating to a person in decisions with legal or material impacts (e.g., employment, housing, insurance, medical, or legal matters).

**No Endorsement or Representation** — Our Services may provide incomplete, incorrect, or offensive Output that does not represent SafetyAI's views. If Output references any third party products or services, it does not imply endorsement or affiliation.

---

## 7. Confidentiality

**Use and Nondisclosure** — "Confidential Information" means non-public business, technical, or financial information disclosed by one party ("Discloser") to the other ("Recipient"), which is identified as confidential or should reasonably be understood as confidential. Confidential Information includes Customer Content.

Recipient agrees to:
(a) use Confidential Information only to fulfill its obligations or exercise its rights under these Terms;  
(b) protect Confidential Information with reasonable measures; and  
(c) not disclose Confidential Information to third parties except as permitted herein.

**Exceptions** — Obligations do not apply to information that:
(a) is public through no fault of the Recipient;  
(b) was known prior to disclosure;  
(c) is received without restriction from a third party; or  
(d) is independently developed without using the Discloser’s Confidential Information.

Recipient may share Confidential Information with employees, contractors, and agents who need to know and are bound by obligations at least as restrictive as these Terms. Disclosure may occur if required by law, with reasonable advance notice to the Discloser.

---

## 8. Privacy & Technical Specifications

We respect your privacy. The following disclosures apply to all users of our Services:

- **Personal Information You Provide** — For example, email addresses for account creation, authentication, and communication.
- **User Content** — For the SaaS version, stored in Dropbox; generated files (PPT, PDF) hosted in Google Drive; certain microservices hosted on Amazon EC2; deployed on Posit’s shinyapps.io (temporary session storage only). For the in-house Azure version, all storage and processing occur within Azure services.
- **LLM APIs** — ChatSafetyAI may use multiple Large Language Models (LLMs) via commercial APIs (e.g., GPT, Claude, Gemini). Data sent to these APIs is not used to train their models and is retained by the provider only temporarily (up to 30 days, **subject to change by the provider**) for moderation, after which it is deleted.
- **Location Data** — May be requested solely to improve prediction accuracy (e.g., weather-related safety predictions). Location is not stored beyond the session unless required for your specific use case.
- **IP, Device, and Browser Data** — We do **not** collect IP addresses, device/browser fingerprints, logs, or other analytics.
- **Cookies** — We do not set cookies. However, hosting platforms (e.g., shinyapps.io, Azure) may set cookies for the sole purpose of login persistence, MFA, or SSO.
- **Content Disclosure** — We do **not** sell or share your content with third parties. Conversations may be reviewed internally to improve the Services unless you opt out by emailing `antoine.tixier@safetyai.io`.

---

## 9. Our Intellectual Property Rights

We own all right, title, and interest in and to the Services, including all software, models, APIs, algorithms, content, features, and any modifications, improvements, or derivative works thereof. All rights not expressly granted to you in these Terms are reserved by us.

You may not copy, modify, adapt, translate, create derivative works from, reverse engineer, decompile, disassemble, or otherwise attempt to discover the source code, object code, or underlying structure, ideas, know-how, or algorithms relevant to the Services. You must not remove, alter, or obscure any proprietary notices or labels on the Services or related documentation.

You may only use our name, logos, and trademarks with our prior written consent.

---

## 10. Feedback

You may provide feedback, suggestions, or ideas about the Services, including conversation ratings or other usage-related feedback (“Feedback”). You agree that we may use Feedback freely, without restriction or compensation to you, for any purpose, including to improve or develop the Services.

---

*Sections 11 and 12 apply only to subscribed company accounts (non-demo users), not to demo accounts.*

## 11. SafetyAI Council & Logo Use

We may by default display your company’s name and logo on [https://safetyai.io](https://safetyai.io) as part of the SafetyAI Council member list. You may opt out at any time by emailing `antoine.tixier@safetyai.io`.

---

## 12. Training & Support

We may provide reasonable remote onboarding and technical support, including guidance on invitations, account setup, and using the Services. Where possible, training will be provided via **group sessions** to optimize resources.

---

## 13. Discontinuation of Services

We may discontinue the Services in whole or in part, with advance notice. If discontinued, we will refund any prepaid, unused fees.

---

## 14. Disclaimer of Warranties

OUR SERVICES ARE PROVIDED **"AS IS"**. TO THE MAXIMUM EXTENT PERMITTED BY LAW, WE DISCLAIM ALL WARRANTIES, EXPRESS OR IMPLIED, INCLUDING MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, NON-INFRINGEMENT, ACCURACY, OR UNINTERRUPTED OPERATION.

We do not guarantee:
- Continuous availability of the Services;
- Accuracy or reliability of any Output; or
- That Content will be secure or not lost or altered.

---

## 15. Limitation of Liability

TO THE MAXIMUM EXTENT PERMITTED BY LAW:
- WE WILL NOT BE LIABLE FOR INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL, OR EXEMPLARY DAMAGES, EVEN IF ADVISED OF THE POSSIBILITY.  
- OUR TOTAL LIABILITY WILL NOT EXCEED THE AMOUNT YOU PAID FOR THE SERVICES IN THE 12 MONTHS PRECEDING THE CLAIM.  
- DATA LOSS, ALTERATION, UNAUTHORIZED ACCESS, OR SYSTEM DOWNTIME—INCLUDING ISSUES INVOLVING THIRD-PARTY HOSTS (Dropbox, Google Drive, Amazon EC2, shinyapps.io, Azure, LLM API providers)—ARE AT YOUR OWN RISK.
- ACKNOWLEDGMENT OF RISK ALLOCATION — USER ACKNOWLEDGES THAT THE FEES APPLICABLE FOR THE SERVICES REFLECT THE ALLOCATION OF RISK SET FORTH IN THESE TERMS, AND THAT SAFETYAI WOULD NOT HAVE ENTERED INTO THIS AGREEMENT WITHOUT THE DISCLAIMERS OF WARRANTY AND LIMITATIONS OF LIABILITY AND DAMAGES CONTAINED HEREIN.

---

## 16. Indemnity

YOU WILL INDEMNIFY AND HOLD HARMLESS SAFETYAI, ITS AFFILIATES, AND THEIR PERSONNEL FROM ANY THIRD-PARTY CLAIMS, DAMAGES, LIABILITIES, AND EXPENSES (INCLUDING REASONABLE ATTORNEYS’ FEES) ARISING FROM YOUR:

(A) BREACH OF THESE TERMS;  
(B) MISUSE OF THE SERVICES; OR  
(C) VIOLATION OF LAW.

---

## 17. Dispute Resolution

**Mandatory Arbitration** — You and SafetyAI agree to resolve disputes through final and binding arbitration, regardless of when the dispute arose.

**Informal Resolution** — Before filing a claim, both parties will attempt informal resolution for 60 days.

**Governing Law** — Colorado law governs these Terms, excluding conflicts of laws principles. Arbitration or court proceedings will occur in Boulder or Denver, Colorado.

---

## 18. General Terms

**Assignment** — You may not assign or transfer your rights or obligations under these Terms without our prior written consent. Any attempt to do so will be void. We may assign or transfer our rights and obligations under these Terms to any affiliate, subsidiary, or successor in interest without restriction.

**Changes to These Terms or Our Services** — We may update these Terms or the Services from time to time, including for:  
- Changes to applicable laws or regulatory requirements;  
- Security or safety reasons;  
- Circumstances beyond our reasonable control;  
- Changes made in the ordinary course of developing or improving the Services;  
- Adaptation to new technologies.  

Unless otherwise required by law, changes will take effect upon posting to this GitHub page. If you do not agree to the changes, you must stop using the Services.

**Delay in Enforcing These Terms** — Our failure to enforce any provision is not a waiver of our right to do so later.

**Severability** — Except as provided in the dispute resolution section, if any portion of these Terms is determined to be invalid or unenforceable, that portion will be enforced to the maximum extent permissible, and the remaining portions will remain in full force and effect.

**Trade Controls** — You must comply with all applicable export control, sanctions, and trade laws. The Services may not be used in, for the benefit of, or exported or re-exported to (a) any U.S.-embargoed country or territory, or (b) any individual or entity subject to applicable trade restrictions. You may not use the Services for any end use prohibited by applicable trade laws.

---
