# Reflection — Day 17

1. The easiest silent failure is the `traces → Bronze` flattening step. If child spans lose `trace_id`, `parent_id`, status, or token fields, the pipeline still runs but later summaries become wrong. I would detect it with row-count checks per trace, span-tree depth checks, and alerts when successful/error outcomes suddenly disappear.

2. If decontamination is skipped, the model trains on prompts that also appear in the eval set. Offline metrics then look better than reality because the model has already seen the question-answer pattern. The lie would show up as high eval scores but weaker performance on fresh user prompts.

3. A dangerous feature is `lifetime_spend` for fraud or credit scoring. If training joins the latest value instead of the value known at the transaction time, the model sees future purchases and learns from information unavailable during serving.

4. The knowledge graph handles the multi-hop question `widget -> accessory -> Hanoi fulfillment center` well because it can traverse relationships across facts. Flat vector retrieval is enough for simple lookup questions like `what is the gadget warranty?`; a graph would be overkill there.
