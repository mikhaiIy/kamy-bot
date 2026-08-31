# Kamy-Bot domain context

## Kamy-Bot

The provisional name for the new reusable workflow platform. Kamy-Bot accepts work from external intake sources, routes that work through GitHub issues, and coordinates approved implementation through agent workflows.

## Kamy Jemput

The existing production-specific orchestrator. Kamy Jemput remains stable while Kamy-Bot is designed and tested independently.

## Project

A repository and its workflow policy as managed by Kamy-Bot. A project has its own GitHub repository, ownership configuration, labels, runtime state, and external-tool adapters.

## Intake source

An external system that submits a work request to Kamy-Bot. Obsidian daily notes are one intake source. Intake sources create or update work requests; they do not decide implementation scope.

## Work request

A user-originated request that enters the workflow. A work request becomes a GitHub issue before implementation work begins.

## Workflow run

One recorded attempt to move a work request through the Kamy-Bot lifecycle. A run includes state transitions, external actions, evidence, failures, and approvals.

## Adapter

A project- or provider-specific implementation behind a Kamy-Bot interface. GitHub, SSSF, Herdr, browser verification, and intake sources are adapters rather than workflow rules.

## Workflow auditor

A read-only analysis process that examines prior workflow runs and proposes improvements. It cannot change production policy without a reviewed and approved change.
