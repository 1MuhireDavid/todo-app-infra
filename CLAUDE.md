# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Authorship

Never add AI attribution anywhere. No `Co-Authored-By: Claude`, no "Generated
with Claude Code", no "written by Claude", in commit messages, PR titles or
bodies, file headers, or code comments. Commits and pull requests are authored by
the repository owner alone.

## Comments

The default is no comment. Add one only when a reader who knows CloudFormation
would still ask "why".

- **Explain why, never what.** If a comment restates the line below it, delete it.
- **Never interrupt a property block.** No comments between YAML keys, inside a
  `Properties:` map, or between the entries of a list. If a choice needs
  explaining, put one short comment above the whole resource.
- **One comment per resource, at most, and two lines at most.** Anything longer
  belongs in the README or in `docs/`, not in the template.
- **No section-divider banners.** No `# ----- Network -----` blocks.
- **No comments on parameter declarations.** Parameters already have a
  `Description` field; use it and leave the declaration clean.
- **No comments restating a policy.** An IAM statement with a `Sid` is already
  labelled.

Bad:

```yaml
  Service:
    Type: AWS::ECS::Service
    Properties:
      DeploymentController:
        # CodeDeploy owns rollout from here on, so stack updates do not fight it
        Type: CODE_DEPLOY
      NetworkConfiguration:
        AwsvpcConfiguration:
          # no public IP - everything the task needs is behind a VPC endpoint
          AssignPublicIp: DISABLED
```

Good:

```yaml
  # CodeDeploy owns task definition rollout, so pipeline deploys and stack
  # updates cannot fight over this service.
  Service:
    Type: AWS::ECS::Service
    Properties:
      DeploymentController:
        Type: CODE_DEPLOY
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: DISABLED
```

### The two exceptions

These are graded, so they stay — one line each, above the resource or on the line
itself:

- Every `Resource: "*"` in an IAM policy states why the API cannot be scoped.
- Every `DependsOn` states why implicit ordering is not enough.

## YAML style

Block style everywhere, including intrinsics. No inline `{}` or `[]` flow style.
Embedded JSON uses a `|` block literal. Quote anything YAML would coerce:
`"256"`, `"false"`, `"2010-09-09"`.

## Template hashes

`cfn/deployment-file.yaml` holds a content hash per nested template, maintained
by `.github/workflows/package-templates.yml`. Editing a template changes its
hash. Do not hand-edit the hash values; let the workflow write them, or
regenerate all seven with `git hash-object <file> | cut -c1-12`.

## Before proposing a change as done

`cfn-lint cfn/root.yaml cfn/nested-templates/*.yaml` must pass with no errors
and no warnings.

## Scope

This repository holds the application stack only. The templates bucket, ECR
repository, artifact bucket and the two OIDC roles live in `todo-app-bootstrap`
and arrive here as `Fn::ImportValue`. Do not recreate them; do not add resources
here that need to survive this stack's teardown.
