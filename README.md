# ChatSafetyAI Version History

## 10-22-23
- plenty of red teaming and example testing
- adversarial and off-topic classifiers now take the conversation as context to make better-informed decisions and reduce annoying false alarms, also better instructions
- **TODO** mode is now automatically detected for each query: several modes can now be used within the same chat, alleviating filtering false alarms and improving acceptance of follow-up questions
- prettify now handles markdown tables and is called during generation
- updated message at the bottom of the page
- SHIFT + ENTER now does not trigger
- made textArea and scrollbar styles consistent with rest of the page
- slightly more width for discussion thread
- examples now randomly selected from a large pool at the beginning of each chat
- new safety climate survey analysis mode
- now calling the latest generic models in prediction mode (SUPER-COMPANY 2023 update)
- now showing the latest generic risk assessment in prediction mode (SUPER-COMPANY 2023 update)
- now using GPT-4, and 11/06 version of GPT-3.5-turbo
- added suggestions of follow-up questions in default mode
- fixed scroll down
- added link to changelog (this page)
- reduced user budget and max nb of threads in single session to account for the additional cost of GPT-4
- global parameter to enable privacy selection
- `prettify_text` now handles double stars for bold
- logo now links to website
- fixed several instruction files
- knowledge about energy wheel and HECA 

## TODO determine based on email announcements
- OSHA and prediction modes
- tab names are now automatically renamed as extreme summaries of their conversations
- now using GPT-3.5-turbo 16k
- TODO finish

## TODO determine based on first email
Initial release
