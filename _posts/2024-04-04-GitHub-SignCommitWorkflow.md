---
layout: post
title: "GitHub: signing commit in a workflow"
date: 20240404
published: true
categories: [GitHub]
tags: Security, AppSec, GitHub
canonical_url: https://0xdbe.github.io/GitHub-SignCommitWorkflow/
---

Committing in your workflow can normally be done using git commands or other actions that perform commits for you.
However, if your repository requires commit signing, it is difficult to manage securely a GPG keys and set up GitHub Runner to sign your commit.
Fortunately, this can be done through the GitHub GraphQL API.

## GraphQL Mutation

Github provides a GraphQL Mutation [createcommitonbranch](https://docs.github.com/en/graphql/reference/mutations#createcommitonbranch) which creates a commit.
Commits made using this mutation are automatically signed by GitHub and will be marked as verified in the user interface.

For better reusability, we can define a GraphQL mutation in a file ``.github/api/createCommitOnBranch.gql``:

```graphql
mutation (
    $githubRepository: String!,
    $branchName: String!,
    $expectedHeadOid: GitObjectID!
    $commitMessage: String!
    $files: [FileAddition!]!
) {
  createCommitOnBranch(
    input: {
    branch:
    {
        repositoryNameWithOwner: $githubRepository,
        branchName: $branchName
    },
    message: {headline: $commitMessage},
    fileChanges: {
        additions: $files
    }
    expectedHeadOid: $expectedHeadOid
    }
  ){
    commit {
    url
    }
  }
}
```

The following input fields must be set when you call this mutation:

- ``githubRepository``
- ``branchName``
- ``expectedHeadOid``
- ``commitMessage``
- ``files``

## Usage with gh CLI

The mutation can now be called using ``gh`` CLI:

```bash
gh api graphql \
  -F $githubRepository="owner/repo" \
  -F branchName="main" \
  -F expectedHeadOid=$(git rev-parse HEAD) \
  -F commitMessage="chore: commit file" \
  -F files[][path]="README.md" -F files[][contents]=$(base64 -w0 README.md) \
  -F files[][path]="HELLO.md" -F files[][contents]=$(base64 -w0 HELLO.md) \
  -F 'query=@.github/api/createCommitOnBranch.gql'
```

Warning: ``files`` is an array of ``FileAddition``, but fortunately, the ``gh`` CLI allows defining nested objects in fields.


## Usage in a GitHub Workflow

In order to sign a new commit in your workflow, you can add a step to call the mutation using the ``gh`` CLI:

```yaml
permissions:
  checks: write
  statuses: write
  contents: write

jobs:

  committing:
    runs-on: ubuntu-latest
    steps:

    - name: checkout
      uses: actions/checkout@v4

    - name: edit file
      run: |
        echo "hello" >> README.md

    - name: commit
      run: |
        gh api graphql \
          -F $githubRepository=$GITHUB_REPOSITORY \
          -F branchName=$BRANCH \
          -F expectedHeadOid=$(git rev-parse HEAD) \
          -F commitMessage="chore: commit file" \
          -F files[][path]="README.md" -F files[][contents]=$(base64 -w0 README.md) \
          -F files[][path]="HELLO.md" -F files[][contents]=$(base64 -w0 HELLO.md) \
          -F 'query=@.github/api/createCommitOnBranch.gql'
      env:
        GH_TOKEN: ${{ github.token }}
        BRANCH: "main"
```

This commit will be signed by ``github-actions[bot]``.
