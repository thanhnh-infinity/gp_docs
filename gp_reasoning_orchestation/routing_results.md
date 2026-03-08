# Router Agent — Routing Results

> Generated on 2026-03-04 16:29 UTC  

> Total questions: **139**


---

## COMPETITIVE PRICING INTELLIGENCE

`workflow_id = competitive_pricing_intelligence_rheem_workflow`

| # | Question | Level | Task Type | Thinking | Conf | Model | Fallbacks |
|--:|----------|-------|-----------|----------|-----:|-------|-----------|
| 1 | is there anything advance insightful information I can get from above report to decide which location and I should update the Price for Rheem to dominant AO Smith ? | L1_SIMPLE | TaskType.ANALYSIS | low | 0.52 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 2 | Which market has the highest and lowest rebate % ? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.79 | gemini-3-flash | claude-haiku-4-5 |
| 3 | Which SKUs truly belong together across systems? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.64 | gemini-3-flash | claude-haiku-4-5 |
| 4 | Which channels are structurally margin-negative? | L1_SIMPLE | TaskType.MATH | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 5 | What does our actual price architecture look like across all channels? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.53 | gemini-3-flash | claude-haiku-4-5 |
| 6 | Which SKUs drive system-level margins? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 7 | Where is cannibalization eating gross profit? | L0_LOOKUP | TaskType.MATH | minimal | 0.77 | gemini-3-flash | claude-haiku-4-5 |
| 8 | What's the true causal driver of low margin? Price? Channel? Warranty? | L2_ANALYTICAL | TaskType.MATH | high | 0.83 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 9 | What would have happened last quarter if we raised prices earlier? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.71 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 10 | Which promotions had negative ROI? | L1_SIMPLE | TaskType.MATH | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 11 | Where are we margin-leaking? | L0_LOOKUP | TaskType.MATH | minimal | 0.83 | gemini-3-flash | claude-haiku-4-5 |
| 12 | What are the SKUs with the widest range in prices? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 13 | what is the average elasticity of the market? | L1_SIMPLE | TaskType.MATH | low | 0.90 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 14 | How could we improve the price recommendation engine? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.85 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |


---

## WARRANTY INTELLIGENCE RHEEM

`workflow_id = warranty_intelligence_rheem_workflow`

| # | Question | Level | Task Type | Thinking | Conf | Model | Fallbacks |
|--:|----------|-------|-----------|----------|-----:|-------|-----------|
| 1 | Is there anything insightful information warranty I can get from these report data in workflow ? | L1_SIMPLE | TaskType.ANALYSIS | low | 0.71 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 2 | Which things I should focus on to improve the user experience with Rheem products ? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.59 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 3 | How many dollars in claims were submitted in the Great Lakes region for heat pumps in the past year? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.53 | gemini-3-flash | claude-haiku-4-5 |
| 4 | How much dollars in claims were submitted in the NY METRO region for heat pumps in FY2025? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.59 | gemini-3-flash | claude-haiku-4-5 |
| 5 | Which stores in NYC METRO region have the highest rate of claims submitted within 2025 ? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.62 | gemini-3-flash | claude-haiku-4-5 |
| 6 | Which stores in NYC METRO region have the highest rate of claims submitted within 2025 and 2024? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.59 | gemini-3-flash | claude-haiku-4-5 |
| 7 | What product families create the most warranty risk? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.78 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 8 | Where are we inconsistent across Region vs Retail vs Distributor? | L1_SIMPLE | TaskType.COMPARISON | low | 0.68 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 9 | Which installers create systemic warranty liability? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.68 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 10 | What are the top 5 latent drivers of warranty cost? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 11 | Which new failure modes are not covered today? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 12 | Where is the real root cause? Component? Assembly? Installer? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.85 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 13 | Which subsystems are causing cascading claims? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 14 | What failure patterns are emerging pre-visibility? | L0_LOOKUP | TaskType.ANALYSIS | minimal | 0.58 | gemini-3-flash | claude-haiku-4-5 |
| 15 | Which stores will see a spike in the next 4 weeks? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.76 | gemini-3-flash | claude-haiku-4-5 |
| 16 | Which installers are trending toward problematic behavior? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 17 | Which claims represent a new failure mode? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 18 | Which part combinations predict early failure? | L1_SIMPLE | TaskType.CREATIVE | - | 0.68 | claude-haiku-4-5 | gemini-3-flash, gemini-3-pro |
| 19 | Which installers should NOT install specific SKUs? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 20 | Which installers are driving warranty cost per region? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.57 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 21 | Which claims violate the contract? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 22 | Which parts fail together and why? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 23 | Which regions have structural early-failure risk? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.65 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 24 | Which product families are misclassified? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 25 | If I raise price by 3%, what happens to warranty cost and channel conflict? | L2_ANALYTICAL | TaskType.MATH | high | 0.51 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |


---

## PRICING SIMULATOR INTELLIGENCE

`workflow_id = pricing_simulator_intelligence_workflow`

| # | Question | Level | Task Type | Thinking | Conf | Model | Fallbacks |
|--:|----------|-------|-----------|----------|-----:|-------|-----------|
| 1 | pricing simulator intelligence workflow for company: Rheem | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 2 | Which is the most optimal pricing scenario for Rheem? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 3 | Can you design a complete new pricing scenario for Rheem to maximize the profit ? | L3_COMPLEX | TaskType.CREATIVE | - | 0.90 | claude-opus-4-6 | gemini-3.1-pro, claude-sonnet-4-6 |
| 4 | Can you design a new pricing scenario for Rheem to maximize the market share ? | L3_COMPLEX | TaskType.CREATIVE | - | 0.90 | claude-opus-4-6 | gemini-3.1-pro, claude-sonnet-4-6 |
| 5 | What are the key assumptions you are making for this scenario? | L0_LOOKUP | TaskType.ANALYSIS | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 6 | What is the most optimal pricing scenario for Rheem ? Give me insightful information about Scenario 4 and Scenario 2. | L0_LOOKUP | TaskType.ANALYSIS | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 7 | Give me top 10 records ranking by profit for Rheem for each Scenarios 2 ,3 and 4 | L1_SIMPLE | TaskType.MATH | low | 0.90 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 8 | Provide the top 10 largest opportunities | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 9 | Where are the largest price differences? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 10 | Where are the largest price differences between Home Depot and Lowe's? | L1_SIMPLE | TaskType.COMPARISON | low | 0.59 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 11 | On average, in which markets is Home Depot cheaper and where is it more expensive? | L0_LOOKUP | TaskType.MATH | minimal | 0.51 | gemini-3-flash | claude-haiku-4-5 |
| 12 | What are the patterns of opportunity for scenario 4? | L0_LOOKUP | TaskType.ANALYSIS | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 13 | Can you calculate scenario 2 & 3's financial impact as a percentage of the entire portfolio for an apple-to-apple comparison to Scenario 1 and 4? | L2_ANALYTICAL | TaskType.MATH | high | 0.85 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 14 | What is the average elasticity by market? Please provide straight average and then weighted by current units | L1_SIMPLE | TaskType.MATH | low | 0.90 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 15 | Can you summarize scenario 4 by product segment? | L1_SIMPLE | TaskType.ANALYSIS | low | 0.71 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 16 | what pricing actions would benefit Home Depot the most? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.82 | gemini-3-flash | claude-haiku-4-5 |
| 17 | Can you design another scenario which would optimize Home Depot's margin instead? | L3_COMPLEX | TaskType.CREATIVE | - | 0.85 | claude-opus-4-6 | gemini-3.1-pro, claude-sonnet-4-6 |
| 18 | For scenario 4, can you highlight the top products, groups of products, and markets driving the total profit pool? | L2_ANALYTICAL | TaskType.MATH | high | 0.78 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 19 | Which offers will THD/Lowes accept? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |


---

## COMPOSITION WORKFLOW (Pricing + Competitive + Warranty + Supply Chain)

`workflow_id = pricing_simulator_competitive_warranty_composition_workflow`

| # | Question | Level | Task Type | Thinking | Conf | Model | Fallbacks |
|--:|----------|-------|-----------|----------|-----:|-------|-----------|
| 1 | Show me total spend for "Regal Rexnord Corp" this year, but break it out by parent supplier and then by local supplier name under that parent | L1_SIMPLE | TaskType.MATH | low | 0.90 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 2 | Show me YTD direct spend for Ternium and Copeland this year, and compare it to YTD last year for the same period | L1_SIMPLE | TaskType.COMPARISON | low | 0.56 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 3 | If geopolitical risk in Eastern Europe escalates, which 10 suppliers should we proactively dual-source? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.63 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 4 | Recommend the top 10 substitution opportunities that reduce both cost and geopolitical risk without increasing lead time. | L3_COMPLEX | TaskType.CREATIVE | - | 0.81 | claude-opus-4-6 | gemini-3.1-pro, claude-sonnet-4-6 |
| 5 | Which sole-source items have at least one technically compatible alternative supplier? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.76 | gemini-3-flash | claude-haiku-4-5 |
| 6 | We're buying the same fasteners from 14 suppliers, how much could we save if we consolidated to the top 3 lowest price suppliers? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.63 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 7 | For WHD business unit, what are the top 10 Direct spend categories this year, and let me drill down from Level 1 to Level 3 for the top category. | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 8 | Trend weekly direct spend for gas valves from Jan 1 last year to today for Resideo Technologies Inc. Flag weeks where spend is 2x the trailing 8-week average | L2_ANALYTICAL | TaskType.MATH | high | 0.62 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 9 | What is the direct spend and total quantity for item '0012345' by month for the last 18 months by business unit | L1_SIMPLE | TaskType.MATH | low | 0.90 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 10 | For ACD, compare direct vs indirect spend by month for the past 2 years starting Jan 1, and show the top 3 categories contributing to each | L2_ANALYTICAL | TaskType.COMPARISON | - | 0.73 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 11 | In Fasteners category, for the top 20 items by spend this year, which items had the largest unit price increase compared to last year? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 12 | For January 2026, in LATAM, what is spend for each plant, and what is the ratio of direct vs indirect | L2_ANALYTICAL | TaskType.COMPARISON | - | 0.62 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 13 | In the last 12 weeks, did any WHD purchases from Grainger include item numbers containing '1000? Show me those weeks and the total spend. | L1_SIMPLE | TaskType.MATH | low | 0.62 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 14 | Are suppliers in higher-risk countries also showing longer lead times? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 15 | If we improved on-time delivery by 5% for our bottom quartile suppliers, what would the estimated impact be on expedited freight spend? | L2_ANALYTICAL | TaskType.MATH | high | 0.85 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 16 | ACD's direct spend mix shifted this year, what categories replaced what? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.76 | gemini-3-flash | claude-haiku-4-5 |
| 17 | Are we paying materially different prices for the same item across regions due to foreign exchange timing? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.64 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 18 | Copper index dropped 12% this year, why didn't our copper-based components drop in price? | L2_ANALYTICAL | TaskType.MATH | high | 0.65 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 19 | Okay- based on this data what are the 3 most critical suppliers? What do they produce for me, what is the spend and which product model do they touch? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.62 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 20 | Can you tell me if I have a supplier concentration risk? Who are my critical suppliers, and what are my biggest spend categories? Is there anything I might be overlooking? | L2_ANALYTICAL | TaskType.ANALYSIS | - | 0.55 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 21 | Can you summarize the key insights from the supply chain domain? Anything I should pay specific attention to? | L2_ANALYTICAL | TaskType.ANALYSIS | - | 0.63 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 22 | Can you tell me which suppliers carry elevated risk scores? What type of components are they producing? Is any of them connected directly to the products (components) you highlighted in the highest warranty variance markets? | L2_ANALYTICAL | TaskType.MATH | high | 0.80 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 23 | Are any of them connected directly to the products in high warranty variance markets? | L0_LOOKUP | TaskType.MATH | minimal | 0.54 | gemini-3-flash | claude-haiku-4-5 |
| 24 | Based on the supplier data we have. Is there anything that would indicate or explain the failures we're saying (high % of claims for the 24V40FNEV model)? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.80 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 25 | What other components go to 24V40FNEV? Do we have any other suppliers for that? Can you pull the product specification too? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.53 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 26 | Okay, so what is included in the supplier data then? Anything else that might help me with the warranty & claims? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.53 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 27 | I want you to do: Root Cause Mapping: Join the edp_spend_f_curated (spend) with module_3c_top_models_by_impact (claims) to see if specific high-cost components from high-risk suppliers are driving the $3.87M impact seen in the 24V40FNEV model | L3_COMPLEX | TaskType.ANALYSIS | - | 0.69 | claude-opus-4-6 | gemini-3.1-pro, claude-sonnet-4-6 |
| 28 | Okay. Let's first identify high-spend vendors with low stability scores. Then let's analyze the spend categories | L1_SIMPLE | TaskType.ANALYSIS | low | 0.71 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |


---

## ASP REASONING (SIG-PRE)

`workflow_id = gp_contextual_reasoning_engine_problems_workflow`

| # | Question | Level | Task Type | Thinking | Conf | Model | Fallbacks |
|--:|----------|-------|-----------|----------|-----:|-------|-----------|
| 1 | Which nurse is assigned to the morning shift on day 1? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.76 | gemini-3-flash | claude-haiku-4-5 |
| 2 | What is the total cost of the optimal model? | L1_SIMPLE | TaskType.MATH | low | 0.90 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 3 | Compare model 1 and model 2 | L0_LOOKUP | TaskType.COMPARISON | minimal | 0.56 | gemini-3-flash | claude-haiku-4-5 |
| 4 | What blocks are clear at time step 3? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 5 | Which constraints are violated? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 6 | What happens if nurse n1 cannot work night shifts? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.82 | gemini-3-flash | claude-haiku-4-5 |
| 7 | Add a constraint that no nurse works more than 2 consecutive days | L3_COMPLEX | TaskType.PROGRAM_EXECUTION | - | 0.50 | sig-pre-0.1 | claude-opus-4-6, gemini-3.1-pro, claude-sonnet-4-6 |
| 8 | What if we add a new nurse n4 available for morning and evening shifts? | L3_COMPLEX | TaskType.PROGRAM_EXECUTION | - | 0.90 | sig-pre-0.1 | claude-opus-4-6, gemini-3.1-pro, claude-sonnet-4-6 |
| 9 | Remove the constraint on maximum overtime and re-solve | L3_COMPLEX | TaskType.PROGRAM_EXECUTION | - | 0.62 | sig-pre-0.1 | claude-opus-4-6, gemini-3.1-pro, claude-sonnet-4-6 |
| 10 | What is the cost impact if we increase max_step from 5 to 8? | L2_ANALYTICAL | TaskType.ANALYSIS | - | 0.67 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 11 | Which rules are causing model 2 to have higher cost than model 1? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.52 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 12 | Can we get a solution where block d is never placed on the table? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.68 | gemini-3-flash | claude-haiku-4-5 |


---

## DEEP ANALYTICAL (cross-workflow)

`workflow_id = default`

| # | Question | Level | Task Type | Thinking | Conf | Model | Fallbacks |
|--:|----------|-------|-----------|----------|-----:|-------|-----------|
| 1 | What is the best achievable profit if we cap share loss at 2%? | L2_ANALYTICAL | TaskType.MATH | high | 0.62 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 2 | Which price moves increase profit without hurting warranty or channel? | L1_SIMPLE | TaskType.MATH | low | 0.60 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 3 | What is our profit frontier by product family? | L1_SIMPLE | TaskType.MATH | low | 0.52 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 4 | What are the top objections likely to arise? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.90 | gemini-3-flash | claude-haiku-4-5 |
| 5 | What's the probability of deal success under different structures? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.82 | gemini-3-flash | claude-haiku-4-5 |
| 6 | Which SKUs have a rising warranty signal before it becomes visible? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.71 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 7 | Which failure spikes are real vs noise? | L0_LOOKUP | TaskType.COMPARISON | minimal | 0.53 | gemini-3-flash | claude-haiku-4-5 |
| 8 | Which anomalies require root-cause investigation? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 9 | Where is the system drifting out of normal behavior? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.82 | gemini-3-flash | claude-haiku-4-5 |
| 10 | Why is the warranty cost 2x higher in THD vs Wholesale? | L2_ANALYTICAL | TaskType.COMPARISON | - | 0.78 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 11 | What channel factors explain regional differences? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.79 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 12 | Is this failure an installer error, a defective part, or misuse? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.76 | gemini-3-flash | claude-haiku-4-5 |
| 13 | Which failure modes are systemic vs localized? | L2_ANALYTICAL | TaskType.COMPARISON | - | 0.66 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 14 | How do we reduce warranty cost by 10-20% without degrading service levels? | L2_ANALYTICAL | TaskType.MATH | high | 0.85 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 15 | Which repairs should be preemptively approved? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |
| 16 | What is the cost-minimizing service policy for each region? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.90 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 17 | What will warranty cost be next quarter by region? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.54 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 18 | Which installers will cause budget overruns? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 19 | Which new product launches create warranty risk? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.75 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 20 | Where should we enforce more training? | L0_LOOKUP | TaskType.RETRIEVAL | minimal | 0.85 | gemini-3-flash | claude-haiku-4-5 |


---

## STRATEGIC / GROWTH (cross-workflow)

`workflow_id = default`

| # | Question | Level | Task Type | Thinking | Conf | Model | Fallbacks |
|--:|----------|-------|-----------|----------|-----:|-------|-----------|
| 1 | What if Menards undercuts us? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.66 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 2 | If we raise the price by 2%, what is Menards likely to do? | L1_SIMPLE | TaskType.MATH | low | 0.52 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 3 | What competitor will react first? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.62 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 4 | Where are competitors structurally unable to follow? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.68 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 5 | What counter-structure satisfies both margin & retailer targets? | L1_SIMPLE | TaskType.MATH | low | 0.66 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 6 | Which promos are margin leaks? | L0_LOOKUP | TaskType.MATH | minimal | 0.80 | gemini-3-flash | claude-haiku-4-5 |
| 7 | Which promos risk channel conflict? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.85 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 8 | Where does promo misalignment cause retailer tension? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.82 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 9 | Are price drops attracting high-risk installers? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.60 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 10 | What price actions reduce long-term service costs? | L1_SIMPLE | TaskType.CREATIVE | - | 0.65 | claude-haiku-4-5 | gemini-3-flash, gemini-3-pro |
| 11 | If we launch SKU X in region Y, what warranty cost will it create? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.85 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 12 | Which design choices reduce long-term service cost? | L3_COMPLEX | TaskType.CREATIVE | - | 0.65 | claude-opus-4-6 | gemini-3.1-pro, claude-sonnet-4-6 |
| 13 | Which SKU/store combos have elevated warranty risk before any claim occurs? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.80 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 14 | Which promotions drive low-quality demand? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.72 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 15 | Are returns early indicators of failures? | L1_SIMPLE | TaskType.RETRIEVAL | low | 0.60 | gemini-3-flash | claude-haiku-4-5, gemini-3-pro |
| 16 | Which returns predict 70%+ chance of warranty claim? | L2_ANALYTICAL | TaskType.MATH | high | 0.52 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 17 | Which SKUs have negative lifetime value even if initial margins look strong? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.71 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 18 | Where does discounting create long-term warranty liabilities? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.60 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 19 | Which preventive actions reduce warranty by 20%? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.85 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |
| 20 | Which installer behaviors drive systemic failures? | L2_ANALYTICAL | TaskType.RETRIEVAL | high | 0.85 | gemini-3-pro | claude-sonnet-4-6, gemini-3.1-pro |
| 21 | Where is design change required? | L2_ANALYTICAL | TaskType.CREATIVE | - | 0.85 | claude-sonnet-4-6 | gemini-3-pro, gemini-3.1-pro |


---

## Summary (139 questions)

### Level Distribution

| Level | Count | % | Bar |
|-------|------:|--:|-----|
| L0_LOOKUP | 48 | 34.5% | ████████████████████████████████████████████████ |
| L1_SIMPLE | 41 | 29.5% | █████████████████████████████████████████ |
| L2_ANALYTICAL | 41 | 29.5% | █████████████████████████████████████████ |
| L3_COMPLEX | 9 | 6.5% | █████████ |

### Model Distribution

| Model | Count | % | Bar |
|-------|------:|--:|-----|
| claude-haiku-4-5 | 2 | 1.4% | ██ |
| claude-opus-4-6 | 6 | 4.3% | ██████ |
| claude-sonnet-4-6 | 17 | 12.2% | █████████████████ |
| gemini-3-flash | 87 | 62.6% | ███████████████████████████████████████████████████████████████████████████████████████ |
| gemini-3-pro | 24 | 17.3% | ████████████████████████ |
| sig-pre-0.1 | 3 | 2.2% | ███ |
