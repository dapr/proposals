# Allow Workflow API to bypass irrelevant beta api criteria

This is a proposal for allowing Dapr Workflow (WF) to reach beta status without having to meet the criteria that is irrelevant to the workflow api specifically (API Lifecycle)[[https://github.com/dapr/proposals/blob/main/templates/lifecycle.md](https://github.com/dapr/proposals/blob/main/templates/lifecycle.md#beta)]. 

## Background
Dapr WF differs from other Dapr APIs in that there is only one component implamentation of the API, the built-in actor-based wf engine. Therefore this makes it difficult to meet stable criteria that related to the following:

- [ ] Minimum of N (three?) implementations of this building block 
- [ ] Conformance tests updated to match any API changes that have been made
- [ ] Conformance tests exercise both positive and negative cases 
- [ ] Certification tests for implementations 

I propose we allow the Dapr WF to mimic the stability process of Dapr Actors, and remove these criteria as stable requirements for the the workflow feature. Without doiong so, workflow will not have a clear path to beta and overall stability.  


