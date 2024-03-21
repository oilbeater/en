---
title: How to Easily Integrate AI-Generated Content into Logseq Notes
date: March 21, 2024 10:39:37
tags: [logseq, ai, learning]
---

My daily life has already incorporated a lot of assistance from AI, such as translating text, explaining a term, answering a question, etc. However, after frequent use, I encountered another problem: these AI outputs are beneficial supplements to my knowledge, but the use is usually one-off. I don't settle these contents down. It is inconvenient when I need to collect them into notes because I have to organize them again. Imagine if this temporary query information could be automatically saved to my note-taking system, there would no longer be a worry about losing important knowledge snippets. Thus, I came up with the idea of creating a workflow to automatically store these AI-generated contents in my note system, and then review them periodically. Here, I will discuss the limitations of some tools I researched and how I used a simple framework to automatically import AI-generated content into logseq and make it into flashcards.

# Problem Decomposition and Preliminary Research

Here, I take a daily translation scenario as an example: When encountering a sentence that I don't understand, I wish to highlight and translate it, then have AI analyze the difficult words in the sentence, and finally save these contents as flashcards format in logseq. This way, with the synchronization on the mobile side, I can review this sentence during fragmented time. So, the problem essentially consists of three parts: 1. Extracting the highlighted text 2. Writing a prompt to generate the content I want 3. Saving it to logseq.

The first two steps have been integrated into many tools, including Raycast AI and OpenAI-translator, but none of them offer an extension interface to export the results of the first two steps to a third party. Raycast could theoretically accomplish this with its plugin system, but I'm not very familiar with JS and also don't want to rely too much on Raycast AI, as I plan to discontinue the subscription. So, I decided to write a python script to do these things and then directly call the script with Raycast's script command and shortcut keys, theoretically completing the entire workflow with one shortcut key.

# How to Obtain Selected Text

The first challenge is obtaining the selected text, which seems to be an operation at the system level and is not easy to implement. It was not until I saw the implementation of a dictionary software that I had an epiphany:

```bash 
# https://github.com/raycast/script-commands/blob/master/commands/apps/dictionary/look-up-in-dictionary.applescript
tell application "System Events"
    keystroke "c" using {command down}
    delay 0.1
    do shell script "open dict://" & the clipboard
end
```

This script essentially simulates pressing `cmd + v` on the keyboard, then directly reads from the clipboard, bypassing the issue of obtaining selected text. Considering that I often encounter recognition errors of selected text when using Raycast, I've developed the habit of using `cmd + v`, so there's no need to find other methods for obtaining selected text; directly reading the clipboard will suffice.

# Calling AI

This part is pretty straightforward. The key is setting an appropriate prompt, translating while generating a teaching guide for a sentence, and outputting as much as possible in the format of logseq:

```python
message_text = [
    {
        "role": "system",
        "content": "You are a university English teacher, below is a paragraph in English. " +
        "Please first translate it into Chinese. Then extract difficult words and phrases from the source paragraph, sort them in descending order of importance, choose only the top 3, and output them with an explanation of their usage to me in detail from a linguistic perspective." +
        "The overall output should look like this: \n" +
        "- {The Chinese Translation} \n" +
        "- Explanation: \n" +
        "  - {word or phrase}: {explanation}\n"
    },
    {
        "role":"user",
        "content": content
    }
]
```

# Output to Logseq

When it came to outputting to logseq, I initially planned to call Logseq's API, which provides a local HTTP service that can be called through HTTP in developer mode. However, when I looked at their API documentation, I was almost overwhelmed. All operations required an ID, and there was no direct way to index page IDs; it was necessary to getAll and then filter. appendBlock did not allow inserting blocks with hierarchy, requiring several API calls to complete a simple operation.

When I was about to give up, I realized that Logseq is essentially a renderer for markdown files. Since the API was difficult to use, I decided to directly write files, thus making a series of complicated API calls into enjoyable file append operations. This not only circumvented the limitations of the Logseq API but also potentially allowed integration with other markdown-based note systems.

The final code is as follows:

```python
import pyperclip
from openai import AzureOpenAI

content = pyperclip.paste()

azure_endpoint = "YOUR AZURE ENDPOINT"
api_key = "YOUR API KEY"
model = "YOUR MODEL NAME"
logseq_path = "YOUR LOGSEQ PAGE FILE PATH"

client = AzureOpenAI(
    azure_endpoint=azure_endpoint,
    api_key=api_key,
    api_version="2024-02-15-preview"
)

message_text = [
    {
        "role": "system",
        "content": "You are a university English teacher, below is a paragraph in English. " +
        "Please first translate it into Chinese. Then extract difficult words and phrases from the source paragraph, sort them in descending order of importance, choose only the top 3, and explain their usage to me in detail from a linguistic perspective." +
        "The overall output should look like this: \n" +
        "- {The Chinese Translation} \n" +
        "- Explanation: \n" +
        "  - {word or phrase}: {explanation}\n"
    },
    {
        "role":"user",
        "content": content
    }
]

completion = client.chat.completions.create(
    model=model,  # model = "deployment_name"
    messages=message_text,
    temperature=0.7,
    max_tokens=500,
    top_p=0.95,
    frequency_penalty=0,
    presence_penalty=0,
    stop=None
)

if completion.choices[0].message.content is None:
    print("No response")
    exit()

response = completion.choices[0].message.content
print(response)

with open(logseq_path, 'a') as f:
    f.write("\n- " + content.rstrip() + " #card #English" + '\n')
    for line in response.splitlines():
        if line.startswith("```"):
            continue
        if line.rstrip() == "":
            continue
        if not line.startswith("- "):
            line = "- " + line.rstrip()
        f.write("  " + line + '\n')
```
The remaining task is to set a shortcut for this script in Raycast, so the next time I encounter a sentence I don't understand, I can select, copy, and complete the translation, difficulty extraction, and flashcard generation workflow with a shortcut key.

The final flashcard effect is as follows:

![alt text](../../images/logseq-card.png)

# Conclusion

By integrating AI-generated content into Logseq notes automatically, we not only enhance the efficiency of information management but also optimize the learning and workflow process. With a simple script, anyone can easily integrate this powerful technology into their daily life, enabling the accumulation and review of knowledge.

> This post was originally written in Chinese and I translated it to English with the help of GPT4. If you find any errors, please feel free to let me know.