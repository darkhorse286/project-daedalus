# Project DAEDALUS

**Project DAEDALUS** is an evidence-driven hybrid optimization runtime for dynamic fleet-routing workloads.

It evaluates a routing problem, extracts workload features, applies a configurable scheduling objective, selects an appropriate solver backend, executes the job, persists telemetry, and produces an evidence report explaining the decision.

The goal is not to claim universal quantum advantage.

The goal is to build a production-style runtime that can prove when cheap classical methods are sufficient, when more expensive solvers are justified, and when experimental quantum-style execution should be rejected.

## Core Thesis

Most routing problems should never touch a quantum computer.

A serious optimization runtime creates value by deciding when advanced backends are useful, when they are wasteful, and when they should be excluded by policy.

## MVP Objective

Build an employer-signaler proof of concept demonstrating:

* C++ runtime and solver core
* C# ASP.NET Core control plane API
* Queue-based asynchronous job execution
* PostgreSQL persistence
* Configurable cost / latency / quality scheduler policy
* Classical routing baselines
* QUBO simulated annealing backend
* Structured telemetry
* OpenTelemetry traces
* Reproducible synthetic routing scenarios
* HTML evidence reports suitable for technical writing

## System Components

* Daedalus API
* Daedalus Worker
* Daedalus Core
* Daedalus Scheduler
* Daedalus Evidence Log
* Daedalus Report Generator
* Daedalus CLI
* Python Solver Adapter

## Initial Runtime Flow

1. Submit a routing job through the API or CLI.
2. Persist the routing problem and scheduler configuration.
3. Publish the job to the queue.
4. Worker consumes the job.
5. Core extracts workload features.
6. Scheduler scores eligible solvers.
7. Worker executes the selected solver.
8. Runtime evaluates quality, cost, runtime, and regret.
9. Evidence log is persisted.
10. HTML report is generated.

## Included in MVP

* Routing problem model
* Synthetic workload generator
* Classical baseline solvers
* QUBO formulation layer
* Configurable scheduler objective
* Hybrid scheduler policy engine
* Runtime execution engine
* Result quality and regret analysis
* Telemetry and evidence log
* HTML evidence report generator
* Developer CLI
* Docker Compose environment

## Deferred

* IBM Quantum Runtime execution
* QAOA hardware execution
* Learned AI scheduler policy
* Kubernetes deployment
* Multi-tenant security
* Full dashboard
* Real traffic API integration

## First Article Thesis

**QED: Most Routing Problems Should Never Touch A Quantum Computer**

A production-style scheduler should treat quantum execution like an expensive backend, not a magical default. The runtime earns credibility by rejecting experimental solvers when cheaper classical methods satisfy the configured cost, latency, and quality objective.
