# MP2 Reflection

**GitHub:** https://github.com/uditbatra87/week-9-mp2-mini-rag-project

## What worked

The paragraph-based chunking worked really well for me. I tried splitting the text on double newlines (`\n\n`) instead of just cutting at every 500 characters, and this made a huge difference. I also added a simple check to detect section headers - basically if a line is short and doesn't end with punctuation like a period or comma, I treat it as a header. This way each chunk has proper context about what section it belongs to.

The reason this worked so well is that when I retrieve chunks later, they make sense on their own. Like when someone asks "Who was Vincent Spaulding?", the retrieved chunk includes the whole paragraph about him being John Clay, not just half a sentence that gets cut off. I set the target to around 500 characters which seemed like a sweet spot - big enough to have complete information but small enough to be precise.

When I tested my questions, the retrieval kept pulling the exact sections I expected. For example, the question about the murder method got the chunk explaining the snake and the bell-rope mechanism, not some random chunk from the same story.

## What didn't work

My first attempt at chunking was pretty bad. I just did simple sliding windows - basically `text[0:500]`, then `text[420:920]`, etc. This was a mistake because it would cut paragraphs right in the middle of sentences. 

Two big problems happened:
1. When I embedded these broken chunks, the embeddings didn't really capture what the chunk was about since the text was incomplete
2. When the LLM got these chunks as context, the answers were confusing or incomplete because important details were split between chunks

The worst example was when I asked about how Doctor Roylott killed Julia. The fixed-size chunks split the explanation - one chunk had the snake part and another had the bell-rope part, so the answer didn't explain the full mechanism. I realized I needed to keep paragraphs together to maintain the logic flow.

## What I'd change

If I had more time (like 5 more hours), I would definitely try adding BM25 for keyword matching along with the embeddings. Right now I'm only using dense embeddings from OpenAI, which work great for "similar meaning" searches but sometimes miss exact name matches.

The Sherlock Holmes stories have a lot of specific names like "Jabez Wilson" or "Stoke Moran" and objects like "Blue Carbuncle". I noticed that sometimes if you search for these exact terms, the semantic search might not rank them at the top because it's looking at overall meaning instead of exact words.

From the reference code I saw something called Reciprocal Rank Fusion (RRF) which basically combines results from both keyword search (BM25) and semantic search (embeddings). This would probably help a lot. The idea is dense embeddings answer "what's this generally about?" while BM25 answers "does this mention this exact thing?".

I would also try using a reranker model to look at the top 10 results and pick the best 3. I read somewhere that this can improve accuracy by 10-20% but didn't have time to implement it.

## One surprise

This was genuinely surprising to me - the LLM automatically knows how to cite sources without me writing complicated code for it!

I just added `[Source: <title> — <section>]` at the start of each chunk I sent to the LLM, and GPT-4o-mini started including proper citations in its answers naturally. I thought I would need to write parsing logic or give very detailed instructions, but it just worked.

Even cooler - when the question was ambiguous and chunks from multiple stories got retrieved, the model would cite both sources and sometimes even say when they had different information. I didn't program this behavior at all.

I think this happens because the model has seen lots of examples of cited text during training, so it recognizes the pattern and knows what to do with it. The practical benefit is my code is much simpler - I don't need regex patterns or special citation extraction. Just format the context clearly and the LLM handles the rest.

This makes me realize that sometimes simple formatting is more powerful than complex prompting tricks.

---

## Additional Notes

### My Implementation Details
- Used paragraph splitting for chunking (~72 chunks total from 5 stories)
- OpenAI's text-embedding-3-small model (1536 dimensions)
- Stored everything in local Qdrant using COSINE similarity
- Retrieved top 3 chunks for each question
- Used GPT-4o-mini with temperature=0.3 (not too random, not too rigid)
- Tested with 5 questions - 2 provided + 3 I wrote myself

### How I Designed My Questions
1. **Easy one (q3)**: Just asked about one person (Henry Baker) - tests if basic retrieval works
2. **Medium one (q4)**: Asked about a specific location detail (where the photo was hidden)
3. **Hard one (q5)**: Asked about the murder mechanism which needs info from multiple parts - tests if it can connect different details

### Validation Approach
The validation script checks if the retrieved chunks come from the right story file. This is a good way to test because if it pulls the right story, the answer will probably be correct. The "expected facts" check is extra - sometimes the LLM paraphrases correctly but uses different words, so I don't worry too much if that doesn't match perfectly.

### Things I Could Improve Later
- Try adding BM25 keyword search with the embeddings
- Use a reranker to improve the final ranking
- Maybe include text from neighboring chunks to give more context
- Add automatic scoring metrics like ROUGE to measure answer quality

### Cost
- Embeddings: about $0.01 
- LLM answers: about $0.05
- Total: less than $0.20 for everything

### What I Learned
The biggest lesson for me is that getting the chunks right matters way more than which model you use. Spending time on good chunking strategy, understanding the text structure, and formatting context properly gave me better results than just using a fancier model would have. The quality of your chunks is more important than the size of your embeddings or how sophisticated your retrieval is.
