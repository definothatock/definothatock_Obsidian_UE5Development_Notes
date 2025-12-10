THE AI behaviour consist of 4 main categories (in order):
 - Perception
 - Cache
 - Evaluation
 - Execution

Brief introduction to the categories:
Perception (AI Perception)
The main functionality is provided by engine plugin "AI Perception". Perception provides raw data input for the AI system.

Cache (Working Memory)
The is a data layer that stores parameters of individual Actors. It provides basic queries for Decision making.

Evaluation (Evaluator)
This is the algorithm layer, judge the motivation and booleans via processing entry data from Cache.

Execution (AI State Tree)
This layer that perform actions. Essentially a State Tree that react base on judgments made from previous layer.


[[Creature Behaviour Machine]]
[[AI Perceptions]]
[[Working Memory]]