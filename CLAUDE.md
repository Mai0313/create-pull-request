# AI Implementation Instructions: Add Gitea Support to create-pull-request Action

## Objective
Extend the GitHub-only `create-pull-request` action to support GitHub Enterprise Server (GHES) and Gitea instances while maintaining full backward compatibility with existing GitHub functionality.

## Background
This action creates pull requests using the GitHub API. Gitea is a lightweight, self-hosted Git service with a GitHub-compatible API, but with several key differences that must be handled.

## Core Strategy

### 1. Platform Detection Mechanism

**File: `src/octokit-client.ts`**

Add two utility functions to detect Gitea instances and determine the correct API base URL:

```typescript
// Determine if a hostname is a Gitea instance
export function isGitea(hostname: string): boolean {
  return process.env.GITEA_INSTANCES
    ? process.env.GITEA_INSTANCES.split(',').includes(hostname)
    : false
}

// Get the API base URL for a given hostname
export function getApiBaseUrl(hostname: string): string {
  // For GitHub, we'll use their standard API endpoint
  if (hostname === 'github.com') {
    return 'https://api.github.com'
  }

  // For Gitea, we need to modify the API path
  if (isGitea(hostname)) {
    return `https://${hostname}/api/v1`
  }

  // For GitHub Enterprise or other GitHub-compatible APIs
  return `https://${hostname}/api/v3`
}
```

**Key Points:**
- Use the `GITEA_INSTANCES` environment variable (comma-separated hostnames) to identify Gitea instances
- Gitea uses `/api/v1` endpoint, GitHub uses `/api/v3`, GHES uses `/api/v3`
- Export these functions for use in `github-helper.ts`

### 2. Environment Configuration

**File: `src/main.ts`**

Add a function to configure Gitea instances from the action input:

```typescript
// Set Gitea instances from environment variable or input
function configureGiteaInstances() {
  // First check if there's already an environment variable
  if (!process.env.GITEA_INSTANCES) {
    // If not, check if it was provided as input
    const giteaInstancesInput = core.getInput('github-server-url')
    if (giteaInstancesInput) {
      core.info(
        `Setting GITEA_INSTANCES environment variable to: ${giteaInstancesInput}`
      )
      process.env.GITEA_INSTANCES = giteaInstancesInput
    }
  }

  if (process.env.GITEA_INSTANCES) {
    core.info(`Configured Gitea instances: ${process.env.GITEA_INSTANCES}`)
  }
}

async function run(): Promise<void> {
  try {
    // Configure Gitea instances before anything else
    configureGiteaInstances()

    // ... rest of the run function
  } catch (error) {
    core.setFailed(utils.getErrorMessage(error))
  }
}
```

**Key Points:**
- Call `configureGiteaInstances()` at the very beginning of `run()`
- This sets up the environment variable before any GitHubHelper instantiation

### 3. Action Input Definition

**File: `action.yml`**

Add a new input parameter:

```yaml
github-server-url:
  description: >
    A comma-separated list of Gitea hostnames (e.g., 'gitea.example.com,gitea.company.org').
    Required when using this action with Gitea instances.
```

Add this to the inputs section after `maintainer-can-modify`.

### 4. GitHubHelper Constructor Modification

**File: `src/github-helper.ts`**

Add an instance variable and modify the constructor:

```typescript
export class GitHubHelper {
  private octokit: InstanceType<typeof Octokit>
  private isGiteaInstance: boolean  // Add this instance variable

  constructor(githubServerHostname: string, token: string) {
    const options: OctokitOptions = {}
    if (token) {
      options.auth = `${token}`
    }

    // Check if this is a Gitea instance
    this.isGiteaInstance = isGitea(githubServerHostname)

    // Set the appropriate API base URL for GitHub or Gitea
    options.baseUrl = getApiBaseUrl(githubServerHostname)

    if (this.isGiteaInstance) {
      core.info(
        `Detected Gitea instance at ${githubServerHostname}. Using API endpoint ${options.baseUrl}`
      )
    }

    options.throttle = throttleOptions
    this.octokit = new Octokit(options)
  }

  // ... rest of the class
}
```

**Key Points:**
- Import `isGitea` and `getApiBaseUrl` from `octokit-client.ts`
- Store `isGiteaInstance` as a class property for use throughout the class
- Log when a Gitea instance is detected

### 5. Pull Request Creation Logic

**File: `src/github-helper.ts` - `createOrUpdate()` method**

Modify the PR creation logic to handle platform differences:

#### 5.1 Add Gitea-specific error constant at the top of the file:

```typescript
const ERROR_PR_ALREADY_EXISTS = 'A pull request already exists for'
const ERROR_GITEA_PR_ALREADY_EXISTS = 'pull request already exists for these targets'  // Add this
const ERROR_PR_REVIEW_TOKEN_SCOPE = 'Validation Failed: "Could not resolve to a node with the global id of'
```

#### 5.2 Modify the `createOrUpdate()` method:

```typescript
private async createOrUpdate(
  inputs: Inputs,
  baseRepository: string,
  headRepository: string
): Promise<Pull> {
  const [headOwner] = headRepository.split('/')
  const headBranch = `${headOwner}:${inputs.branch}`
  // For Gitea, the head branch format is different - it's just the branch name
  const giteaHeadBranch = this.isGiteaInstance ? inputs.branch : headBranch

  // Try to create the pull request
  try {
    core.info(`Attempting creation of pull request`)
    const createParams = {
      ...this.parseRepository(baseRepository),
      title: inputs.title,
      head: this.isGiteaInstance ? giteaHeadBranch : headBranch,
      base: inputs.base,
      body: inputs.body,
      maintainer_can_modify: inputs.maintainerCanModify
    }

    // Add draft parameter only for GitHub (Gitea doesn't support draft PRs via the API)
    if (!this.isGiteaInstance) {
      Object.assign(createParams, {draft: inputs.draft.value})
    }

    // For Gitea, if using fork, we need to specify the head_repo
    if (this.isGiteaInstance && inputs.pushToFork) {
      Object.assign(createParams, {head_repo: headRepository})
    }

    const {data: pull} = await this.octokit.rest.pulls.create(createParams)
    core.info(
      `Created pull request #${pull.number} (${headBranch} => ${inputs.base})`
    )
    return {
      number: pull.number,
      html_url: pull.html_url,
      node_id: pull.node_id,
      draft: pull.draft,
      created: true
    }
  } catch (e) {
    const errorMessage = utils.getErrorMessage(e)
    if (errorMessage.includes(ERROR_PR_ALREADY_EXISTS) ||
        (this.isGiteaInstance && errorMessage.includes(ERROR_GITEA_PR_ALREADY_EXISTS))) {
      core.info(`A pull request already exists for ${headBranch}`)
    } else if (errorMessage.includes(ERROR_PR_FORK_COLLAB)) {
      core.warning(
        'An attempt was made to create a pull request using a token that does not have write access to the head branch.'
      )
      core.warning(
        `For this case, set input 'maintainer-can-modify' to 'false' to allow pull request creation.`
      )
      throw e
    } else {
      throw e
    }
  }

  // Update the pull request that exists for this branch and base
  core.info(`Fetching existing pull request`)
  const {data: pulls} = await this.octokit.rest.pulls.list({
    ...this.parseRepository(baseRepository),
    state: 'open',
    head: this.isGiteaInstance ? giteaHeadBranch : headBranch,
    base: inputs.base
  })

  if (!pulls || pulls.length === 0) {
    core.warning('Expected to find an existing pull request, but none was found.')
    throw new Error('Failed to locate existing pull request for update.')
  }

  core.info(`Attempting update of pull request`)
  const {data: pull} = await this.octokit.rest.pulls.update({
    ...this.parseRepository(baseRepository),
    pull_number: pulls[0].number,
    title: inputs.title,
    body: inputs.body
  })
  core.info(
    `Updated pull request #${pull.number} (${headBranch} => ${inputs.base})`
  )
  return {
    number: pull.number,
    html_url: pull.html_url,
    node_id: pull.node_id,
    draft: pull.draft,
    created: false
  }
}
```

**Key Points:**
- Gitea uses simple branch names (e.g., `feature-branch`) while GitHub uses `owner:branch` format
- Gitea doesn't support the `draft` parameter in pull request creation
- Gitea requires `head_repo` parameter when creating PRs from forks
- Gitea has a different error message for existing pull requests

### 6. Assignee Handling

**File: `src/github-helper.ts` - `createOrUpdatePullRequest()` method**

Modify the assignee assignment logic:

```typescript
// Apply assignees
if (inputs.assignees.length > 0) {
  core.info(`Applying assignees '${inputs.assignees}'`)
  // Gitea has different assignee handling
  if (this.isGiteaInstance) {
    try {
      for (const assignee of inputs.assignees) {
        await this.octokit.request(
          'POST /repos/{owner}/{repo}/issues/{issue_number}/assignees',
          {
            ...this.parseRepository(baseRepository),
            issue_number: pull.number,
            assignees: [assignee]
          }
        )
      }
    } catch (error) {
      core.warning(
        `Error assigning users in Gitea: ${utils.getErrorMessage(error)}`
      )
    }
  } else {
    // GitHub standard API
    await this.octokit.rest.issues.addAssignees({
      ...this.parseRepository(baseRepository),
      issue_number: pull.number,
      assignees: inputs.assignees
    })
  }
}
```

**Key Points:**
- Gitea may require assignees to be added one at a time
- Wrap in try-catch to handle potential Gitea API differences gracefully
- Use `this.octokit.request()` for Gitea to have more control over the request format

### 7. Reviewer Assignment

**File: `src/github-helper.ts` - `createOrUpdatePullRequest()` method**

Modify the reviewer request logic to skip for Gitea:

```typescript
// Skip reviewers functionality for Gitea as it might not be compatible
if (
  !this.isGiteaInstance &&
  (inputs.reviewers.length > 0 || inputs.teamReviewers.length > 0)
) {
  const requestReviewersParams = {}
  if (inputs.reviewers.length > 0) {
    requestReviewersParams['reviewers'] = inputs.reviewers
    core.info(`Requesting reviewers '${inputs.reviewers}'`)
  }
  if (inputs.teamReviewers.length > 0) {
    const teams = utils.stripOrgPrefixFromTeams(inputs.teamReviewers)
    requestReviewersParams['team_reviewers'] = teams
    core.info(`Requesting team reviewers '${teams}'`)
  }
  if (Object.keys(requestReviewersParams).length > 0) {
    try {
      await this.octokit.rest.pulls.requestReviewers({
        ...this.parseRepository(baseRepository),
        pull_number: pull.number,
        ...requestReviewersParams
      })
    } catch (e) {
      if (utils.getErrorMessage(e).includes(ERROR_PR_REVIEW_TOKEN_SCOPE)) {
        core.error(
          `Unable to request reviewers. If requesting team reviewers a 'repo' scoped PAT is required.`
        )
      }
      throw e
    }
  }
} else if (
  this.isGiteaInstance &&
  (inputs.reviewers.length > 0 || inputs.teamReviewers.length > 0)
) {
  core.warning('Reviewer assignment is not supported for Gitea instances')
}
```

**Key Points:**
- Only execute reviewer logic for GitHub instances
- Provide a warning if reviewers are requested on Gitea
- This prevents errors from unsupported API calls

### 8. Signed Commits Handling

**File: `src/github-helper.ts` - `pushSignedCommits()` method**

Add a Gitea-specific fallback at the beginning of the method:

```typescript
async pushSignedCommits(
  git: GitCommandManager,
  branchCommits: Commit[],
  baseCommit: Commit,
  repoPath: string,
  branchRepository: string,
  branch: string
): Promise<CommitResponse> {
  // For Gitea, fall back to standard Git push if signed commits are not supported
  if (this.isGiteaInstance) {
    core.warning(
      'Signed commits via API may not be fully supported in Gitea. Falling back to standard Git push.'
    )
    await git.push([
      '--force-with-lease',
      'origin',
      `${branch}:refs/heads/${branch}`
    ])

    // Return a simplified commit response
    return {
      sha: branchCommits[branchCommits.length - 1]?.sha || baseCommit.sha,
      tree: branchCommits[branchCommits.length - 1]?.tree || baseCommit.tree,
      verified: false
    }
  }

  // Original GitHub implementation continues here...
  let headCommit: CommitResponse = {
    sha: baseCommit.sha,
    tree: baseCommit.tree,
    verified: false
  }
  // ... rest of the method
}
```

**Key Points:**
- Gitea's API may not fully support the signed commit creation workflow
- Fall back to standard Git push for Gitea
- Return a valid CommitResponse with `verified: false`

### 9. Commit Verification Structure

**File: `src/github-helper.ts` - `createCommit()` method**

Handle optional verification object:

```typescript
const {data: remoteCommit} = await this.octokit.rest.git.createCommit({
  ...repository,
  parents: [parentCommit.sha],
  tree: treeSha,
  message: `${commit.subject}\n\n${commit.body}`
})
core.info(
  `Created commit ${remoteCommit.sha} for local commit ${commit.sha}`
)

// Gitea might not have the same verification structure
let verified = false
if (
  remoteCommit.verification &&
  typeof remoteCommit.verification.verified !== 'undefined'
) {
  verified = remoteCommit.verification.verified
  core.info(
    `Commit verified: ${verified}; reason: ${remoteCommit.verification.reason || 'unknown'}`
  )
} else {
  core.info('Commit verification information not available')
}

return {
  sha: remoteCommit.sha,
  tree: remoteCommit.tree.sha,
  verified: verified
}
```

**Key Points:**
- Check if `remoteCommit.verification` exists before accessing it
- Default to `verified: false` if verification info is unavailable
- Provide informative logging for both cases

### 10. Repository Parent Handling

**File: `src/github-helper.ts` - `getRepositoryParent()` method**

Add error handling for Gitea:

```typescript
async getRepositoryParent(headRepository: string): Promise<string | null> {
  try {
    const {data: headRepo} = await this.octokit.rest.repos.get({
      ...this.parseRepository(headRepository)
    })
    if (!headRepo.parent) {
      return null
    }
    return headRepo.parent.full_name
  } catch (error) {
    // Gitea may not have the same parent repository structure
    // Fall back to null if this fails
    if (this.isGiteaInstance) {
      core.warning(
        `Unable to determine parent repository for ${headRepository}. This is expected for Gitea.`
      )
      return null
    }
    throw error
  }
}
```

**Key Points:**
- Gitea's fork structure may differ from GitHub
- Return `null` for Gitea instead of throwing an error
- Log a warning to inform users this is expected behavior

### 11. Get Commit Method Enhancement

**File: `src/github-helper.ts` - `getCommit()` method**

Add similar verification handling:

```typescript
async getCommit(
  sha: string,
  branchRepository: string
): Promise<CommitResponse> {
  const repository = this.parseRepository(branchRepository)
  try {
    const {data: remoteCommit} = await this.octokit.rest.git.getCommit({
      ...repository,
      commit_sha: sha
    })

    // Handle different verification structure between GitHub and Gitea
    let verified = false
    if (
      remoteCommit.verification &&
      typeof remoteCommit.verification.verified !== 'undefined'
    ) {
      verified = remoteCommit.verification.verified
    }

    return {
      sha: remoteCommit.sha,
      tree: remoteCommit.tree.sha,
      verified: verified
    }
  } catch (error) {
    if (this.isGiteaInstance) {
      core.warning(
        `Unable to get commit details from Gitea. This might be expected: ${utils.getErrorMessage(error)}`
      )
      // Return a placeholder response
      return {
        sha: sha,
        tree: '', // We don't know the tree SHA
        verified: false
      }
    }
    throw error
  }
}
```

**Key Points:**
- Gitea may not support all commit retrieval operations
- Return a placeholder response for Gitea if the API call fails
- Preserve error throwing for GitHub to catch real issues

### 12. Draft Pull Request Conversion

**File: `src/github-helper.ts` - `convertToDraft()` method**

Skip GraphQL operations for Gitea:

```typescript
async convertToDraft(id: string): Promise<void> {
  // Skip for Gitea since GraphQL API likely isn't compatible
  if (this.isGiteaInstance) {
    core.warning(
      'Draft pull requests are not supported in Gitea via the GraphQL API'
    )
    return
  }

  core.info(`Converting pull request to draft`)
  await this.octokit.graphql({
    query: `mutation($pullRequestId: ID!) {
      convertPullRequestToDraft(input: {pullRequestId: $pullRequestId}) {
        pullRequest {
          isDraft
        }
      }
    }`,
    pullRequestId: id
  })
}
```

**Key Points:**
- Gitea likely doesn't support GraphQL
- Return early with a warning instead of attempting the GraphQL mutation
- This prevents crashes when draft conversion is requested on Gitea

## Implementation Checklist

- [ ] Add `isGitea()` and `getApiBaseUrl()` functions to `src/octokit-client.ts`
- [ ] Add `configureGiteaInstances()` function to `src/main.ts` and call it at the start of `run()`
- [ ] Add `github-server-url` input to `action.yml`
- [ ] Add `isGiteaInstance` property to `GitHubHelper` class
- [ ] Modify `GitHubHelper` constructor to detect Gitea and set API base URL
- [ ] Add `ERROR_GITEA_PR_ALREADY_EXISTS` constant
- [ ] Modify `createOrUpdate()` method to handle head branch format differences
- [ ] Modify `createOrUpdate()` method to conditionally include draft parameter
- [ ] Modify `createOrUpdate()` method to add head_repo for Gitea forks
- [ ] Update error handling in `createOrUpdate()` to recognize Gitea error messages
- [ ] Update `createOrUpdate()` method to use correct head format when listing/updating PRs
- [ ] Modify assignee handling in `createOrUpdatePullRequest()` to loop for Gitea
- [ ] Add conditional logic to skip reviewer assignment for Gitea
- [ ] Add Gitea fallback to `pushSignedCommits()` method
- [ ] Update commit verification handling in `createCommit()` method
- [ ] Update commit verification handling in `getCommit()` method
- [ ] Add error handling to `getRepositoryParent()` for Gitea
- [ ] Add early return to `convertToDraft()` for Gitea
- [ ] Update documentation in `README.md` with Gitea usage examples:
  - [ ] Add `github-server-url` input to the action inputs table
  - [ ] Add "Gitea support" section with usage example and limitations
- [ ] Build the TypeScript code to update `dist/` files: `npm run build`

## Testing Recommendations

1. **GitHub Functionality**: Verify all existing GitHub functionality still works
2. **Gitea Basic PR**: Test creating a simple PR on Gitea
3. **Gitea Fork PR**: Test creating a PR from a fork on Gitea
4. **Gitea with Assignees**: Test assigning users on Gitea
5. **Gitea with Reviewers**: Verify warning appears and doesn't crash
6. **Mixed Workflow**: Test a workflow that could run on both GitHub and Gitea

## Design Principles

1. **Backward Compatibility**: All changes must not break existing GitHub functionality
2. **Conditional Execution**: Use `isGiteaInstance` flag to branch logic
3. **Graceful Degradation**: Unsupported Gitea features should warn, not fail
4. **Error Tolerance**: Catch Gitea-specific errors and provide helpful messages
5. **API Isolation**: Handle API URL differences at the lowest level (octokit-client)
6. **Clear Logging**: Log when Gitea is detected and when Gitea-specific code paths execute

## Usage Example

After implementation, users should be able to use the action with Gitea like this:

```yaml
- name: Create Pull Request
  uses: your-org/create-pull-request@main
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    github-server-url: gitea.example.com
    base: master
    branch: feature-branch
    title: "Feature: Add new functionality"
    body: "This PR adds new functionality"
    commit-message: "feat: add new functionality"
```

For GitHub, the usage remains unchanged (no need to specify `github-server-url`).
