---
title: "Learning Lang Graph"
date: 2025-10-31T18:54:26-04:00
draft: true
---

###### What Is LangGraph?
The first challenge I encountered in learning this library was discerning its purpose. Why LangChain and LangGraph (and LangSmith for that matter...)?

My initial understanding is that LangChain is for linear pipelines, and LangGraph is for dynamic workflows. Dynamic meaning workflows that can potentially have branches, loops, decision points, and sub-tasks.

This tool should allow me to create more powerful automations using LLMs and their associated tooling. After following the basic tutorial in the LangGraph documentation, my next problem was deciding what problem to solve with this tool.

###### What to Create?
One learns best from projects rather than staring at poorly written documentation, so the next problem to solve was determining what problem to solve. 

I landed on a simple, recurring, but irritating problem in my own life - deciding what to meal prep each week. Cooking is great but meal prep is just the worst. Cookbooks don't cater to the fitness crowd so unless you're willing to do manual conversions to eight high-protein, moderate-carb, no sugar servings with all of the ingredients weighed and measured, you're out of luck. But perhaps the robots could do this for me...

So that is what I decided to create - a simple workflow to intake an idea for meal prep, no matter how vague, and translate it into a recipe with ingredients, measurements, caloric data, and tagging. Not especially exciting but it will be enough to learn the basics of LangChain.


###### Implementation
LangGraph implements any workflow as a graph. Their recommended first step in creating your own LangGraph-powered application is to define your workflow and break it down into steps. 

For this simple project, the workflow is downright boring, bordering on linear. So linear it's hardly worth implementing in LangChain. But I'm just learning it so this is fine as a first project. I am sure my next attempts will be sickening with complexity. 

```
User Idea → Ideation Agent → Nutrition Agent → Recipe Formatter Agent → Approval → Output to file

```

Once you understand your workflow is defined, you then need to clarify what state the workflow needs to operate on. I found this a little puzzling when I read about it in the documentation. It was not until I actually implemented it that it made sense. 

Essentially the graph's state is just a dictionary. In the examples I have seen, each step's output gets its own key in the dictionary. Of course multiple steps could also modify the same key. But we will keep it simple for now.

What state would I need? My initial attempt looked like this:

```Python
from typing import TypedDict

class RecipeState(TypedDict):
	idea: str
	concept: str
	recipe: str
	tags: list[str]
	final_output: str
	approval: str
```

Simple enough. Each key roughly maps to one step in the workflow. The next task is to then define these steps, which are just functions that modify the shared `RecipeState` instance.

It's actually so simple I thought it was kind of lame. All this talk of *multi-agent pipelines* and *dynamic workflows* and oh it's just functions that update values in a dictionary. You don't even need an llm to use this. So, to recap, we have a shared dictionary, and we're about to create some functions to modify the shared dictionary.

Once we have initialized an LLM model to use:
```Python
llm = ChatOpenAI(model="gpt-4o-mini")
```

The first step is to elaborate an idea into a full recipe concept. We're using a few basic techniques to improve the quality of the response from the LLM, nothing especially exciting.

```Python
def develop_meal_idea(state: RecipeState):
	print("Developing meal idea...")
	idea = state["idea"]
	response = llm.invoke(f"""
	You are an expert bodybuilding meal designer.
	Expand the following idea into a structured meal concept:
	{idea}
	Include: main protein, cooking style, carb source, fat source, 
	flavor profile. Favor clean ingredients and avoid unhealthy 
	ingredients lacking nutritional value. Maximize flavor.
	""")
	return {"concept": response.content}
```

The only thing to note is the return statement. That dictionary will be merged with our shared state. The next two steps are really just more of the same.

```Python
def create_nutrition_profile(state: RecipeState):
	print("Creating nutrition profile...")
	concept = state["concept"]
	response = llm.invoke(f"""
	You are an expert in quantifying recipes. Quantify this 
	bodybuilding meal concept:
	{concept}
	Think very carefully and include the following in your response:
	- Output ingredients with gram weights for 8 servings.
	- Include per-serving calories, protein, carbs, fat. These values 
	  must be calculated correctly and carefully.
	- Prioritize high-protein, moderate-carb allocations.
	""")
	return {"recipe": response.content}


def format_recipe(state: RecipeState):
	print("Formatting recipe...")
	recipe = state["recipe"]
	
	response = llm.invoke(f"""
	Format this bodybuilding recipe cleanly in Markdown:
	{recipe}
	Include title, ingredients (with grams), macros, and brief cooking 
	instructions.
	Write the ingredients, macros, and instructions as bulleted lists. 
	Never use tables.
	Also include Obsidian-style tags under the title for protein 
	source, continent of origin, and country of origin if appropriate 
	(eg. #chicken-thigh #asia #thai if the recipe is for chicken pad 
	thai)
	""")
	
	return {"final_output": response.content}
```

###### Approvals Added Complexity
So far so good. I included an approval step out of curiosity. I wanted to see how it was implemented and how it would behave. I was not especially prepared for the inconvenience it caused. Perhaps I neglected to read the documentation as thoroughly as I should have. Either way, adding a manual approval step to the workflow require more effort and code than I expected.

```Python
def approval_node(state: RecipeState) -> Command[Literal["write_to_file", "cancel"]]:
	# Pause execution; payload shows up under result["__interrupt__"]
	is_approved = interrupt({
		"question": "Do you want to proceed with this action?",
		"details": state["recipe"]
	})

	# Route based on the response
	if is_approved:
		return Command(goto="write_to_file")
	else:
		return Command(goto="cancel")
```

(We'll define the file writing and cancel nodes further down). When running the program with this node in the graph, it exited silently. No request for approval, no failure, nothing. What was going on?

I had incorrectly assumed that this approval node would pause the program's execution and prompt me for an approval. Not so. Instead, the approval node calls `interrupt`, which pauses the graph and returns an `__interrupt__` from `app.invoke()`. You have to handle this eventuality yourself. 

After much faffing, I ended up with this monstrosity.
```Python
checkpointer = MemorySaver()
app = graph.compile(checkpointer=checkpointer)
user_input = input("Enter your meal idea: ")
config = {"configurable": {"thread_id": "recipe-run-1"}}

input_or_resume: RecipeState | Command = cast(RecipeState, {"idea": user_input})

while True:
	result = app.invoke(input_or_resume, config=config)
	if "__interrupt__" in result:
		interrupts = result["__interrupt__"]
			if isinstance(interrupts, list) and interrupts:
				payload = interrupts[0]
			else:
				payload = interrupts

			if isinstance(payload, dict):
				print(payload.get("question", "Approval required"))
				if "details" in payload and payload["details"] is not None:
					print(payload["details"])
			else:
				print("Approval required")
			approved = input("Approve? (y/n): ").strip().lower().startswith("y")
			input_or_resume = Command(resume=approved)
			continue
	break
```

With that figured out, the workflow just needed the additional steps defined, and some assemblage.
```Python
def write_to_file(state: RecipeState):
	print("Writing final recipe to file...")
	final_output = state["final_output"]
	# Extract the recipe name from the first line (assuming it's a markdown title)
	lines = final_output.strip().split('\n')
	recipe_name = "recipe"

	for line in lines:
		if line.strip().startswith('#'):
			recipe_name = line.strip().lstrip('#').strip()
			recipe_name = recipe_name.replace(' ', '_').replace('/', '_').replace('\\', '_')
			recipe_name = ''.join(c for c in recipe_name if c.isalnum() or c in ('_', '-')).lower()
			break

	filename = f"{recipe_name}.md"
	print(f"Saving to {filename}...")
	with open(filename, "w") as f:
		f.write(final_output)
	return {}

  
def cancel(state: RecipeState):
	print("Operation cancelled by user.")
	return {}
```

And assembly:
```Python
graph = StateGraph(RecipeState)
graph.add_node("develop_idea", develop_meal_idea)
graph.add_node("create_nutrition", create_nutrition_profile)
graph.add_node("format_recipe", format_recipe)
graph.add_node("approval", approval_node)
graph.add_node("write_to_file", write_to_file)
graph.add_node("cancel", cancel)

graph.add_edge("develop_idea", "create_nutrition")
graph.add_edge("create_nutrition", "format_recipe")
graph.add_edge("format_recipe", "approval")
graph.add_edge("write_to_file", END)
graph.add_edge("cancel", END)
graph.set_entry_point("develop_idea")
```

Tremendous. This simple program is now ready to run and save me some time.