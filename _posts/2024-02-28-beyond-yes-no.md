---
layout: post
title: "Beyond Yes/No: Rich Decisions and Event-Driven Actions"
date: 2024-02-28
categories: [fintech, opa, payment-gateway]
---

_This is **Part 4** of a five-part series on building a real-time payment gateway with Open Policy Agent (OPA). Read the series: [Part 1]({% post_url 2022-03-15-speed-vs-complexity %}), [Part 2]({% post_url 2022-09-20-enter-opa %}), [Part 3]({% post_url 2023-06-10-architecture-deep-dive %}), [Part 5]({% post_url 2025-11-05-production-lessons %})._

In our journey so far, we've explored the [critical need for agile policy management]({% post_url 2022-03-15-speed-vs-complexity %}) in real-time payments, [introduced Open Policy Agent (OPA)]({% post_url 2022-09-20-enter-opa %}) as the solution, and delved into the [architecture of our sub-second decision engine]({% post_url 2023-06-10-architecture-deep-dive %}). Now, we move beyond simple "allow" or "deny" decisions to uncover one of OPA's most powerful features: the ability to generate **rich, metadata-driven decisions** that trigger intelligent, event-driven actions [1].

## Metadata-Driven Intelligence: The Power of #METADATA

In a sophisticated FinTech environment, a binary decision is often insufficient. We need to know *why* a decision was made, its context, and what actions should follow. This is where OPA's `#METADATA` feature shines. By annotating Rego rules with custom, structured information, we can transform simple policies into self-documenting, context-aware enforcement tools [1].

### Rich Decision Objects: Tags, Scores, Severity, and Actions

With `#METADATA`, we can associate a wealth of information with each policy rule, including:

*   **Tags:** For categorizing rules (e.g., "fraud," "compliance," "credit_limit").
*   **Scores:** To quantify the risk associated with a transaction.
*   **Severity Levels:** To indicate the urgency of a policy violation (e.g., "low," "medium," "high").
*   **Actions:** A list of recommended downstream actions (e.g., "notify_customer," "decline_transaction," "freeze_card").

This metadata is then included in the decision object returned by OPA, providing a rich, structured output that goes far beyond a simple boolean [1].

### Code Example: Metadata in Action

Let's see how this works in practice. Here is a Rego rule that detects transactions involving high-risk merchants, annotated with rich metadata:

```rego
# METADATA
# title: High-Risk Merchants
# description: Detects transactions involving merchants flagged as high-risk
# custom:
#   tag: "tr_high_risk_merchants"
#   score: "10"
#   severity: "high"
#   actions: ["notify_customer", "notify_admin", "decline_transaction"]

risk[{"title": "High-Risk Merchants"}] {
    # Rule logic here: checks if the input merchant is in the high-risk list
    input.merchant.id in data.high_risk_merchants
}
```

When this rule is triggered, the decision object returned by OPA will contain the `custom` metadata, providing a clear and actionable context for the decision.

### Self-Documenting Policies

An often- overlooked benefit of this approach is that it makes policies **business-readable and self-documenting**. A business analyst or compliance officer can look at the `#METADATA` block and immediately understand the purpose, risk, and consequences of a rule, without needing to decipher complex Rego logic.

## CloudEvents Integration: From Decisions to Actions

Once OPA provides a rich decision, the next step is to translate that decision into action. To keep our payment gateway fast and responsive, we employ an **event-driven architecture** using **CloudEvents** [1]. Instead of handling complex business logic synchronously, we publish the OPA decision as a CloudEvent, which is then consumed by downstream services.

### Synchronous vs. Asynchronous: Keeping the Gateway Fast

By offloading the business process execution to asynchronous event handlers, we ensure that the payment gateway's primary responsibility—making a fast decision—is not compromised. The gateway simply publishes the event and moves on to the next transaction, maintaining high throughput and low latency.

### Event-Driven Architecture: A System Diagram

(Note: A visual data flow diagram would typically be included here, showing the OPA decision being published as a CloudEvent and consumed by various downstream services, such as a notification service, a compliance logging service, and a risk management service.)

### Action Routing: Intelligent Business Processes

The rich metadata in the OPA decision allows for intelligent **action routing**. The event-driven system can inspect the `actions` array in the decision and route the event to the appropriate services. For example:

*   If the `actions` array contains `"notify_customer"`, the event is routed to a notification service that sends an email or SMS to the cardholder.
*   If it contains `"freeze_card"`, the event is routed to a card management service that temporarily suspends the card.
*   If it contains `"decline_transaction"`, the gateway itself can take immediate action.

## Real-World Use Cases

This combination of rich decisions and event-driven actions unlocks a wide range of powerful use cases:

*   **Card Freezing Workflows:** Automatically freeze a card when a high-risk transaction is detected, and notify the customer to confirm the transaction.
*   **Smart Notification Systems:** Send targeted, context-aware notifications to customers based on the specific policy that was triggered.
*   **Automated Audit Trails:** Create detailed, structured audit logs for every transaction, including the policy decisions and actions taken, which is invaluable for compliance reporting.
*   **Risk Scoring and Pattern Detection:** Feed the rich decision data into an analytics platform to identify emerging fraud patterns and continuously improve risk models.

## Implementation Patterns

To build a robust and scalable event-driven system, we follow several key implementation patterns:

*   **Flexible Event Payload Design:** The CloudEvent payload is designed to be flexible and extensible, allowing us to add new metadata and actions without breaking existing consumers.
*   **Error Handling and Retry Mechanisms:** We implement robust error handling and retry logic to ensure that events are not lost in case of downstream service failures.
*   **Monitoring Decision-to-Action Latency:** We closely monitor the end-to-end latency from the initial policy decision to the final action, ensuring that our event-driven system meets its performance SLAs.

## Conclusion

By leveraging OPA's rich decision capabilities and an event-driven architecture with CloudEvents, we have transformed our payment gateway from a simple decision-maker into an intelligent, context-aware system. This approach not only enhances our security and compliance posture but also enables us to create more sophisticated and responsive user experiences. In the final article of this series, we will share the key lessons learned from three years of running this system in production, including our strategies for sandbox testing, feedback loops, and our vision for the future of policy-as-code.

---

## References

[1] Torpago. (2025). *Breaking the Rules: Overcoming Challenges in Building a Real-Time Payment Gateway with OPA*. FinTech DevCon Speaker Slide. [PDF]


