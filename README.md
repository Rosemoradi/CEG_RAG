# CEG_RAG
CEG-RAG defends RAG from corpus poisoning using cross-encoder reranker [CLS] activations. With a MIL model, we detect poisoned contexts and localize risky chunks, then repair evidence by filtering/replacing passages before generation with no extra LLM calls. Tested on MS-MARCO, NQ, HotpotQA datasets for poisoned-RAG attack setup. .
