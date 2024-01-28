# Gorilla Openfunctions

💡 Comes with Parallel Function Calling!  
🚀 Try it out on [Colab](https://colab.research.google.com/drive/16M5J2H9F8YQora_W2PDnp120slZH-Mqd?usp=sharing)   
📣 Read more in our [OpenFunctions release blog](https://gorilla.cs.berkeley.edu/blogs/4_open_functions.html)

## Introduction
Gorilla OpenFunctions extends Large Language Model(LLM) Chat Completion feature to formulate 
executable APIs call given natural language instructions and API context.

## Models Available
|Model | Functionality|
|---|---|
|gorilla-openfunctions-v0 | Given a function, and user intent, returns properly formatted json with the right arguments|
|gorilla-openfunctions-v1 | + Parallel functions, and can choose between functions|

All of our models are hosted on our Huggingface UC Berkeley gorilla-llm org: [gorilla-openfunctions-v0](https://huggingface.co/gorilla-llm/gorilla-openfunctions-v0) and [gorilla-openfunctions-v1](https://huggingface.co/gorilla-llm/gorilla-openfunctions-v1).

## Example Usage

1. OpenFunctions is compatible with OpenAI Functions

```bash
!pip install openai==0.28.1
```

2. Point to Gorilla hosted servers

```python
import openai

def get_gorilla_response(prompt="Call me an Uber ride type \"Plus\" in Berkeley at zipcode 94704 in 10 minutes", model="gorilla-openfunctions-v0", functions=[]):
  openai.api_key = "EMPTY"
  openai.api_base = "http://luigi.millennium.berkeley.edu:8000/v1"
  try:
    completion = openai.ChatCompletion.create(
      model="gorilla-openfunctions-v1",
      temperature=0.0,
      messages=[{"role": "user", "content": prompt}],
      functions=functions,
    )
    return completion.choices[0].message.content
  except Exception as e:
    print(e, model, prompt)
```

3. Pass the user argument and set of functions, Gorilla OpenFunctions returns a fully formatted json

```python
query = "Call me an Uber ride type \"Plus\" in Berkeley at zipcode 94704 in 10 minutes"
functions = [
    {
        "name": "Uber Carpool",
        "api_name": "uber.ride",
        "description": "Find suitable ride for customers given the location, type of ride, and the amount of time the customer is willing to wait as parameters",
        "parameters":  [{"name": "loc", "description": "location of the starting place of the uber ride"}, {"name":"type", "enum": ["plus", "comfort", "black"], "description": "types of uber ride user is ordering"}, {"name": "time", "description": "the amount of time in minutes the customer is willing to wait"}]
    }
]
get_gorilla_response(query, functions=functions)
```

4. Expected output

```bash
uber.ride(loc="berkeley", type="plus", time=10)
```

## Running OpenFunctions Locally

If you want to Run OpenFunctions locally, here is the prompt format that we used: 

```python
def get_prompt(user_query, functions=[]):
  if len(functions) == 0:
    return f"USER: <<question>> {user_query}\nASSISTANT: "
  functions_string = json.dumps(functions)
  return f"USER: <<question>> {user_query} <<function>> {functions_string}\nASSISTANT: "
```

Note: Use the `get_prompt` format only if you are hosting it Locally. If you are using the Berkeley hosted models through the Chat-completion API, we do this in the backend, so you don't have to do this. The model is supported in Hugging Face 🤗 Transformers and can be run up locally:

```python
import json
import torch
from transformers import AutoTokenizer, AutoModelForCausalLM, pipeline

def get_prompt(user_query: str, functions: list = []) -> str:
    """
    Generates a conversation prompt based on the user's query and a list of functions.

    Parameters:
    - user_query (str): The user's query.
    - functions (list): A list of functions to include in the prompt.

    Returns:
    - str: The formatted conversation prompt.
    """
    if len(functions) == 0:
        return f"USER: <<question>> {user_query}\nASSISTANT: "
    functions_string = json.dumps(functions)
    return f"USER: <<question>> {user_query} <<function>> {functions_string}\nASSISTANT: "

# Device setup
device : str = "cuda:0" if torch.cuda.is_available() else "cpu"
torch_dtype = torch.float16 if torch.cuda.is_available() else torch.float32

# Model and tokenizer setup
model_id : str = "gorilla-llm/gorilla-openfunctions-v0"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype=torch_dtype, low_cpu_mem_usage=True)

# Move model to device
model.to(device)

# Pipeline setup
pipe = pipeline(
    "text-generation",
    model=model,
    tokenizer=tokenizer,
    max_new_tokens=128,
    batch_size=16,
    torch_dtype=torch_dtype,
    device=device,
)

# Example usage
query: str = "Call me an Uber ride type \"Plus\" in Berkeley at zipcode 94704 in 10 minutes"
functions = [
    {
        "name": "Uber Carpool",
        "api_name": "uber.ride",
        "description": "Find suitable ride for customers given the location, type of ride, and the amount of time the customer is willing to wait as parameters",
        "parameters":  [
            {"name": "loc", "description": "Location of the starting place of the Uber ride"},
            {"name": "type", "enum": ["plus", "comfort", "black"], "description": "Types of Uber ride user is ordering"},
            {"name": "time", "description": "The amount of time in minutes the customer is willing to wait"}
        ]
    }
]

# Generate prompt and obtain model output
prompt = get_prompt(query, functions=functions)
output = pipe(prompt)

print(output)
```

## Self-Hosting OpenFunctions

This section provides a guide on how to self-host the OpenFunctions model on your local machine. The example server enables the deployment of the OpenFunctions model using FastAPI and uvicorn, while the example client demonstrates how to interact with this local server using the OpenAI package.

### Setting Up Your Local Server

The server API endpoint mirrors the interface of the `openai.ChatCompletion.create` API call, ensuring compatibility with clients using the OpenAI package.

Ensure you have the required libraries:
```bash
!pip install fastapi uvicorn transformers torch
```

Similar to the process outlined in the "Running OpenFunctions Locally" section, the model is loaded from HuggingFace for local serving. 

```python
from fastapi import FastAPI
from pydantic import BaseModel
import torch
from transformers import pipeline, AutoModelForCausalLM, AutoTokenizer
import uvicorn
import json

# Define the API request model
class ChatCompletionRequest(BaseModel):
    model: str
    temperature: float
    messages: list
    functions: list = []

# Initialize the FastAPI app
app = FastAPI()

# Device setup
device = "cuda:0" if torch.cuda.is_available() else "cpu"
torch_dtype = torch.float16 if torch.cuda.is_available() else torch.float32

# Model and tokenizer setup
model_id = "gorilla-llm/gorilla-openfunctions-v0"
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype=torch_dtype, low_cpu_mem_usage=True)

# Move model to device
model.to(device)

# Pipeline setup
pipe = pipeline(
    "text-generation",
    model=model,
    tokenizer=tokenizer,
    max_new_tokens=128,
    batch_size=16,
    torch_dtype=torch_dtype,
    device=device,
)

# Function to generate prompt
def get_prompt(user_query: str, functions: list) -> str:
    """
    Generates a conversation prompt based on the user's query and a list of functions.

    Parameters:
    - user_query (str): The user's query.
    - functions (list): A list of functions to include in the prompt.

    Returns:
    - str: The formatted conversation prompt.
    """
    if len(functions) == 0:
        return f"USER: <<question>> {user_query}\nASSISTANT: "
    functions_string = json.dumps(functions)
    return f"USER: <<question>> {user_query} <<function>> {functions_string}\nASSISTANT: "

# API endpoint
@app.post("/chat/completions")
async def chat_completion(request: ChatCompletionRequest):
    user_query = request.messages[0]['content']
    prompt = get_prompt(user_query, request.functions)
    output = pipe(prompt)
    generated_text = output[0]['generated_text']

    # Return in the format expected by OpenAI package
    return {
        "id": "0",
        "object": "text_completion",
        "created": 0,
        "model": request.model,
        "choices": [{"text": generated_text, "index": 0, "logprobs": None, "finish_reason": "length"}]
    }

# Run the server
if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Setting Up the Client

The example client below demonstrates how to interact with the locally hosted OpenFunctions model using `openai.ChatCompletion.create`, akin to using Gorilla hosted servers.

```python
import openai

def get_gorilla_response(prompt="Call me an Uber ride type \"Plus\" in Berkeley at zipcode 94704 in 10 minutes", model="gorilla-openfunctions-v0", functions=[]):
    openai.api_key = "EMPTY"
    openai.api_base = "http://localhost:8000"  # Point to the local server
    try:
        completion = openai.ChatCompletion.create(
            model="gorilla-openfunctions-v0",
            temperature=0.0,
            messages=[{"role": "user", "content": prompt}],
            functions=functions,
        )
        return completion.choices[0].text
    except Exception as e:
        print(e, model, prompt)

# Example usage
query: str = "Get the latest news headlines from CNN."
functions = [
    {
        "name": "News Headlines",
        "api_call": "news.get_headlines",
        "description": "Retrieve the latest news headlines from a specific news source.",
        "parameters": {
            "type": "object",
            "properties": {
                "source": {
                    "type": "string",
                    "description": "The news source, e.g. CNN"
                }
            },
            "required": [
                "source"
            ]
        }
    }
]

resp = get_gorilla_response(query, functions=functions)
print(resp)
```

## Evaluation

<img src="https://github.com/ShishirPatil/gorilla/blob/gh-pages/assets/img/blog_post_4_OpenFunctions_Distribution.png" width=50% height=50%>



## Contributing

All the models, and data used to train the models is released under Apache 2.0.
Gorilla is an open source effort from UC Berkeley and we welcome contributors. 
Please email us your comments, criticism, and questions. More information about the project can be found at [https://gorilla.cs.berkeley.edu/](https://gorilla.cs.berkeley.edu/)
