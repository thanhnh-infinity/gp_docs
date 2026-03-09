# WIP

## Router AI:
- I believe we should implement a Router Agent to detect the best model for each task
- I believe we should have the option to let users change the hyper-parameters (model, fastest, thinkingDeep, etc.) over-the-fly.

## Improve in Semantic Query Planner:
- I believe that given more example to map <question> --> <required schema/fields/record/filtering> the model will be able to generalize better.  Implement a white-paper distionary like that will have to improve the results. That could be TODO for future experiments.
- I believe and did some works: With the Formal Enterprise Ontology involved in the project, we can also leverage that to improve the mapping from question to schema/fields/record/filtering and make our reasoning better to delivery the Plan.
- At the moment there  are 3 mechanism in Semantic Query Planner ( default as (1) now)
  - 1. Batch of few-shot examples to guide the LLM to produce the plan for all partitions ( fast and good for small output)
  - 2. Separate call for each partition with few-shot examples ( slower than 1 and good for large output)
  - 3. Deterministic approach to generate Plan based on Semantic Searnc + Pre-trained Model ( fastest but not good for complex question)
  -  We might need to develop a smart agent to know where/when we should use either {1,2,3} based on the question and the expected output size.

## Improve in Grounded Answering Engine:
- I believe that area of improvement is to have more Multi-Agent System and Symbolic AI RE + Ontology
- We can also leverage the Formal Enterprise Ontology involved in the project to improve the reasoning capability of the Grounded Answering Engine.

## Improve in Structure Data Retriever:
- Retrieve data now is in Workflow Output, we need to extend it to use our IRS and GP Database like Snowflake or ElasticSearch.
- For Data Retriever from another sources, we should able to use caching mechanism to improve the performance.

## Quality Management:
- Register that for workflow and test