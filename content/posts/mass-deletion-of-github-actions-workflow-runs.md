---
title: "Mass Deletion of Github Actions Workflow Runs By Name and Date"
date: 2024-08-28T16:52:24+01:00
tags:
    - github-actions
    - Github
    - jq
    - api
categories:
    - development
keywords:
    - github-actions
    - shell
    - jq
    - tools
    - github
---
{{< notice tip >}}
TL;DR: https://gist.github.com/StivDC/e793340564d23e084a8c8f6429e1ae42
{{< /notice >}}

## Intro
Deleting Github Action workflows the [old fashioned way](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-workflow-runs/deleting-a-workflow-run) can be painful. I for example had a repository which had old workflows that were no longer used as seen below. These old workflows were in an abundance, in both their name and how many times they were run and deleting them one by one just was not going to cut it.

{{< img src="images/mass-deletion-of-gh/all_workflows.png" alt="Old Workflows" >}}

### My use case and previously made solutions
I knew the answer to this lied within creating a script using the [GitHub API](https://docs.github.com/en/rest?api), more specifically the [workflow runs API](https://docs.github.com/en/rest/actions?apiVersion=2022-11-28#workflow-runs). I've used GitHub's API briefly before, but only for testing and reading, not to create or delete. 

My objective was clear, I wanted to either get rid of all workflows under a specific name, or to remove workflows before a certain date. In future, I would even want to add the functionality to select multiple workflows and delete en masse. 

Before I went into possibly reinventing the wheel, I did a quick google and came across two articles to get me started: [DJ Adam's very handy script "mass-deletion-of-github-actions-workflow-runs"](https://qmacro.org/blog/posts/2021/03/26/mass-deletion-of-github-actions-workflow-runs/) and then to fit my date use case [Michael Heap's post "Filter for dates before today with jq"](https://michaelheap.com/jq-select-date-before-today/)

I would fully recommend following DJ's post on the matter, it's very easy to follow and provides a very good starting point for customising what you need from GitHub's API. The following is a variation of his script. 

## Prerequisites
- [GitHub API Authentication](https://docs.github.com/en/rest/using-the-rest-api/getting-started-with-the-rest-api?apiVersion=2022-11-28#making-a-request), preferably with a [PAT](https://docs.github.com/en/rest/authentication/authenticating-to-the-rest-api?apiVersion=2022-11-28#authenticating-with-a-personal-access-token). 
- [JQ](https://jqlang.github.io/jq/) (Json Processor), I'm using macOS so I installed JQ using [brew](https://formulae.brew.sh/formula/jq)
- The [GitHub CLI](https://cli.github.com/)
    - If you want to test out your API authentication using GH CLI run this command:
    {{< highlight bash "linenos=false" >}}
    gh api --method GET /octocat \
    --header 'Accept: application/vnd.github+json' \
    --header "X-GitHub-Api-Version: 2022-11-28"
    {{< / highlight >}}

## Creating the script

### Viewing your worflows
Using Github's API alone gives you a lot of data that isn't necessary, DJ provides a neat solution in using `jq` to filter down the parameters you want. 

{{<highlight bash "linenos=false">}}
gh api /repos/<owner>/<repo>/actions/runs \
 | jq -r '.workflow_runs[] | [.id, .conclusion, .name] | @tsv' \
 | head -5
{{</ highlight>}}

Which will neatly return:

{{< img src="images/mass-deletion-of-gh/recent-workflow-runs.png" alt="Recent Workflow Runs" >}}

<!-- because notice didn't want to leave any space -->
<br/><br/> 

{{< notice info >}}
There's built-in support for pagination with gh api, with the `--paginate` switch.
{{< /notice >}}

The different parts of the jq call are as follows:

| Part                      | Description                                                |
|---------------------------|------------------------------------------------------------|
| `-r`                        | Tells jq to output raw values, rather than JSON structures |
| `.workflows_runs[]`         | Process each of the entries in the workflow runs array     |
| `[.id, .conclusion, .name]` | Show values of these three properties                      |
| `@tsv`                      | Convert everything into tab separated values               |

### Getting all workflow runs by name and date
Here DJ continues to use [fzf](https://github.com/junegunn/fzf) to allow a user to select multiple workflows to delete in parallel. For my usecase, my primary goal was to completely rid of a specific workflow run, or runs before a certain date.

{{<highlight bash>}}
while getopts ":n:d:" opt; do
  case "$opt" in
    n) select_name=$OPTARG ;;
    d) select_date=$(date -u -j -f "%Y-%m-%d %H:%M:%S" "$OPTARG" +"%Y-%m-%dT%H:%M:%SZ") ;;
    # GH CLI uses format: 2024-08-23T17:12:55Z
    # Humans want to use format "2024-08-23 17:12:55"
  esac
done
{{</highlight>}}

I started the script by allowing command line arguments for a `name` and a human usable `date`. There is room in that script to only require the `year` `month` and `day` for input, but I wanted to be specific and allow input of `hour`, `minute` and `second`. 

#### Filtering runs by name and date
Next was to add in a function to, in a more readable fashion, pipe our `jq` script to the API call. 
{{<highlight bash "hl_lines=5">}}
jqscript() {

  cat <<EOF
    .workflow_runs[]
      | select((.name | startswith("$select_name")) and (.created_at < "$select_date"))
      | [
          (.conclusion),
          (.created_at),
          .id,
          .event,
          .name
        ]
      | @tsv
EOF

}
{{</highlight>}}

Line 5 here being of note, `startswith()` is used as if a user doesn't want to filter by a specific name we can pre-set `$select_name` to blank, where it will fetch all workflow runs. 

Then following Michael's advice, or in my case the inverse of his advice. 
> jq has built in support for ISO8601 format dates, so 2021-10-02 > 2021-10-02.
> — <cite>michaelheap [^1]</cite>
[^1]: The above quote is excerpted from Michael's post on [Filter for dates before today with jq](https://michaelheap.com/jq-select-date-before-today/).

### Deleting runs
Similar to how one can retrieve runs with the GH API using 
- `/repos/<owner>/<repo>/actions/runs`

 one can delete by using:
- `DELETE /repos/<owner>/<repo>/actions/runs/<run_id>`

Then it becomes a case of getting the selected runs, and passing them through a delete runs function:

{{<highlight bash>}}
local run id result
run=$1
id="$(cut -f 3 <<< "$run")"
gh api -X DELETE "/repos/$repo/actions/runs/$id"
[[ $? = 0 ]] && result="OK!" || result="BAD"
printf "%s\t%s\n" "$result" "$run")
{{</highlight>}}


The end result is this script here:
- https://gist.github.com/StivDC/e793340564d23e084a8c8f6429e1ae42

Feel free to comment on the gist and to use the script!
