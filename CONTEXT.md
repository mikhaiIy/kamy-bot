# Kamy-Bot domain context

## Agentic Development Workflow

An Agentic Development Workflow is a software delivery process in which agents perform bounded development actions, produce evidence, and pass work through explicit workflow states. Humans retain control of product direction, ownership, approved scope, and exceptional failures.

## Agentic framework

The implementation framework that Kamy-Bot uses to construct and run its Agentic Development Workflow. Kamy-Bot does not commit to one framework. The framework is an implementation choice behind Kamy-Bot interfaces.

## Agentic software factory

A software system that receives software requests, coordinates agents through an Agentic Development Workflow, validates evidence, and prepares reviewed changes for merge.

## Kamy-Bot

The provisional name for the reusable Agentic Development Workflow platform. Kamy-Bot supports multiple Projects and accepts requests from external Intake sources.

## Kamy Jemput

The existing production-specific orchestrator. Kamy Jemput remains stable while Kamy-Bot is designed and tested independently.

## Project

A software repository and its workflow policy as managed by Kamy-Bot. Each Project has its own repository, ownership policy, workflow labels, profile, and external integrations.

## Project profile

The Project-specific description that tells Kamy-Bot which repository, ownership policy, workflow labels, Intake sources, and external integrations belong to one Project.

## Intake source

An external system that submits a Work request to Kamy-Bot. Obsidian daily notes are one Intake source. An Intake source provides request content and source identity. It does not decide implementation scope.

## Work request

The user-originated unit of work that enters the Agentic Development Workflow. A Work request becomes a GitHub issue before implementation work begins. Experiment 001 uses one GitHub issue for one Work request. Later Discovery or Triage may create approved child issues.

## GitHub issue

The shared work record that connects a Work request to product decisions, technical scope, approvals, agent work, and delivery status.

## Workflow

The ordered set of states and actions that moves a Work request from intake to a reviewed change. The long-term Workflow is Intake, Discovery, Triage, approval, agent implementation, review, and merge.

## Workflow run

One recorded attempt to move one Work request through the Workflow. A run includes state transitions, external actions, evidence, failures, and approvals.

## Human decision

An explicit decision that only an authorized human can make. Product direction, ownership, approved scope, exceptional recovery, and merge authorization are Human decisions when the Workflow requires them.

## Approved scope

The behavior, repository paths, and acceptance conditions that a Human decision authorizes an agent to change. An agent cannot expand Approved scope by inference.

## Worker

An agent session that performs one assigned Workflow action. A Worker executes Approved scope and reports Evidence. A Worker does not decide product direction, ownership, Approved scope, or merge authorization.

## Evidence

The observable record that proves a Workflow action occurred and met its conditions. Evidence can include structured results, repository state, external state, verification output, and Human decisions.

## Adapter

A Project- or provider-specific implementation behind a Kamy-Bot interface. Intake sources, GitHub, agent execution, browser verification, and durable state are Adapters. Workflow rules remain in Kamy-Bot.

## Controller version

One released implementation of the Kamy-Bot controller. A Project can have only one active Controller version processing its Work requests at a time.

## Workflow auditor

A read-only analysis process that examines prior Workflow runs and proposes improvements. It cannot change production policy, start production work, or approve its own proposal.

## Full Agentic Development Workflow

```text
Intake source
→ Work request
→ GitHub issue
→ Discovery
→ Triage
→ Human approval
→ Agent implementation
→ Pull request
→ Integration review
→ Required Human or visual approval
→ Merge
```

Experiment 001 designs only the first foundation slice. It does not implement the full Agentic Development Workflow.
