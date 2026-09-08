#### How the dependency graph is built

Terraform builds a directed acyclic graph before performing any operation. The graph is built in a specific sequence:

Resource nodes are added from the configuration with any diff or state metadata attached. Resources are then mapped to provisioners if any are defined. Explicit `depends_on` edges are added between resources. Orphan resources from state are added next. Orphan resources exist in state but not in configuration and have no config attached since state does not store configuration. Resources are then mapped to providers and provider configuration nodes are created with edges from each resource to its provider. Interpolations are parsed and reference expressions are turned into dependency edges. A root node is created pointing to all resources so the graph has a single entry point. Resources being destroyed and recreated are split into two nodes, one for destroy and one for create, because destroy order is often different from create order. Finally the graph is validated to confirm it has no cycles and a single root.

---

#### The three graph node types

A **Resource Node** represents a single resource instance. If `count = 4` there are four separate resource nodes, one per instance.

A **Provider Configuration Node** represents the moment a provider gets fully configured with its credentials and settings. Every resource node has an edge to its provider configuration node because the provider must be configured before any resource using it can run.

A **Resource Meta-Node** groups all instances of a resource with `count > 1` into a single entity in the graph. It represents no action itself. It exists purely for convenience when expressing dependencies and for cleaner graph visualization.

---

#### Walking the graph

Terraform walks the graph using depth-first traversal. A node is processed as soon as all its dependencies have been processed. Independent nodes are processed in parallel.

The default parallelism is 10 concurrent operations. You can override it with `-parallelism=n` on plan, apply, and destroy. This is considered an advanced operation and is not necessary for normal use. Some providers handle API rate limiting internally through graceful backoff in their API clients, so adjusting parallelism is not the right tool for rate limit issues.

---

#### Implicit vs explicit dependencies

An implicit dependency is created automatically from any reference expression. When resource A references an attribute of resource B, Terraform adds an edge from A to B in the graph without any extra declaration needed.

An explicit dependency is declared with `depends_on` when the dependency exists but no reference expresses it. Use it sparingly. Frequently needing `depends_on` usually signals that coupling between resources should be made explicit through direct references instead.


```hcl
resource "aws_s3_bucket_policy" "example" {
  bucket = aws_s3_bucket.main.id
  policy = "..."

  depends_on = [aws_s3_bucket_public_access_block.main]
}
```

`depends_on` is supported on resource blocks, data blocks, and module blocks.

---

#### Visualizing the graph

bash

```bash
terraform graph
```

This outputs the dependency graph in DOT format. You can render it with Graphviz or paste it into an online DOT visualizer. It is useful for debugging unexpected dependency ordering or understanding how a complex configuration fits together.