---
layout: post
title: "How to Perform Open-Heart Surgery on Your Code (and Not Get Fired)"
date: 2025-10-06 10:00:00 -0500
categories: engineering
author: Jose Rodriguez
---

Have you ever looked at a piece of your code and thought, "I really need to replace this, but if I touch it, the whole thing will explode"? It feels like trying to change a car's engine while driving on the highway. Scary, right?

You have this old, clunky part. Maybe it is a library everyone used five years ago. And you want to use the new, super-fast, shiny thing. But the old part is connected to everything. It is like a giant octopus with its tentacles in every corner of your application.

Well, do not be a scared cat! I am here to tell you about a magic trick. It is called **Branch by Abstraction**. It is a fancy name, but it is actually a simple and safe way to perform surgery on your code without breaking everything.

### The Big Idea: Build a New Bridge Next to the Old One

Imagine you have an old, shaky wooden bridge. Your entire town uses it every day. You cannot just blow it up. People need to get to work!

So, what do you do? You build a strong, new metal bridge right next to the old one. For a while, you have two bridges. Once the new bridge is ready and tested, you open it. Then, you tell everyone, "Please use the new bridge!" Finally, you can safely take down the old wooden one. No one gets wet. Everyone is happy.

Branch by Abstraction is exactly that, but for your code.

### Our Example: Goodbye LangChain, Hello Claude!

Let’s get real. Imagine we have an agent application. It is very smart, and it uses the LangChain library to talk to AI models. Our code is full of `import langchain` and `langchain.chains...`. It is everywhere!

But now, we want to use the official Claude SDK. It is newer, maybe faster, and we think it is cooler. How do we switch without stopping our app from working for weeks?

Here is our step-by-step guide for scared cats.

#### Step 1: Create a "Wrapper" (Our Universal Remote)

First, we create a new file. Let’s call it `ai_service.py`. This file is our "abstraction" or "wrapper". Think of it like a universal remote control.

Inside this file, we create a class.

```python
# ai_service.py
from langchain.llms import OpenAI # The old stuff

class MyAIService:
    def __init__(self):
        self._langchain_model = OpenAI(api_key="...")

    def ask_question(self, prompt):
        # For now, it just calls the old LangChain code
        print("DEBUG: Using the old LangChain bridge!")
        return self._langchain_model.predict(prompt)

```

Our new class `MyAIService` has a method `ask_question`. Inside, it just uses the old LangChain code. It is a simple pass through.

#### Step 2: Use the Wrapper Everywhere

Now, we go on a little adventure through our codebase. Everywhere we see code that calls LangChain directly, we change it. Instead of calling LangChain, we call our new `MyAIService`.

So, this old code:

```python
# old_file.py
from langchain.llms import OpenAI

model = OpenAI(api_key="...")
response = model.predict("Hello, how are you?")
```

Becomes this new code:

```python
# new_file.py
from ai_service import MyAIService

ai = MyAIService()
response = ai.ask_question("Hello, how are you?")
```

This is the most work, but it is safe. Our app still works exactly the same. It is just talking to our wrapper now, not directly to LangChain.

#### Step 3: Secretly Build the New Thing

Now the fun begins. We can add the Claude SDK to our project. In our `ai_service.py` file, we can write the code to talk to Claude.

```python
# ai_service.py
from langchain.llms import OpenAI # The old stuff
import anthropic # The new shiny Claude SDK!

class MyAIService:
    def __init__(self):
        self._langchain_model = OpenAI(api_key="...")
        self._claude_client = anthropic.Anthropic(api_key="...")

    def ask_question(self, prompt):
        # For now, it still calls the old LangChain code
        print("DEBUG: Using the old LangChain bridge!")
        return self._langchain_model.predict(prompt)

    def _ask_question_with_claude(self, prompt):
        # Here is our new, secret code!
        print("DEBUG: Using the new Claude bridge!")
        message = self._claude_client.messages.create(
            model="claude-3-opus-20240229",
            max_tokens=1024,
            messages=[
                {"role": "user", "content": prompt}
            ]
        )
        return message.content
```

See? We added the new code, but the main `ask_question` method has not changed. The rest of our app is safe and sound, still using the old LangChain bridge. We can test our `_ask_question_with_claude` method separately. No pressure.

#### Step 4: The Big, Magical Switch

This is the magic moment. The big reveal. Are you ready?

We change one line in our `ai_service.py` file.

```python
# ai_service.py

# ... (all the other code is the same)

class MyAIService:
    # ...

    def ask_question(self, prompt):
        # THE SWITCH!
        # return self._langchain_model.predict(prompt) # Goodbye old bridge
        return self._ask_question_with_claude(prompt) # Hello new bridge!

    # ...
```

That is it! We just changed what our "universal remote" does. Our entire application now uses the new Claude SDK. And it did not even notice the change. It just called `ai.ask_question()` like it always does.

If something goes wrong, what do we do? Easy! We just switch that one line back. Phew. Safe.

#### Step 5: Clean Up Your Room

After running the app for a while, we are happy. The new Claude code works perfectly. It is time to clean up.

*   Delete the old LangChain code from `ai_service.py`.
*   Remove the LangChain library from your project requirements.
*   Enjoy your clean, modern, and shiny new code!

### Why Is This So Cool?

*   **No Big Bang.** You avoid a massive, scary change. You make small, safe steps.
*   **Your App Is Always Working.** The app is never broken. You can even do this over many days or weeks.
*   **Happy Team.** Your teammates can continue to work without you breaking the application. Nobody will be angry with you.

So, next time you have to do scary surgery on your code, remember the Branch by Abstraction strategy. It is a developer’s best friend. Now go be a brave cat!
