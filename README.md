# ChatSafetyAI

Help us improve ChatSafetyAI by taking this [quick survey](https://forms.gle/E6sK87u7DdoL4Kg86) !

## About
- ChatSafetyAI is built with care by [SafetyAI](https://www.safetyfunction.com/safetyai-details) 🛠️❤️.
- ChatSafetyAI relies on the [OpenAI API](https://platform.openai.com/docs/api-reference/introduction).
- OpenAI [does not](https://platform.openai.com/docs/models/how-we-use-your-data) use the conversations for training their models, and does not store the conversations permanently.
- We at SafetyAI record conversations for continuous improvement purposes. Conversations are inspected at random. Only one person has access.
- Screencast demos (not necessarily up-to-date) can be found on our [website](https://www.safetyfunction.com/safetyai-details).

Table of Contents
- [Usage Tiers](https://github.com/safetyAI/ChatSafetyAI/blob/main/README.md#usage-tiers)
- [Version History](https://github.com/safetyAI/ChatSafetyAI/blob/main/README.md#version-history)
- [Contact](https://github.com/safetyAI/ChatSafetyAI/blob/main/README.md#contact)
- [Terms of Use](https://github.com/safetyAI/ChatSafetyAI/blob/main/README.md#terms-of-use)

## Usage Tiers
There is a free [demo](https://safetyapp.shinyapps.io/chatsafetyai_demo_access_request/) with a limited budget, just for quick testing purposes.<br>
Contact us for membership pricing (see contact info at the bottom of this page, or on our [website](https://www.safetyfunction.com/safetyai-details)).<br>
Note: we do not currently offer plans for individuals, SafetyAI membership is for companies only.

<!-- html table generated in R 4.3.2 by xtable 1.8-4 package -->
<!-- Tue Sep 10 11:07:27 2024 -->
<table border=1>
<tr> <th>  </th> <th> Demo </th> <th> SafetyAI Council </th>  </tr>
  <tr> <td align="right"> Engine </td> <td> GPT-4o </td> <td> GPT-4o </td> </tr>
  <tr> <td align="right"> Conversation History </td> <td> 20k words, 5 turns </td> <td> 100k words, 10 turns </td> </tr>
  <tr> <td align="right"> Max Chats per Session </td> <td> 4 </td> <td> Unlimited </td> </tr>
  <tr> <td align="right"> PDF/CSV/Photo Upload </td> <td> Max 4 pages, 3 MB per file </td> <td> Max 120 pages, 10 MB per file </td> </tr>
  <tr> <td align="right"> Budget per Session </td> <td> 150k words </td> <td> Unlimited (within company budget) </td> </tr>
  <tr> <td align="right"> Budget per User </td> <td> 0.8M words </td> <td> Unlimited (within company budget) </td> </tr>
  <tr> <td align="right"> Custom Databases </td> <td> No </td> <td> Yes </td> </tr>
  <tr> <td align="right"> Custom Predictive Models </td> <td> No </td> <td> Yes </td> </tr>
  <tr> <td align="right"> Custom Risk Analysis </td> <td> No </td> <td> Yes </td> </tr>
  <tr> <td align="right"> Chat History </td> <td> Yes </td> <td> Yes </td> </tr>
   </table>

## Version History

### Next steps / wish list
- explore Gemini for reasoning on very long documents (2M-token context)
- incognito mode
- chat sharing as individual icons
- image passed to every call
- search engine for chats
- input multiple files at once

### 02-02-25
- support for deployment relying 100% on Azure services
- better handling of unknown language in mode instructions ("cannot tell") to avoid "sorry, I can't help" responses
- pagination in Dropbox file listing function to support directories containing many files
- generated PDFs and presentations now have a unique ID and contain the chat index, so can be deleted w/ the conv
- sensitive info now passed as environment variables

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
Bug reports, questions: `antoine DOT tixier AT safetyfunction .com`

## Terms of Use
These Terms of Use apply to your use of ChatSafetyAI, along with any associated models, APIs, software applications and websites (all together, "Services"). These Terms form an agreement between you and the Colorado Construction Safety Laboratory, LLC, a Colorado company, and they include our Service Terms and important provisions for resolving disputes through arbitration. By using our Services, you agree to these Terms.

### Who We Are
ChatSafetyAI is developed by SafetyAI, an entity within Safety Function, d/b/a the Colorado Construction Safety Laboratory, LLC.
Our mission is to do R&D that improves safety in the construction industry.

### Using Our Services
**What You Can Do.** Subject to your compliance with these Terms, you may access and use our Services. In using our Services, you must comply with all applicable laws as well as any other documentation, guidelines, or policies we make available to you.

**What You Cannot Do.** You may not use our Services for any illegal, harmful, or abusive activity. For example, you may not:
- Use our Services in a way that infringes, misappropriates or violates anyone’s rights.
- Modify, copy, lease, sell or distribute any of our Services.
- Attempt to or assist anyone to reverse engineer, decompile or discover the source code or underlying components of our Services, including our models, algorithms, or systems.
- Automatically or programmatically extract data or Output (defined below).
- Represent that Output was human-generated when it was not.
- Interfere with or disrupt our Services, including circumvent any rate limits or restrictions or bypass any protective measures or safety mitigations we put on our Services.
- Use Output to develop models that compete with the Colorado Construction Safety Laboratory, LLC.

**Feedback.** We appreciate your feedback, and you agree that we may use it without restriction or compensation to you.

### Content
**Your Content.** You may provide input to the Services ("Input"), and receive output from the Services based on the Input ("Output"). Input and Output are collectively "Content". You are responsible for Content, including ensuring that it does not violate any applicable law or these Terms. You represent and warrant that you have all rights, licenses, and permissions needed to provide Input to our Services.

**Similarity of Content.** Due to the nature of our Services and artificial intelligence generally, output may not be unique and other users may receive similar output from our Services.

**Our Use of Content.** We may use Content to provide, maintain, develop, and improve our Services, comply with applicable law, enforce our terms and policies, and keep our Services safe.

**Opt Out.** If you do not want us to use your Content to provide, maintain, develop, and improve our Services, you can opt out by emailing `antoine.tixier AT safetyfunction DOT com`. Please note that this may limit the ability of our Services to better address your specific use cases.

**Accuracy.** ChatSafetyAI relies on the GPT family of Large Language Models (LLMs) via the OpenAI API. GPTs, like other LLMs, can make mistakes in some situations. Therefore, use of our Services may, in some situations, result in Output that is not factually correct.

When you use our Services you understand and agree:

- Output may not always be accurate. You should not rely on Output from our Services as a sole source of truth or factual information, or as a substitute for professional advice.
- You must evaluate Output for accuracy and appropriateness for your use case, including using human review as appropriate, before using or sharing Output from the Services.
- You must not use any Output relating to a person for any purpose that could have a legal or material impact on that person, such as making credit, educational, employment, housing, insurance, legal, medical, or other important decisions about them. 
- Our Services may provide incomplete, incorrect, or offensive Output that does not represent the Colorado Construction Safety Laboratory's views. If Output references any third party products or services, it doesn't mean the third party endorses or is affiliated with the Colorado Construction Safety Laboratory.

### Confidentiality
**Use and Nondisclosure.** "Confidential Information" means any business, technical or financial information, materials, or other subject matter disclosed by one party ("Discloser") to the other party ("Recipient") that is identified as confidential at the time of disclosure or should be reasonably understood by Recipient to be confidential under the circumstances. For the avoidance of doubt, Confidential Information includes Customer Content. Recipient agrees it will: (a) only use Discloser's Confidential Information to exercise its rights and fulfill its obligations under this Agreement, (b) take reasonable measures to protect the Confidential Information, and (c) not disclose the Confidential Information to any third party except as expressly permitted in this Agreement.

**Exceptions.** These obligations do not apply to any information that (a) is or becomes generally available to the public through no fault of Recipient, (b) was in Recipient’s possession or known by it prior to receipt from Discloser, (c) was rightfully disclosed to Recipient without restriction by a third party, or (d) was independently developed without use of Discloser’s Confidential Information. Recipient may disclose Confidential Information only to its employees, contractors, and agents who have a need to know and who are bound by confidentiality obligations at least as restrictive as those of this Agreement. Recipient will be responsible for any breach of this section by its employees, contractors, and agents. Recipient may disclose Confidential Information to the extent required by law, provided that Recipient uses reasonable efforts to notify Discloser in advance.

### Privacy
We at the Colorado Construction Safety Laboratory, LLC ('we', 'our' or 'us') respect your privacy and are strongly committed to keeping secure any information we obtain from you or about you. This Privacy Policy describes our practices with respect to Personal Information we receive from or about you when you use our Services.

**Personal Information You Provide:** We receive Personal Information if you create an account to use our Services or communicate with us as follows:
- email address.

**Personal Information We Receive Automatically From Your Use of the Services:** When you use the Services, we receive the following information:
- *User Content:* the date and time of each conversation, as well as the conversation content. This is necessary to allow saving and reloading the chat history from session to session. We use Dropbox as our main data storage solution, Google Drive for hosting the automatically generated PPT files (presentation mode...) and PDF files (prejob brief summarization, chat sharing...), and Amazon EC2 for some data storage and to host the microservices of ours that ChatSafetyAI calls (regulations, predictions, etc.). ChatSafetyAI is deployed on Posit's shinyapps.io but data is only saved there temporarily, for the duration of the session. We record and analyze conversations for sole continuous improvement purposes. If you do not want us to use your conversations for improvement of our Services, or if you do not want your conversations to be saved at all (no chat history from session to session), you may opt out by emailing `antoine.tixier AT safetyfunction DOT com`. Please note that this may limit the ability of our Services to better address your specific use cases.
- *OpenAI API:* To function, ChatSafetyAI relies on the OpenAI API's Chat and Embeddings endpoints. OpenAI [does not](https://platform.openai.com/docs/models/how-we-use-your-data) use the data passed to their API for training their models, and only temporarily saves the data (for 30 days) for moderation purposes, after which it is permanently deleted.
- *Content Disclosure:* we **do not** share or sell your content with/to anyone. We record and analyze your conversations for the sole purpose of improving our models, if you have not opted out.
- *IP address, device and browser information, logs, and other analytics:* we **do not** record such information. We may ask for your location (latitude and longitude) for the sole purpose of knowing your local weather, in order for our incident characteristics predictive models to generate more accurate predictions.

- *Cookies:* we **do not** use cookies. Shinyapps.io, on which ChatSafetyAI is deployed, may collect a cookie for the sole purpose of allowing you to log in without having to re-enter your username and password every time.

### Our IP Rights
We own all rights, title, and interest in and to the Services. You may only use our name and logo by asking us first.

### Permission to Access the Services
We grant you a limited, non-exclusive permission subject to conditions, to access our Services.
You shall have no right to copy, modify, save, reproduce, publish or convey any part of our Services, and shall acquire no ownership in our Services.
ChatSafetyAI is deployed on the Posit's shinyapps.io server and is accessible as a web application.
We make no claim of responsibility for interruptions in the server.
You will receive an invitation by email to access ChatSafetyAI with no ability to install or save software.
Constant access to the Internet will be required to use ChatSafetyAI.

We shall own all right, title and interest in our Services and all other intellectual property rights inherent therein, including all modifications, improvements, and derivative works relating to our Services. Other than where we provide prior written consent, the foregoing permission does not include any right to reproduce our Services or to make and/or distribute or sublicense variations of or derivative works developed from our Services, or any right to utilize our Services or information relating to them other than as expressly permitted herein.

### Discontinuation of Services
We may decide to discontinue our Services, but if we do, we will give you advance notice and a refund for any prepaid, unused Services.

### Disclaimer of Warranties
OUR SERVICES ARE PROVIDED "AS IS." EXCEPT TO THE EXTENT PROHIBITED BY LAW, WE MAKE NO WARRANTIES (EXPRESS, IMPLIED, STATUTORY OR OTHERWISE) WITH RESPECT TO THE SERVICES, AND DISCLAIM ALL WARRANTIES INCLUDING, BUT NOT LIMITED TO, WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, SATISFACTORY QUALITY, NON-INFRINGEMENT, AND QUIET ENJOYMENT, AND ANY WARRANTIES ARISING OUT OF ANY COURSE OF DEALING OR TRADE USAGE. WE DO NOT WARRANT THAT THE SERVICES WILL BE UNINTERRUPTED, ACCURATE OR ERROR FREE, OR THAT ANY CONTENT WILL BE SECURE OR NOT LOST OR ALTERED.

IN NO EVENT SHALL WE BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL, EXEMPLARY OR PUNITIVE DAMAGES, INCLUDING WITHOUT LIMITATION DAMAGES FOR LOSS OF PROFITS, LOSS OF GOOD WILL, LOSS OF DATA OR USE, OR ANY BUSINESS INTERRUPTION OR DISRUPTION, INCURRED BY EITHER USER OR ANY THIRD PARTY, WHETHER IN AN ACTION SOUNDING IN CONTRACT, TORT (INCLUDING NEGLIGENCE), INDEMNITY, WARRANTY, FIDUCIARY DUTY, STATUTORY CLAIM UNDER ANY FEDERAL, STATE, LOCAL LAW OF THE UNITED STATES OF AMERICA OR ANY OTHER JURISDICTION, OR ANY OTHER TYPE OF LEGAL CLAIM, EVEN IF THE OTHER PARTY HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGES.

FURTHER, NEITHER WE WILL BE RESPONSIBLE FOR ANY COMPENSATION, REIMBURSEMENT, LOSSES, COSTS OR DAMAGES ARISING IN CONNECTION WITH: (A) USER'S INABILITY TO USE THE SERVICES, INCLUDING AS A RESULT OF ANY (I) TERMINATION OR SUSPENSION OF THIS AGREEMENT OR USER'S USE OF OR ACCESS TO THE SERVICES, (II)  OUR DISCONTINUATION OF ANY OR ALL ACCESS TO THE SERVICES, OR (III) ANY UNANTICIPATED OR UNSCHEDULED DOWNTIME OF ALL OR A PORTION OF THE ACCESS TO THE SERVICES FOR ANY REASON WHATSOEVER, INCLUDING AS A RESULT OF POWER OUTAGES, SYSTEM FAILURES OR OTHER INTERRUPTIONS; (B) THE COST OF PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; (C) ANY INVESTMENTS, EXPENDITURES, OR COMMITMENTS BY USER TO ANY THIRD PARTIES IN CONNECTION WITH THIS AGREEMENT OR USER'S USE OF OR ACCESS TO THE SERVICES; OR (D) ANY UNAUTHORIZED ACCESS TO, ALTERATION OF, OR THE DELETION, DESTRUCTION, DAMAGE, LOSS, DENIAL OF ACCESS, OR FAILURE TO MAINTAIN OR STORE ANY OF USER'S CONTENT OR OTHER DATA.

YOU ACCEPT AND AGREE THAT ANY USE OF OUTPUTS FROM OUR SERVICE IS AT YOUR SOLE RISK AND YOU WILL NOT RELY ON OUTPUT AS A SOLE SOURCE OF TRUTH OR FACTUAL INFORMATION, OR AS A SUBSTITUTE FOR PROFESSIONAL ADVICE.

### Limitation of Liability
WE MAKE NO CLAIM OF RESPONSIBILITY FOR ANY PARTIAL OR TOTAL LOSS, ALTERATION, DELETION, DESTRUCTION, DAMAGE, FAILURE TO MAINTAIN OR STORE, RELEASE, LEAKAGE, OR DENIAL OF ACCESS, OF ANY OF THE DATA UPLOADED TO CHATSAFETYAI, DROPBOX, AMAZON EC2, GOOGLE DRIVE, POSIT'S SHINYAPPS.IO, THE OPENAI API, OR ANY OTHER THIRD PARTY SERVER OR CLOUD-BASED SOLUTION USED BY US AS PART OF PROVIDING OUR SERVICES, FOR ANY REASON WHATSOEVER.

WE WILL NOT BE LIABLE FOR ANY INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL, OR EXEMPLARY DAMAGES, INCLUDING DAMAGES FOR LOSS OF PROFITS, GOODWILL, USE, OR DATA OR OTHER LOSSES, EVEN IF WE HAVE BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGES. OUR AGGREGATE LIABILITY UNDER THESE TERMS WILL NOT EXCEED ​​THE GREATER OF THE AMOUNT YOU PAID FOR THE SERVICE THAT GAVE RISE TO THE CLAIM DURING THE 12 MONTHS BEFORE THE LIABILITY AROSE. THE LIMITATIONS IN THIS SECTION APPLY ONLY TO THE MAXIMUM EXTENT PERMITTED BY APPLICABLE LAW.

Some countries and states do not allow the disclaimer of certain warranties or the limitation of certain damages, so some or all of the terms above may not apply to you, and you may have additional rights. In that case, these Terms only limit our responsibilities to the maximum extent permissible in your country of residence.

OUR AGGREGATE AND CUMULATIVE TOTAL LIABILITY FOR DAMAGES, INCLUDING FOR DIRECT DAMAGES, UNDER THIS AGREEMENT, SHALL IN NO EVENT EXCEED THE AMOUNT OF FEES PAID BY USER UNDER THIS AGREEMENT THAT GAVE RISE TO THE CLAIM DURING THE 12 MONTHS PRECEDING THE CLAIM, AND IF SUCH DAMAGES RELATE TO PARTICULAR SERVICES, SUCH LIABILITY SHALL BE LIMITED TO FEES PAID FOR THE SERVICES GIVING RISE OR RELATED TO THE ALLEGED LIABILITY AND DAMAGES UNDER THIS AGREEMENT DURING THE 12 MONTHS PRECEDING THE CLAIM.
USER ACKNOWLEDGES THAT THE FEES APPLICABLE FOR THE SERVICES REFLECT THE ALLOCATION OF RISK SET FORTH IN THIS AGREEMENT AND THAT WE WOULD NOT HAVE ENTERED INTO THIS AGREEMENT WITHOUT THE DISCLAIMERS OF WARRANTY AND LIMITATIONS OF BOTH LIABILITY AND DAMAGES SET FORTH IN THIS AGREEMENT.

### Indemnity
To the extent permitted by law, you will indemnify and hold harmless us and our personnel, from and against any costs, losses, liabilities, and expenses (including attorneys' fees) from third party claims arising out of or relating to your use of the Services and Content or any violation of these Terms.

### Dispute Resolution
YOU AND THE COLORADO CONSTRUCTION SAFETY LABORATORY AGREE TO THE FOLLOWING MANDATORY ARBITRATION AND CLASS ACTION WAIVER PROVISIONS:

**MANDATORY ARBITRATION.** You and The Colorado Construction Safety Laboratory agree to resolve any claims arising out of or relating to these Terms or our Services, regardless of when the claim arose, even if it was before these Terms existed (a "Dispute"), through final and binding arbitration.

**Informal Dispute Resolution.** We would like to understand and try to address your concerns prior to formal legal action. Before either of us files a claim against the other, we both agree to try to resolve the Dispute informally. If we are unable to resolve a Dispute within 60 days, either of us has the right to initiate arbitration.

**Governing Law.** Colorado law will govern these Terms except for its conflicts of laws principles. Except as provided in the dispute resolution section above, all claims arising out of or relating to these Terms will be brought exclusively in the federal or state courts of Boulder or Denver, Colorado.

### General Terms
**Assignment.** You may not assign or transfer any rights or obligations under these Terms and any attempt to do so will be void. We may assign our rights or obligations under these Terms to any affiliate, subsidiary, or successor in interest of any business associated with our Services.

**Changes to These Terms or Our Services.** We are continuously working to develop and improve our Services. We may update these Terms or our Services accordingly from time to time. For example, we may make changes to these Terms or the Services due to:

- Changes to the law or regulatory requirements.
- Security or safety reasons.
- Circumstances beyond our reasonable control.
- Changes we make in the usual course of developing our Services.
- To adapt to new technologies.
- All other changes will be effective as soon as we post them to our website. If you do not agree to the changes, you must stop using our Services.

**Delay in Enforcing These Terms.** Our failure to enforce a provision is not a waiver of our right to do so later. Except as provided in the dispute resolution section above, if any portion of these Terms is determined to be invalid or unenforceable, that portion will be enforced to the maximum extent permissible and it will not affect the enforceability of any other terms.

**Trade Controls.** You must comply with all applicable trade laws, including sanctions and export control laws. Our Services may not be used in or for the benefit of, or exported or re-exported to (a) any U.S. embargoed country or territory or (b) any individual or entity with whom dealings are prohibited or restricted under applicable trade laws. Our Services may not be used for any end use prohibited by applicable trade laws, and your Input may not include material or information that requires a government license for release or export. 

**Entire Agreement.** These Terms contain the entire agreement between you and the Colorado Construction Safety Laboratory regarding the Services and, other than any Service-specific terms, supersedes any prior or contemporaneous agreements between you and the Colorado Construction Safety Laboratory. 

**Governing Law.** Colorado law will govern these Terms except for its conflicts of laws principles. Except as provided in the dispute resolution section above, all claims arising out of or relating to these Terms will be brought exclusively in the federal or state courts of Boulder or Denver, Colorado.
