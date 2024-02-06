# ChatSafetyAI Version History

## 02-10-24
- Image, CSV, and PDF inputs.
- new "HECA" image mode.
- Upgraded to latest models: `gpt-4-0125-preview`, `gpt-4-vision-preview`, and `gpt-3.5-turbo-0125`.
- NSFW classifiers for text and images
- off-topic classifier for images, handling of API refusals

## 01-16-24
- Expanded knowledge about direct controls, HECA, and the energy method

## 01-15-24
- First version of "prejob safety brief and paperwork filling" mode
- Prejob mode examples
- Number of examples adapts to screen width
- Improved off-topic classifier to reduce false alarms
- Improved presentation instructions to increase figures' inclusion rate
- Replaced traffic safety example by fire prevention example (presentation mode)

## 12-28-23
- Fixed error vectors in if statements of NLP tool (new version of R issues error instead of warning)
- NLP API error handling in `get_attributes.R`

## 12-26-23
- improved presentation instructions
- global knowledge in presentation instructions
- better sanitization of presentation Markdown code (handle triple backquotes)
- added one example of the presentation category
- shorter bottom info message
  
## 12-25-23
- enforce the sampling of examples from different categories (covering different modes)

## 12-24-23
- fixed 'New chat' button placement for mobile devices

## 12-23-23
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

## 07-25-23
- OSHA and prediction modes
- tab names are now automatically renamed as extreme summaries of their conversations
- now using GPT-3.5-turbo 16k
- TODO finish

## 05-12-23
- initial release
