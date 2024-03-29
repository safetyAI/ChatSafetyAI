# ChatSafetyAI

## About
- ChatSafetyAI is built with care by [SafetyAI](https://www.safetyfunction.com/safetyai-details) 🛠️❤️.
- ChatSafetyAI relies on the [OpenAI API](https://platform.openai.com/docs/api-reference/introduction).
- OpenAI [does not](https://platform.openai.com/docs/models/how-we-use-your-data) use the conversations for training their models, and does not store the conversations permanently.
- We at SafetyAI record conversations for continuous improvement purposes.
- Screencast demos (not necessarily up-to-date) can be found on our [website](https://www.safetyfunction.com/safetyai-details).

## Usage Tiers
The demo is free to use for limited testing purposes.<br>
Contact us for Enterprise pricing (see contact info at the bottom of this page, or on our [website](https://www.safetyfunction.com/safetyai-details)).<br>
Note: we do not currently offer plans for individuals.
<!-- html table generated in R 4.3.2 by xtable 1.8-4 package -->
<!-- Wed Mar 27 20:36:22 2024 -->
<table border=1>
<tr> <th>  </th> <th> Demo </th> <th> Enterprise </th>  </tr>
  <tr> <td align="right"> Engine </td> <td> GPT-3.5 </td> <td> GPT-4 </td> </tr>
  <tr> <td align="right"> Conversation History </td> <td> 10k words, 3 turns </td> <td> 100k words, 9 turns </td> </tr>
  <tr> <td align="right"> Max Chats per Session </td> <td> 4 </td> <td> 12 </td> </tr>
  <tr> <td align="right"> PDF/CSV/Photo Upload </td> <td> max 3 pages, 2 MB per file </td> <td> max 75 pages, 10 MB per file </td> </tr>
  <tr> <td align="right"> Budget per Session </td> <td> 290k words </td> <td> Unlimited </td> </tr>
  <tr> <td align="right"> Budget per User </td> <td> 0.5M words </td> <td> Custom, min 10M words </td> </tr>
  <tr> <td align="right"> Custom Databases </td> <td> No </td> <td> Yes </td> </tr>
  <tr> <td align="right"> Custom Predictive Models </td> <td> No </td> <td> Yes </td> </tr>
  <tr> <td align="right"> Custom Risk Analysis </td> <td> No </td> <td> Yes </td> </tr>
  <tr> <td align="right"> Chat History </td> <td> No </td> <td> Yes </td> </tr>
   </table>

## Version History

### 03-28-24
- private app, public demo
- routines now reply in the language of the original query (except those relying on the `[START]` and `[END]` special tokens for non-English languages, due to bug: https://community.openai.com/t/gpt-4-1106-preview-is-not-generating-utf-8/482839)
- support for non-English headlines
- refactoring of adversarial and off-topic classifiers to reduce false alarms, especially in non-English languages
- bug fix bad function call handling for follow-up question suggestions
- use of `gpt-4-0125-preview` for follow-up question suggestions, to generate proper accentuated chars in Spanish and French (related to bug below)
- refactoring of supported language classifier to return language instead of just yes/no
- more comprehensive cost measurement

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
