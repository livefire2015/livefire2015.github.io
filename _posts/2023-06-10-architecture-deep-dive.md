---
layout: post
title: "Architecture Deep Dive: Building a Sub-Second Decision Engine"
date: 2023-06-10
categories: [fintech, opa, payment-gateway]
---

_This is **Part 3** of a five-part series on building a real-time payment gateway with Open Policy Agent (OPA). Read the series: [Part 1]({% post_url 2022-03-15-speed-vs-complexity %}), [Part 2]({% post_url 2022-09-20-enter-opa %}), [Part 4]({% post_url 2024-02-28-beyond-yes-no %}), [Part 5]({% post_url 2025-11-05-production-lessons %})._

In the [previous articles]({% post_url 2022-09-20-enter-opa %}), we established the critical need for agile policy management in real-time payment gateways and introduced Open Policy Agent (OPA) as the transformative solution. Now, we delve into the architectural decisions and implementation details that allowed Torpago to build a high-performance, sub-second decision engine powered by OPA [1].

## Design Decisions That Matter

Building a real-time payment gateway with OPA involves crucial design choices that directly impact performance, scalability, and maintainability. One of the most significant decisions we faced was how to integrate OPA into our existing Go-based backend.

### Go Library vs. REST API: The Critical Decision

OPA offers two primary integration methods: as a standalone service accessed via a REST API, or embedded directly into an application as a library [2]. For Torpago, the choice was clear: **embedding OPA as a Go library**.

Here's a detailed comparison that highlights why this decision was critical for our payment gateway:

| Feature           | OPA as Go Library                                    | OPA as REST API                                      |
| :---------------- | :--------------------------------------------------- | :--------------------------------------------------- |
| **Performance**   | Sub-millisecond evaluation times, direct memory access [1] | Network latency overhead, serialization/deserialization |
| **Language**      | Native Go integration, leverages existing codebase   | Language-agnostic, requires HTTP client              |
| **Deployment**    | Single binary, simpler deployment and scaling        | Separate service, additional operational overhead    |
| **Security**      | Minimal attack surface, no network calls             | Requires secure network communication (TLS, auth)    |
| **Complexity**    | Tighter coupling, but optimized for performance      | Looser coupling, but adds distributed system complexity |

Given that our backend is primarily written in Go and the stringent performance requirements of a payment gateway (demanding high throughput and extremely low latency), using OPA as a Go library (`github.com/open-policy-agent/opa/rego`) was the optimal choice. This approach eliminated network latency, allowing for direct, in-memory policy evaluation, which is essential for achieving sub-millisecond decision times [1].

### System Architecture Walkthrough: From Transaction to Decision

Our OPA-powered payment gateway is designed to process transactions with maximum efficiency and policy enforcement at every critical juncture. The request flow is meticulously orchestrated:

1.  **Request Ingestion:** Transactions originate from various **Card Networks** and are routed through **Issuer-Processors** before reaching the Torpago Gateway.
2.  **Initial Processing & Data Enrichment:** Upon arrival at the Torpago Gateway, the raw transaction data undergoes initial parsing and validation. Crucially, this stage involves **PostgreSQL integration** to enrich the transaction context with customer-specific information, historical data, and other relevant metadata. This enriched data forms the `input` for OPA.
3.  **Policy Evaluation (OPA Integration):** The enriched transaction data is then passed to the embedded OPA engine. Here, a sophisticated **Policy Evaluation** process takes place. OPA evaluates the `input` against a comprehensive set of Rego policies, which include:
    *   **Tenant-specific rules:** Policies tailored to individual bank partners or merchants.
    *   **Whitelist/Blacklist rules:** Dynamic lists for approved or denied entities.
    *   **Fraud detection policies:** Real-time rules to identify and mitigate fraudulent activities.
4.  **Decision Pipeline:** OPA returns a rich decision object, not just a simple allow/deny, but an object containing metadata (as discussed in the next article). This decision object then feeds into a **Decision Pipeline** that orchestrates subsequent actions.

### Implementation Details

#### Go Integration Code

Integrating OPA as a Go library is straightforward. A typical Go function would take the query string and an input struct, then return a decision. This allows for seamless integration with existing Go services [1].

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"github.com/open-policy-agent/opa/rego"
)

// Example input struct for a transaction
type TransactionInput struct {
	Amount float64 `json:"amount"`
	Mcc    string  `json:"mcc"`
	// ... other transaction details
}

func evaluatePolicy(ctx context.Context, policy string, input TransactionInput) (interface{}, error) {
	// Create a new Rego object
	r := rego.New(
		rego.Query("data.example.allow"), // Query the 'allow' rule in 'example' package
		rego.Module("example.rego", policy),
		rego.Input(input),
	)

	// Prepare the query
	pq, err := r.PrepareForEval(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to prepare for evaluation: %w", err)
	}

	// Evaluate the policy
	rs, err := pq.Eval(ctx)
	if err != nil {
		return nil, fmt.Errorf("failed to evaluate policy: %w", err)
	}

	// Extract the result
	if len(rs) == 0 || len(rs[0].Expressions) == 0 {
		return nil, fmt.Errorf("no policy evaluation result")
	}
	return rs[0].Expressions[0].Value, nil
}

func main() {
	ctx := context.Background()

	// Example Rego policy
	policy := `
		package example
		default allow := false

		allow if {
			input.Amount < 100
		}
		allow if {
			input.Mcc == "5812"
		}
	`

	// Example transaction input
	input1 := TransactionInput{Amount: 50, Mcc: "1234"}
	input2 := TransactionInput{Amount: 150, Mcc: "5812"}

	result1, err := evaluatePolicy(ctx, policy, input1)
	if err != nil {
		fmt.Printf("Error evaluating policy for input1: %v\n", err)
	} else {
		fmt.Printf("Policy result for input1: %v\n", result1)
	}

	result2, err := evaluatePolicy(ctx, policy, input2)
	if err != nil {
		fmt.Printf("Error evaluating policy for input2: %v\n", err)
	} else {
		fmt.Printf("Policy result for input2: %v\n", result2)
	}
}
```

#### Performance Metrics

By embedding OPA as a Go library, we consistently achieve policy evaluation times in the order of **microseconds**. This performance is critical for handling thousands of transactions per second, ensuring that the policy engine does not become a bottleneck in the payment processing pipeline.

#### Data Flow Diagrams

(Note: A visual data flow diagram would typically be included here, illustrating the path from card networks to the final decision, highlighting OPA's role in the policy evaluation step.)

### Multi-Tenant Considerations

FinTech platforms often serve multiple clients (e.g., different banks or merchants), each with their unique set of policy requirements. Managing these diverse policies efficiently is a significant challenge. OPA addresses this through:

*   **Namespace Isolation:** Rego packages provide a natural way to achieve **bank-specific policy management**. Each client can have its own dedicated policy package, ensuring that their rules are isolated and do not interfere with others.
*   **Configuration Management:** Dynamic policy loading strategies allow us to update and deploy client-specific policies without downtime. Policies can be fetched from a central repository and hot-swapped into the OPA engine, ensuring that the latest rules are always enforced.
*   **Scalability Patterns:** The stateless nature of OPA policy evaluation, combined with its embeddability, allows for horizontal scaling of the payment gateway. Each instance can run its own OPA engine, handling thousands of transactions per second independently.

### Technical Challenges Solved

Implementing OPA in a high-performance FinTech environment presented several technical challenges, which we successfully addressed:

*   **Memory Management for Policy Compilation:** Efficiently managing the memory footprint of compiled policies, especially when dealing with a large number of client-specific rules, was crucial. We optimized policy loading and caching mechanisms to minimize memory consumption.
*   **Hot-Swapping Policies Without Downtime:** The ability to update policies without interrupting live traffic is paramount. Our dynamic policy loading system ensures that new policies can be deployed seamlessly, maintaining continuous service availability.
*   **Monitoring and Observability Integration:** Comprehensive monitoring of OPA's performance and policy evaluation outcomes is essential. We integrated OPA metrics into our existing observability stack, providing real-time insights into policy enforcement and potential issues.

## Conclusion

By carefully selecting OPA as an embedded Go library and designing a robust system architecture, Torpago has successfully built a sub-second decision engine for its real-time payment gateway. This architecture not only meets the stringent performance demands of the FinTech industry but also provides the flexibility and agility required to adapt to rapidly changing business needs. In the next article, we will explore how OPA's rich decision capabilities and event-driven actions transform simple allow/deny decisions into intelligent, context-aware business processes.

---

## References

[1] Torpago. (2025). *Breaking the Rules: Overcoming Challenges in Building a Real-Time Payment Gateway with OPA*. FinTech DevCon Speaker Slide. [PDF]

[2] Open Policy Agent. (n.d.). *Running OPA*. Retrieved from [https://www.openpolicyagent.org/docs/latest/#running-opa](https://www.openpolicyagent.org/docs/latest/#running-opa)

