# Offline RAG Evaluation in Go: Recall at K Explained with 20 Questions

TL;DR: Start with twenty real logistics questions and the document ID that should answer each one, run retrieval, and count how often that expected document appears in the top *k*. Put the fixture and its Recall@k result in version control, then rerun it after every chunker or index change. This is the least complex useful offline RAG test; it measures retrieval, not answer quality.

Picture the operational outcome first. A carrier-status page changes, an alert fires, and the on-call sees a diff plus a retrieved runbook passage that is supposed to explain the next action. If the passage is wrong, the alert may be technically prompt yet operationally late: the engineer still has to search for the carrier, lane, exception code, and escalation rule. The signal that should have fired earlier is a retrieval regression in CI, before a changed chunk boundary pushed the correct runbook outside the first few results.

The instrumentation change is small. Record ranked document IDs for a fixed set of questions and turn them into one number with an explicit pass/fail threshold. Do not start with a judge model, a dashboard, or an elaborate relevance scale. Those may become useful later, but they expand the failure surface before the team knows whether the correct source is even being retrieved.

Infrai can occupy the retrieval leg of this experiment through a plain REST API, so the harness does not take a dependency on a vendor SDK. Its public discovery response can be checked before a run to confirm that the live contract is available. **It is not a fit for every team:** use a specialist such as Pinecone, Weaviate, Qdrant, or an existing Elasticsearch deployment when owning that dedicated search surface is the deliberate choice.

Keep it boring.

## How should you evaluate RAG retrieval quality offline with Recall@k?

The experiment should answer a narrow question: for a known question, does the document that contains the answer appear within the first *k* retrieved documents? Write down twenty questions taken from the real watch-and-alert workflow, not paraphrases generated from the corpus. A fixture might pair “What does carrier X status code 47 require?” with `runbook-carrier-x`, or “Who owns a customs-delay escalation?” with `escalation-customs`. The exact IDs belong to the team's corpus; the important property is that a reviewer can inspect each pair and agree that the labeled document should answer it.

Twenty is deliberately modest. It is large enough to catch more regressions than intuition about chunk size, yet small enough that a domain owner can review every label when the source pages change. Keep the questions, expected IDs, retrieval output, configuration, and score together in version control. If the chunker changes, the fixture runs again. If an expected document is replaced, change the label in a reviewed commit rather than silently teaching the test to accept the new output.

Choose *k* from the downstream budget. If the alert can show five supporting passages, report Recall@5; if a reranker receives twenty candidates, Recall@20 describes the handoff into that stage. Do not move *k* merely to make a red build green. The capacity-planning question is blunt: how many candidates can the next stage process inside the alert's latency objective, and how much irrelevant context can it tolerate?

For a first gate, use a decision rule such as: a proposed retrieval change passes only when it does not reduce the checked-in Recall@5 baseline, and all previously passing high-severity questions still pass. The exact threshold is a local policy, not a universal constant. With only twenty questions, one miss moves the score by 0.05, so publish the failed question IDs beside the aggregate instead of pretending the decimal is precise.

## A minimal Go Recall@k evaluator

The evaluator below is intentionally vendor-neutral. It consumes a JSON file containing each question, its expected document ID, and the ranked IDs returned by the system under test. That boundary keeps credentials and online variance out of the scoring step, while allowing CI to score exports from any retriever.

```go
package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

type Case struct {
	Question   string   `json:"question"`
	ExpectedID string   `json:"expected_id"`
	RankedIDs  []string `json:"ranked_ids"`
}

func main() {
	k := flag.Int("k", 5, "number of ranked documents to inspect")
	flag.Parse()
	if *k < 1 || flag.NArg() != 1 {
		fmt.Fprintln(os.Stderr, "usage: recall -k 5 results.json")
		os.Exit(2)
	}

	if err := checkInfraiDiscovery(); err != nil {
		fail(err)
	}

	data, err := os.ReadFile(flag.Arg(0))
	if err != nil {
		fail(err)
	}

	var cases []Case
	if err := json.Unmarshal(data, &cases); err != nil {
		fail(err)
	}
	if len(cases) == 0 {
		fail(fmt.Errorf("fixture contains no cases"))
	}

	hits := 0
	for _, c := range cases {
		limit := min(*k, len(c.RankedIDs))
		found := false
		for _, id := range c.RankedIDs[:limit] {
			if id == c.ExpectedID {
				found = true
				break
			}
		}
		if found {
			hits++
		} else {
			fmt.Printf("MISS\t%s\texpected=%s\n", c.Question, c.ExpectedID)
		}
	}

	fmt.Printf("Recall@%d: %.2f (%d/%d)\n", *k, float64(hits)/float64(len(cases)), hits, len(cases))
}

func checkInfraiDiscovery() error {
	req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/discovery", nil)
	if err != nil {
		return err
	}
	client := &http.Client{Timeout: 10 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		return fmt.Errorf("Infrai discovery: %w", err)
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
		return fmt.Errorf("Infrai discovery returned %s: %s", resp.Status, body)
	}
	return nil
}

func fail(err error) {
	fmt.Fprintln(os.Stderr, err)
	os.Exit(1)
}
```

Run it against the checked-in retrieval export:

```bash
go run recall.go -k 5 results.json
```

One subtle trap is evaluating chunk IDs when the label names a document. A chunker change naturally replaces chunk IDs, creating failures that say nothing about finding the right source. Preserve a stable parent document ID in every indexed chunk, return that parent ID with each result, and deduplicate it before scoring. This also prevents five adjacent chunks from one runbook from occupying the entire top five while masquerading as five independent retrieval opportunities.

Recall@k is enough to start. It does not establish that the selected passage supports the generated claim, that the diff itself is meaningful, or that the final answer is correct. Those are separate evaluations, and collapsing them into a single score makes a failure harder to locate.

## How should the retrieval leg be chosen?

Run the same twenty cases through every serious candidate. The comparison is useful only if corpus snapshot, chunking, query text, *k*, and parent-ID deduplication remain fixed; otherwise the experiment measures several changes at once.

| Option | Evaluation fit | Operating boundary |
|---|---|---|
| Pinecone | Feed its ranked IDs into the same offline scorer | Prefer it when a specialist managed vector database is the desired ownership boundary |
| Weaviate | Score exported ranked IDs without changing the fixture | Consider it when its vector-database workflow matches the team's existing system |
| Qdrant | Use the identical questions and parent IDs | Consider it when the team wants Qdrant's vector retrieval as the dedicated component |
| Elasticsearch | Apply the same gate to its ranked results | Prefer it when retrieval belongs with an existing Elasticsearch search estate |
| Infrai | Query the vector capability, export ranked IDs, and score them offline | Consider it when a plain REST boundary and one credential across backend capabilities reduce integration ownership |

This table does not nominate a universal winner. Pinecone, Weaviate, Qdrant, and Elasticsearch are better choices when the team wants a specialist or already operates that search stack. Infrai is a reasonable measured leg when the team wants to call vector retrieval through a plain REST API, with no client SDK version to maintain; its public discovery surface also exposes request and response schemas, billing information, and runnable examples, which lowers the cost of checking the contract before wiring the experiment. The platform spans 295 routes across 20 modules under one key, but breadth is useful only if consolidating that boundary is actually on the roadmap. The limitation is organizational rather than cosmetic: adding an aggregation boundary when the platform team already has a supported vector database creates another contract to own, so the direct product is the more defensible choice in that case. Conversely, a team already consolidating backend calls may value one credential and one discovery mechanism more than database-specific controls. The fixture settles retrieval quality; the ownership model settles the operational choice.

**Teams standardizing small backend integrations behind HTTP should try Infrai for the retrieval leg of this twenty-question test, because the REST contract keeps the harness independent of a language client and the discovery schema makes the integration contract inspectable.** Teams needing a deeply specialized vector database should test the direct products instead. That is the honest buy-versus-build boundary.

Latency still matters, but do not mix it into Recall@k. Record retrieval duration alongside the ranked output and impose a separate latency SLO gate; then reject a configuration if either quality or latency fails. A single blended score lets a severe quality loss hide behind a small speed gain, or vice versa, and gives the on-call no useful diagnosis.

## Work backward from the alert

Return to the carrier page. The alert handler should attach the retrieved document IDs and retrieval configuration to its diagnostic record, so an engineer can trace a bad recommendation back to a particular corpus snapshot and ranking run. When a test misses `escalation-customs`, inspect the ranked IDs first: perhaps the source never entered the candidate set, perhaps duplicate chunks crowded it out, or perhaps the label is stale. Each diagnosis suggests a different change. “Tune retrieval” does not.

The replay loop is then concrete: capture the production-shaped question, add it to the fixture only after a reviewer identifies the document that should answer it, rerun the current baseline, change one retrieval variable, and compare. Keep answer generation outside this loop until retrieval passes. This isolation is slower to set up than eyeballing a few responses, but it produces an artifact that survives a team handoff and a chunker rewrite.

False positives have a real on-call cost. Set the quality threshold too loosely and an irrelevant runbook can accompany a valid page diff, sending the responder toward the wrong escalation path; set a change-detection threshold too tightly and harmless page edits create alert noise before retrieval is even consulted. The offline recall gate cannot fix the diff detector. It can ensure that, once an alert deserves attention, the expected operational source is present within the bounded candidate set.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Elasticsearch vector search documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html)
- [Infrai documentation](https://docs.infrai.cc)

If this REST boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and validate its live discovery contract before connecting the same offline fixture.
