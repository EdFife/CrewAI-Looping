# CrewAI-Looping
Quick discussion regarding getting a CREWAI agent to loop through a list of topics.

## Looping through a list of items as topics for an agent and task

I have a workflow I was working on with multiple steps. One of my steps took a json file with an outline containing blog title, subtitle, introduction, conclusion and chapter titles. I tried multiple methods to get the chapter content written. I tried kickoff for each, but could not get it working (although in hindsight I was using a multi- agent crew then) so I decided to do it my self.

Here is a down and dirty approach, it has taught me a bit about the structure and functionality of crewai. This is certainly not production ready, but it is a basis to get a loop working. ChatGPT, Llama 3.1 on huggingface offered some insight, but i found thier approaches difficult to implement. I loved how quickly they helped with errors, both syntax and PEBKAC.

I did not get fancy and break out extra files or classes. Just functions all in one file to keep it lean and easy to debug. Code is commented to help you understand the logic and flow. Good Luck!!

My llm is groq, it is configured in a seperate file, but configuration is standard for this:

### llm.py
llm definition only in this file.

```
from langchain_groq import ChatGroq

groq_llm = ChatGroq(
    groq_api_key = "your key",
    model_name = 'llama-3.1-70b-versatile',
    temperature = 0.2,
)
```

### chapter_crew.py
The only other file involved. This does all the work.
```

import json
from textwrap import dedent
from crewai import Agent, Task
from langchain_community.tools import DuckDuckGoSearchRun
from llm import groq_llm

# Define LLM
default_llm = groq_llm

# Define tools
search_tool = DuckDuckGoSearchRun() # no API key required for searching

# Define the filename to save content
filename = 'your_filename_here.md'

# function to create and process through the list of chapter topics, the real workhorse of the whole file
def process_topics_and_save(inputs, output_file= filename):
    """
    This function processes a list of topics, executes the Crew Agent for each topic,
    and appends the output to a markdown file.
  
    Args:
    inputs (list): A list of topics to process.

    Output:
    The function appends the result to 'your_filename'.
    """
  
    for topic in inputs:
         # Initialize and execute the agent with the current topic
        if not topic:  # Skip invalid or empty topics
            print(f"Skipping invalid topic: {topic}")
            continue
        result = draft_writer.execute_task(chapter_writing_task, topic)  # draft_writer is my agent, chapter_writing_task is my task. Topic is the current list item to be written about. This line will actually launch the agent configurator in your terminal
  
        # Call the function to save the result with title formatting as H2
        save_file_as_md(result, topic, output_file) # result is your written content, other stuff should be evident.

        # print(f"All topics processed and saved to {output_file}") # feedback that your info got passed to the save function

# function to save chapters to file
def save_file_as_md(content, topic_title, filename):
    """
    This function appends the agent's output to a markdown file with proper formatting.
  
    Args:
    content (str): The content to be appended to the markdown file.
    topic_title (str): The title of the current topic, formatted as H2 in the markdown file.
    filename (str): The path to the markdown file where the content will be appended.

    Output:
    The function appends the content to the specified markdown file.
    """
    # Open the file in append mode and write the title and content
    with open(filename, 'a') as file:
        # Format the title as H2 and add two new lines before the content
        file.write(f"## {topic_title}\n\n{content}\n\n")  # formats title as H2 adds line breaks between your heading and content and breaks after content before next heading.
        print(f"Content saved to {filename} for topic: {topic_title}")

# Load JSON file into memory  simple read of my json file, specific to my situation. 
json_filename = 'output/blog_outline.json'
with open(json_filename, 'r') as json_file:
    blog_outline = json.load(json_file)


# Extract relevant data from the outline  This is specific to my data structure of course, modify it to fit your if appropriate.
title = blog_outline.get('title')
subtitle = blog_outline.get('subtitle')
introduction = blog_outline.get('introduction')
conclusion = blog_outline.get('conclusion')
chapters = blog_outline.get('chapters', [])  # Use an empty list if no chapters

# Writing initial content to the output file My previous step wrote title, subtitle, introduction and conclusion and then wrote chapter titles to use for the article.
with open(filename, 'a') as file:
    file.write(f"# {title}\n\n")
    file.write(f"## {subtitle}\n\n")
    file.write(f"{introduction}\n\n")
    # file.write(f"{conclusion}\n\n")
    print("Initial content saved")

# Define Agent, your agent of course will be differant
draft_writer = Agent(
    role='draft writer',
    backstory=dedent(f"""
        - Write detailed, well-thought-out paragraphs discussing multiple viewpoints aligned with {mvv}.
        - Current year is 2024. Do not consider research older than 2020.
        - Your paragraphs should have varying length and sentence structure. You must cite your references in APA format.
        - You consider all points of view when discussing your topics.
        """),
    goal=dedent(f"""
        - You are an expert in creating articles for diverse and unique topics in the realm of ...
        - You consider all aspects and viewpoints before delivering a thorough discussion.
        - Write paragraphs, not bulleted lists.
        - Multiple paragraphs covering the topic completely.
        """),
    max_rpm=99,  # using free version of groq so it is speed throttled
    max_iter=25,
    allow_delegation=False,
    verbose=True,
    tools = [search_tool],
    llm=default_llm
    )

# Define Task, again my Task, yours will vary
chapter_writing_task = Task(
        description = dedent(f"""
            - Write about the topic provided. 
            - Write in an informative style
            - Think about the topic before writing
            - Provide alternative viewpoints to your topic, cover all the differant points of view.
            """
        ),
        expected_output = dedent(f"""
            Well crafted paragraphs of varying length and sentences of varying length and structure, thoughtfully written and referenced in APA style, to discuss the current area of focus. 
            Focus on facts, do not make up information.
            All information presented must include an APA referenced citation in a references section.
            Discuss each topic from multiple viewpoints when possible.
            """
        ),
        agent = draft_writer
    )

inputs = [chapter.get('title') for chapter in chapters if chapter]  # Ensure chapter is not None create inputs list from the json chapters
process_topics_and_save(inputs)  # Process and save outputs for each chapter from the inputs list

with open(filename, 'a') as file: # writing out the conclusion.
    file.write(f"{conclusion}\n\n")  
    # print("Final content saved")      # more prints to help debugging.
    
``` 


As I said down and dirty.  Currently I have not tried to work with more than 1 agent or process. Probably can be done. Did not need it for this use case. Originally was working with 4 files but just got lazy and decided to just make it work, so threw them all together into one file, except for llm.py

so in a nutshell draft_writer.execute_task(chapter_writing_task, topic) does all the work, it requires task, context and tools as arguments, agent and tools are in my task and agent definitions so I only had to pass the task and context(topic).

Hope this helps you figure it out yourself.

