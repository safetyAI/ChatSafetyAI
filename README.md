# ChatSafetyAI Version History

## 10-22-23
- ability to disable autoscroll
- better formatting of tables
- plenty of red teaming and legit example testing
- proper handling of line breaks by sanitizing and HTML display functions
- classifiers take the conversation as context to make better-informed decisions
- classifiers have better instructions
- classifiers do not send green flag when query fails
- mode automatically detected for each query with function calling (taking conversation into account, capable of retrieving arguments in earlier messages). Modes can now be switched within the same chat, removing the need for mode-specific classifiers, and improving acceptance of follow-up questions.
- prettify now handles markdown tables and is called during generation
- updated message at the bottom of the page
- SHIFT + ENTER in text area now does not trigger
- textArea and scrollbar styles consistent with rest of the page
- slightly more width for discussion thread
- examples now randomly selected from a large pool at the beginning of each chat
- new safety climate survey analysis mode
- calling the latest generic models in prediction mode (SUPER-COMPANY 2023 update)
- showing the latest generic risk assessment in prediction mode (SUPER-COMPANY 2023 update)
- now relying on GPT-4-Turbo from 11/06 (bleeding-edge version)
- follow-up questions now suggested via function calling
- fixed scroll down issues
- added link to changelog (this page)
- reduced user budget and max nb of threads in single session to account for the additional cost of GPT-4
- global parameter to enable privacy selection
- `prettify_text` now handles basic markdown tags
- logo now links to website
- improved several instruction files
- injected knowledge about energy wheel and HECA 

## TODO determine based on email announcements
- OSHA and prediction modes
- tab names are now automatically renamed as extreme summaries of their conversations
- now using GPT-3.5-turbo 16k
- TODO finish

## TODO determine based on first email
Initial release
