# Cordis for AI Engineers and Software Developers

## Summary

Cordis is a runtime for building applications from plugins that can be added, removed, replaced, and reconfigured while the process keeps running. It is designed for systems where plugins share one process and therefore cannot rely on process termination to clean up their resources.

Its core idea is simple:

1. A plugin declares the services it needs.
2. A plugin registers every resource it creates together with a cleanup operation.
3. Cordis activates the plugin only when its dependencies are available.
4. If a dependency disappears or changes, Cordis deactivates affected plugins in a safe order.
5. During unload or failed startup, Cordis runs recorded cleanup operations in reverse order.

For an LLM agent platform, this means tools, memory backends, model providers, policy modules, planners, and integrations can become managed runtime components instead of loosely connected callbacks. An agent host could replace a model provider, disable a faulty tool, reload generated code, or switch a tenant-specific service without restarting the entire process.

Cordis does not make arbitrary code reversible or safe. Cleanup must still be correct, external actions such as sending a message or charging a card cannot generally be undone, and untrusted plugins still require process or container isolation. Cordis is best understood as a lifecycle and dependency control plane for in-process components.

## Background

### Why ordinary plugin systems become difficult

Many plugin systems begin with a small interface:

- Load a module.
- Pass it an application object.
- Let it register handlers.
- Optionally call a shutdown hook later.

This works until the application needs live changes. A plugin may have registered event listeners, routes, timers, background jobs, tools, database connections, middleware, child plugins, or services used by other plugins. Removing it safely now requires answers to several questions:

- Which resources belong to this plugin?
- In what order should they be released?
- Which other plugins depend on it?
- Can dependents keep using it while they shut down?
- What happens if startup fails halfway through?
- What happens if configuration changes during asynchronous startup?
- Will hot reload leave duplicate handlers or stale service references?

Traditional lifecycle callbacks make the plugin author remember all of this manually. Restarting the process avoids some cleanup problems, but discards caches, sessions, connections, and in-flight work. That is especially costly for long-running agent hosts.

### Why this matters for LLM agents

An agent system is rarely just one model call. A production runtime may contain:

- model and embedding providers;
- tool registries and MCP clients;
- retrieval and memory services;
- planners, routers, and evaluators;
- authorization and policy components;
- tracing, billing, and rate limiting;
- user or tenant-specific extensions;
- generated code and experimental behaviors.

These pieces change at different speeds. Tools can become unavailable. Credentials rotate. A provider may be replaced. A newly generated component may fail during activation. An administrator may want to disable one integration without interrupting every active agent.

Cordis addresses the in-process lifecycle problem created by these changes.

## 1. What Cordis does

Cordis combines four responsibilities that are often implemented separately.

### 1.1 It tracks plugin-owned effects

A plugin creates an effect when it changes the running application. Examples include:

- registering an event listener;
- adding a route or command;
- starting a timer or worker;
- publishing a service;
- opening a resource with a close operation;
- loading a child plugin.

With Cordis, the plugin registers the change through `ctx.effect` and returns or yields a disposer. Cordis stores the disposer and later runs all disposers in last-in, first-out order.

Conceptually:

~~~ts
ctx.effect(() => {
  const timer = setInterval(runHealthCheck, 10_000)
  return () => clearInterval(timer)
})
~~~

This is more reliable than placing unrelated setup and teardown code in separate callbacks. The cleanup is defined next to the operation it reverses, and Cordis owns the accumulated cleanup stack.

### 1.2 It manages service dependencies

Plugins declare which services they require and which services they provide. A consumer remains inactive until all required services exist. If a provider is removed or replaced, Cordis identifies the consumers bound to it and moves them through deactivation and, when possible, reactivation.

This is dynamic dependency injection. The important difference from common dependency injection containers is that resolution is not limited to startup. Cordis continues managing dependency availability throughout the life of the process.

### 1.3 It coordinates lifecycle order

Cordis represents each plugin instance as a fiber. A fiber can be inactive, loading, active, or unloading. It records:

- the plugin instance and parent;
- required and provided services;
- the exact providers selected for the current activation;
- accumulated cleanup operations;
- lifecycle state and failure information.

Provider removal follows a guarded sequence:

1. Stop advertising the provider to new consumers.
2. Notify consumers that their committed dependency is leaving.
3. Let those consumers unload while the provider is still usable for cleanup.
4. Wait until committed consumers have drained.
5. Run the provider's own cleanup.

This avoids a common shutdown bug: destroying a database, transport, or model client before its consumers have finished releasing their own resources.

### 1.4 It reconciles desired configuration with running state

Cordis includes a loader that treats configuration as desired state. It can compare the desired plugin tree with the currently running tree and apply targeted changes:

- create or remove a plugin;
- enable or disable it;
- update configuration;
- change isolation or interception settings;
- rebuild a component when its module changes.

Hot module replacement uses the same lifecycle system. Cordis identifies affected plugin instances, unloads their effects, invalidates relevant modules, and constructs replacements. If importing the new version fails, it can restore the old module state and rebuild the previous instances.

## 2. Cordis plugins compared with hooks

Hooks and Cordis plugins solve different problems.

| Concern | Hook or callback | Cordis plugin |
|---|---|---|
| Purpose | Extend one event or phase | Own a runtime component and its lifecycle |
| Invocation | Host calls registered functions | Runtime activates a component when dependencies are ready |
| Dependencies | Usually implicit or manually looked up | Declared and continuously monitored |
| Cleanup | Separate unsubscribe or shutdown logic | Disposers are recorded and composed automatically |
| Partial startup failure | Usually handled manually | Completed effects can be rolled back |
| Provider replacement | Hook normally keeps stale references | Bound consumers are unloaded and resolved again |
| Hot reload | Application-specific | Uses the same managed lifecycle |
| Child components | Usually an informal convention | Child plugin creation is itself a tracked effect |
| Safety boundary | None | Lifecycle confinement, but not a security sandbox |

A hook is an extension point. A Cordis plugin is an owned unit of state, dependencies, effects, and recovery. A Cordis plugin may register many hooks, but the plugin lifecycle determines when those hooks exist and guarantees that their registered disposers participate in unload.

For example, an `onMessage` hook only answers "what code runs when a message arrives?" Cordis also answers:

- Who owns this registration?
- Which services must exist before it is installed?
- How is it removed?
- What if installation stops halfway through?
- What if its model or database provider is replaced?

### 2.1 Comparison with familiar plugin and runtime systems

The paper uses VS Code as its detailed motivating comparison and discusses IntelliJ, Eclipse, OSGi, dependency injection, React effects, HMR, and reactive frameworks as adjacent approaches. The comparison below translates those observations into engineering terms.

| System or model | What it does well | What happens during runtime change | Main difference from Cordis |
|---|---|---|---|
| VS Code extensions | Rich, stable host-defined extension points for commands, views, and language features | Executable extensions share an extension host. According to the paper, disabling or uninstalling one generally requires restarting that host. The `deactivate` hook is a graceful shutdown callback, not live per-extension removal | Cordis treats each plugin as an independently unloadable fiber and associates setup with accumulated cleanup |
| IntelliJ plugins | Mature IDE extension ecosystem and explicit plugin lifecycle conventions | The paper groups IntelliJ with systems that delegate recovery to developer-authored unload callbacks | Cordis still requires atomic disposers, but automatically accumulates and runs their composition for the whole plugin |
| Eclipse extension points | Structured host extension points and a large modular ecosystem | The paper groups Eclipse with callback-based cleanup. The extension author remains responsible for complete teardown | Cordis makes effect ownership and recovery part of the runtime component model |
| OSGi Declarative Services | Declared provided and required services with activation tied to service availability | Components can react as services appear and disappear | This is the closest spatial precedent. Cordis adds accumulated effect recovery and an asynchronous, guarded unload protocol |
| iPOJO and Gravity | Provide and require model for autonomous adaptation to changing services | Runtime availability drives component activation and deactivation | Like OSGi DS, recovery relies on a hand-written synchronous deactivation callback. Cordis can await teardown and preserve the departing provider until consumers drain |
| Spring, Guice, Angular DI, Inversify | Strong dependency wiring at initialization, sometimes with scopes or hierarchical injectors | Existing consumers are not normally deactivated and rebuilt when a provider disappears or changes identity | Cordis continuously re-evaluates dependency satisfaction and connects it to lifecycle transitions |
| React `useEffect` | Co-locates an effect and its cleanup | Cleanup runs before re-execution and unmount | It is tied to React's hook rules and UI lifecycle, does not provide a general cross-plugin service graph, and cannot freely compose async effect iterators |
| webpack or Vite HMR | Fast module replacement and optional state handoff | Modules use explicit acceptance and migration APIs to carry state forward | Cordis unloads tracked effects and reapplies the new component from a clean lifecycle boundary. State survives only if moved into a longer-lived service |
| FRP and signals | Fine-grained value propagation and derived-state updates | Values and computations update under a reactive scheduler | Cordis reacts at component granularity and coordinates asynchronous activate and unload operations |
| Processes and containers | Strong isolation and reliable coarse-grained reclamation | Replace the complete process or service instance | Cordis changes a smaller region inside one process and preserves unrelated process-local state, but provides no security sandbox |

The paper gives concrete VS Code ecosystem figures collected on 9 June 2026. Among the 100 most-installed extensions, 87 contained executable code and would therefore require an extension-host restart when removed, while only 7 declared dependencies on non-built-in extensions. The authors use these figures to illustrate both problems: coarse unload boundaries and an ecosystem shaped around host-provided extension points rather than typed plugin-to-plugin composition.

The important point is not that Cordis replaces every system in the table. It combines capabilities that are usually separate:

- IDE extension points define how plugins contribute features.
- OSGi-style services define how components discover one another.
- React-style effects pair setup with cleanup.
- HMR replaces code.
- Process supervisors provide fault boundaries.

Cordis combines service availability, effect ownership, cleanup composition, dependency-aware ordering, and code replacement at one in-process component boundary.

### 2.2 One change viewed through different systems

Suppose a running research agent depends on a memory provider, and that provider must be replaced.

| Approach | Typical engineering response |
|---|---|
| Callback-based IDE plugin | Plugin author manually unregisters everything. The host may require an extension-host or application restart |
| Startup-time DI container | Build a new container or restart the relevant application scope. Existing consumers commonly retain old references |
| OSGi DS or iPOJO | Service withdrawal can deactivate consumers, but teardown remains callback-driven and is described by the paper as synchronous |
| Module HMR | Reload changed modules and use framework-specific acceptance or state migration code |
| Process or container replacement | Restart the whole runtime unit and reconstruct its state |
| Cordis | Withdraw the binding, drain bound consumers, run their accumulated disposers, release the old provider, resolve the new provider, and reactivate only the affected region |

~~~mermaid
flowchart LR
    A["Memory provider replacement requested"] --> B["Cordis stops advertising old provider"]
    B --> C["Bound agent plugins become stale"]
    C --> D["Agents unload tracked effects"]
    D --> E["Old provider waits for consumers to drain"]
    E --> F["Old provider releases resources"]
    F --> G["New provider becomes available"]
    G --> H["Only affected agents reactivate"]
    U["Unrelated plugins"] --> V["Remain active"]
~~~

## 3. How the lifecycle works in practice

### 3.1 Normal activation

Assume an agent plugin requires `model` and `memory` and provides `researchAgent`.

1. The fiber is created but remains inactive.
2. Cordis waits until both required services are available.
3. Cordis records the exact model and memory provider fibers selected.
4. The plugin starts installing effects.
5. Each successful step contributes a disposer.
6. After startup completes, `researchAgent` becomes available to consumers.

The exact provider identities matter. A replacement that returns an equivalent object is still a new provider and may require the consumer to rebuild internal state.

### 3.2 Provider replacement

If the memory provider is replaced:

1. The old memory provider stops accepting new dependents.
2. The agent plugin becomes stale because its committed provider changed.
3. The agent unloads its handlers, jobs, and published service.
4. The old memory provider waits for the agent to finish.
5. The old provider releases its resources.
6. Once the new provider and all other dependencies exist, the agent activates again.

Only the affected dependency region needs to change. Unrelated plugins remain active.

### 3.3 Partial failure and cancellation

Cordis supports multi-step asynchronous activation through effect iterators. A plugin can yield a disposer after each completed step. If step four fails, Cordis can undo steps three, two, and one.

If a dependency changes while one asynchronous step is already running, Cordis lets that step finish, records its cleanup, and then unloads the stale activation. It does not expose the stale plugin as active.

This is valuable for agent components that perform slow initialization such as opening an MCP connection, warming a model client, loading an index, or starting a sandbox.

The same behavior can be summarized as a state machine:

~~~mermaid
stateDiagram-v2
    [*] --> Inactive
    Inactive --> Loading: all dependencies available
    Loading --> Active: activation completes
    Loading --> Unloading: failure or dependency target changes
    Active --> Unloading: retire or provider withdrawal
    Unloading --> Inactive: accumulated cleanup completes
    Inactive --> Loading: a valid new target exists
~~~

## 4. A concrete LLM agent example

Consider a multi-tenant agent host with these components:

- `openaiModel` and `localModel` provide a `model` service in different isolated contexts.
- `vectorMemory` provides `memory`.
- `webTools` provides a tool collection.
- `policyGuard` intercepts tool access.
- `researchAgent` requires `model`, `memory`, and `tools`.
- `evaluationWorker` requires `researchAgent` and `telemetry`.

Cordis can support the following operations:

- Replace `vectorMemory` without restarting model clients or unrelated agents.
- Give one tenant `localModel` and another `openaiModel` under the same logical service name.
- Disable `webTools` and automatically deactivate only agents that require them.
- Update policy metadata around tool access without changing whether the tools dependency is present.
- Hot reload `researchAgent` while retaining unaffected infrastructure.
- Roll back partially installed routes, event subscriptions, and workers if the new agent version fails to start.

The resulting architecture is:

~~~text
desired configuration
        |
        v
Cordis loader and reconciliation
        |
        v
plugin fibers and dependency bindings
        |
        +---- tracked effects and disposers
        +---- provided and required services
        +---- isolation and interception
        +---- reload, unload, and failure recovery
~~~

Cordis is the lifecycle control plane. It does not replace the model gateway, tool protocol, memory database, message bus, or security sandbox.

## 5. Potential impact on the LLM agent industry

### 5.1 From static agents to live agent runtimes

Most agent frameworks are optimized for defining a graph or workflow and then executing it. Cordis points toward agent hosts that can change their component graph while continuing to serve work. This could support long-lived assistants, desktop agents, IDE agents, and multi-tenant automation platforms where full restarts are disruptive.

### 5.2 Safer self-modification

The paper discusses self-evolving agent harnesses as a motivating use case. A model-generated modification could be installed as a new component, observed, and removed if activation fails.

The practical benefit is containment of lifecycle damage, not proof that generated behavior is correct. The generated component must still:

- declare every dependency;
- route every managed effect through the context;
- provide correct and idempotent cleanup;
- avoid or compensate irreversible external actions;
- run behind a real sandbox if it is untrusted.

Cordis could therefore become one layer in a self-modification pipeline:

~~~text
generate -> validate -> sandbox -> activate -> observe -> retain or unload
~~~

It primarily strengthens the activate and unload stages.

### 5.3 Fine-grained fault isolation

When one plugin fails during activation, Cordis can recover that plugin's completed effects and record a local failure without necessarily stopping sibling components. For agent platforms, this can reduce the blast radius of a broken tool adapter, evaluator, or provider integration.

This is lifecycle isolation, not memory or security isolation. A plugin can still crash the process, consume excessive resources, mutate global state, or bypass context APIs unless stronger runtime boundaries are used.

### 5.4 Stateful upgrades with smaller disruption

Agent infrastructure often holds expensive process-local state: connection pools, caches, loaded indexes, model sessions, browser sessions, and task queues. Selective reconciliation can preserve unaffected state while replacing only the relevant component and its dependents.

This may improve developer iteration and operational availability, but the paper does not provide benchmarks for reload latency, memory overhead, or throughput.

### 5.5 A common contract for agent ecosystem components

An ecosystem becomes easier to compose when components share a contract for:

- dependencies;
- service publication;
- resource ownership;
- cleanup;
- configuration changes;
- failure state.

Cordis offers such a contract. If adopted by tool and agent framework authors, it could reduce framework-specific glue and make extensions more portable conceptually. Actual portability still depends on shared service interfaces, versioning, and language bindings.

### 5.6 Better infrastructure for evaluation and rollout

Because plugin identity and lifecycle are explicit, an agent platform could build operational features around them:

- canary replacement of a planner or model router;
- tenant-scoped service selection;
- automatic rollback after health-check failure;
- tracing grouped by plugin fiber;
- measurement of activation, drain, and cleanup latency;
- policy interception around tool or model access.

Some of these are extensions suggested by the architecture rather than completed features demonstrated in the paper.

## 6. What Cordis does not solve

### 6.1 Irreversible external actions

Sending an email, publishing a message, executing a trade, or charging a customer is an emission outside the local runtime. A disposer cannot erase the fact that it happened. Such operations need delayed commit, idempotency keys, transactional outboxes, or domain-specific compensation.

### 6.2 Incorrect cleanup

Cordis can call disposers in the correct order, but it cannot prove that a disposer is complete or semantically correct. Plugin tests should verify repeated unload, partial startup failure, and provider replacement.

### 6.3 Global and ambient state

Direct mutation of global objects, hidden singletons, unmanaged threads, and third-party libraries can escape ownership tracking. Strong results apply only to effects mediated by the Cordis context.

### 6.4 Security isolation

Dependency access checks and interception resemble capability controls, but same-process JavaScript is not a sandbox. Untrusted or model-generated code needs a worker, process, virtual machine, or container boundary with a narrow bridge.

### 6.5 Cyclic dependencies

A cycle can leave every participant waiting forever. Cordis assumes an acyclic provider dependency graph. Teams may need to extract a shared core service or introduce a broker to remove cycles.

### 6.6 Proven production economics

The Koishi ecosystem demonstrates substantial real-world use of the underlying composition model, with more than 4,000 community plugins. However, the paper notes that Koishi uses Cordis v3 while the paper describes v4. It does not provide controlled comparisons or quantitative results for throughput, memory, recovery time, or developer productivity.

## 7. Engineering adoption guide

### Good fits

Cordis is a strong candidate when:

- one process hosts many independently managed extensions;
- plugins have nontrivial resources and dependencies;
- live configuration and hot reload matter;
- preserving unaffected process-local state matters;
- failures should remain localized;
- the team can enforce context-mediated resource ownership.

Examples include an agent development server, an IDE agent host, a multi-tenant bot platform, or a tool gateway with dynamically installed adapters.

### Poor fits

Cordis adds less value when:

- the application is small and restarted safely in seconds;
- components already run in separate processes or containers;
- most actions are irreversible external emissions;
- plugin authors cannot follow resource ownership rules;
- dependencies are highly cyclic or discovered only through global state.

### Minimum production checklist

1. Wrap every listener, timer, route, worker, provider, and child plugin in a tracked effect.
2. Make cleanup idempotent and safe after partial initialization.
3. Declare every service dependency and avoid ambient lookup.
4. Test provider replacement during asynchronous activation.
5. Separate reversible local state from irreversible external actions.
6. Use durable workflow or transaction patterns for emissions.
7. Put untrusted and model-generated code behind a genuine sandbox.
8. Detect dependency cycles before activation.
9. Instrument activation time, unload time, dependent drain time, and disposer failures.
10. Load-test reconciliation and hot reload for the expected plugin count.

## 8. Evaluation of the paper

The paper's strongest contribution is a coherent engineering model backed by formal reasoning. The formal sections justify several useful expectations:

- completed plugin effects can be removed without deleting unrelated effects, if they are independent;
- consumers activate after providers and unload before providers disappear;
- a plugin activation uses one stable dependency resolution;
- under explicit assumptions, different reconfiguration schedules reach an equivalent final state.

For practitioners, these are design constraints rather than automatic guarantees. They depend on correct cleanup, independent effects, finite and acyclic dependencies, and disciplined use of the context. Failures and irreversible emissions are outside the strongest convergence result.

The paper is convincing as an architectural foundation and as an explanation of Cordis. It is not yet a complete business or performance case for replacing established agent runtimes. The next useful evidence would be:

- benchmarks against restart-based and callback-based plugin systems;
- failure-injection tests;
- property-based lifecycle testing;
- a production evaluation of Cordis v4;
- measurements on agent workloads with many tools and providers;
- security analysis for generated plugins.

## Final summary

Cordis turns a plugin from "some code called by the host" into a managed runtime object with declared dependencies, owned effects, recorded cleanup, and a controlled lifecycle.

For normal software, this enables targeted unload, dependency-aware restart, and safer hot reload inside one process. For LLM agent platforms, it offers a foundation for dynamically replacing tools, providers, policies, memory systems, and generated components without restarting the entire agent host.

Its likely industry impact is not a new prompting method or agent reasoning algorithm. It is infrastructure: a more disciplined way to operate mutable, long-lived agent systems. Cordis can reduce stale registrations, leaked resources, invalid provider references, and system-wide restarts. It cannot validate agent behavior, undo external actions, or safely contain hostile code by itself.

The most productive way to adopt it is as one layer in a broader agent platform:

~~~text
security sandbox + durable workflow + Cordis lifecycle + observability
~~~

Within that boundary, Cordis provides a compelling model for plugin-based agent runtimes that need to evolve while they are running.
