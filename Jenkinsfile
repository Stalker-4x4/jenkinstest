// Send a query to github and get the response JSON
def githubQuery(Map args = [:]) {
  def formattedQuery = args.query.replaceAll('\n', ' ').replaceAll('"', '\\\\"')
  def response = httpRequest(
    authentication: args.auth,
    httpMode: 'POST',
    requestBody: """{ "query": "${formattedQuery}" }""",
    url: "https://api.github.com/graphql"
  )
  (response.status == 200) ? readJSON(text: response.content) : [:]
}

def getPRStatus(Map args = [:]) {
  // Build the query to get all open pull requests and their status
  def query = """query {
    organization(login: "${args.organization}") {
      repositories(first: 30) {
        nodes {
          name
          pullRequests(first: 100, states: [OPEN]) {
            nodes {
              number
              state
              reviews(first: 10, states: [APPROVED]) {
                totalCount
              }
            }
          }
        }
      }
    }
  }"""
  def response = githubQuery(args + [query: query])
  def repositories = response?.data.organization.repositories.nodes
  // Organize the pull requests into approved and unapproved groups
  repositories?.collectEntries { repo ->
    // Take out draft pull requests
    def prs = repo.pullRequests.nodes.findAll { it.state != "DRAFT" }
    def repoPrs = [
      unapproved: prs.findAll { it.reviews.totalCount == 0 },
      approved: prs.findAll { it.reviews.totalCount > 0 }
    ].collectEntries { category, categoryPrs ->
      [ category, categoryPrs.collect { it.number } ]
    }
    [ repo.name, repoPrs ]
  }
}

def monitorRecentlyApprovedPRs(Map args = [:]) {
  def prMap = getPRStatus(args)
  // Build recently approved pull requests on each repository
  prMap.each { repoName, repoPrs ->
    // Get previously unapproved pull requests
    def previouslyUnapproved = currentBuild.previousBuild?.buildVariables?."${repoName}"?.tokenize(",").collect { it.toInteger() } ?: []
    // Build recently approved pull requests
    repoPrs.approved.intersect(previouslyUnapproved).each { prNumber ->
      build job: "/${args.multibranch}/PR-${prNumber}", wait: false
    }
    env."${repoName}" = repoPrs.unapproved.join(",")
  }
}


def shouldBuildPR(Map args = [:]) {
  // Get pull request info
  def query = """query {
    organization(login: "${args.organization}") {
      repository(name: "${args.repo}") {
        pullRequest(number: ${args.pr}) {
          state
          reviews(first: 10, states: [APPROVED]) {
            totalCount
          }
        }
      }
    }
  }"""
  def response = githubQuery(args + [query: query])
  def prInfo = response?.data.organization.repository.pullRequest
  def shouldBuild = (
    // Skip merged pull requests
    prInfo.state != "MERGED" &&
    // Check for draft state
    (prInfo.state != "DRAFT") &&
    // Check for approval
    (prInfo.reviews.totalCount > 0)
  )
  shouldBuild
}

monitorRecentlyApprovedPRs organization: "Stalker-4x4", auth: "Stalker-4x4", multibranch: "Test"

if (shouldBuildPR(organization: "Stalker-4x4", repo: "PR-REPO", auth: "Stalker-4x4", pr: env.CHANGE_ID) == false)
  exit 1
  


