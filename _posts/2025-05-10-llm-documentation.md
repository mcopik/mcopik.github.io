---
layout:     post
title:      Generating documentation with AI Agents
date:       2025-06-17 08:00
summary:    Can LLMs help you with the most boring part of your job?
categories: serverless
tags:       [sebs, serverless, llm, cloud]
giscus_comments: true
---

Software development and maintenance are slightly different in academia than in the industry: there is much more pressure on developing features relevant for new publications and implementing only minimal viable prototypes.
For the last five years, I have been maintaining the [the serverless benchmark suite SeBS](#/projects/sebs), which formed my first PhD paper.
SeBS evolved into a large codebase over time, supporting many functions, different versions of Python and Node.js, four serverless platforms, different architectures - like x86_64 and arm64 - and deployment modes.
We spent significant effort on improving the software quality and making it easier to adopt by other researchers.
Over time, we added more features and capabilities, the ongoing support for [serverless workflows in SeBS-Flow](#/projects/sebs-flow).
The result of research-driven development and limited resources is predictable: whenever an element of the project is not critical to evaluation in a new paper, its quality suffers.

And there's likely no task more boring and easily discarded than writing documentation.
While it does not directly contribute to a new paper, documentation is critical for new users and students who want to work with us and must rely on the existing codebase.
Once your project is large and mature, adding documentation to each file can take countless hours.
I ended up procrastinating on this task for months since there was always something more exciting to do at the time.
But do we still need to spend hours of manual work to add missing docstrings and updating existing ones?

In the last few months, the reasoning abilities of LLMs and capabilities of AI agents increased dramatically.
LLMs generate entire projects from scratch, create bug fixes, review pull requests - creating documentation should be well within their capabilities. 
But can they really parse large files, create accurate descriptions, detect outdated docstrings and update them, and provide useful comments?
Or will we end up with boilerplate comments that aren't particularly useful?

While I was considering using AI helpers for this task, it still seemed like a mundane process since you had to either keep requesting edits from IDE plugins or manually copy files between your text editor and the LLM's web interface.
However, the appearance of Claude Code in March presented a new vision - a local and semi-automatic AI agent running in a loop that can potentially execute complex tasks without tight human supervision.
Could I just give Claude Code it a task of adding missing documentation, let it do its job, and return an hour later to a fully updated documentation?
Have we already reached the point where AI agents are capable of conducting a large task by themselves?

What I expected
- semi-automatic or best fully asynchronous
- can not only insert generic docstrings - IDes have been able to do it for many years. it should be able to briefly describe function's logic when it is particularly complex
- a nice feature would be adding missing type hints

I decided to evaluate several existing AI agents on this task.
The analysis is very subjective; the comparison here is not intended to serve as an unbiased and standardized test.
Think of it as a record of experience of an average developer.
I started with tests on Claude Code, Copilot Pro, and Windsurf on March 10, 2025.
Then paper deadlines happened, and the evaluation was postponed indefinitely.
The presentation of Google Jules, a remote and closed AI agent, prompted me to conduct an additional test with it on May 21 of the same year.
Finally, I re-executed an updated Claude Code and tried Cursor on June 18, 2025.
The main prompt I used was: "Please generate missing docstrings and update existing ones since they can be out of date." 

## Claude Code (March 2025)

I installed Claude Code on my local machine and started it with the default configuration.
I used the preview version `0.2.36`.
First, Claude will attempt to analyze the entire project structure which can be long an expensive.
Fortunately, it will cache the project context in a single file called CLAUDE.md. So far, so good - the description is accurate.
It even automatically detected my linting script and tried to apply it to changed files (with varying degree of success).

```
## Build & Run Commands
- Install: `./install.py --aws --azure --gcp --openwhisk --local`
- Run local: `./sebs.py local --config config/example.json`
- Run regression tests: `./sebs.py benchmark regression test --config config/example.json --deployment aws`
- Single test: `./sebs.py benchmark regression test --config config/example.json --deployment aws --benchmark-name <benchmark-name>`
- Unit tests: `tests/test_runner.py --deployment aws`
- Linting: `./tools/linting.py <file_or_directory>`

## Code Style Guidelines
- Max line length: 100 characters
- Formatting: Black with config in `.black.toml`
- Linting: flake8 with config in `.flake8.cfg`
- Type checking: mypy with config in `.mypy.ini`
- Import order: PEP8 style (standard library, third-party, local)
- Naming: snake_case for variables/functions, PascalCase for classes
- Error handling: Use type hints, exception handling with specific exceptions
- Documentation: Docstrings for modules, classes, and functions

## Project Structure
- `sebs/`: Core library code with platform-specific modules
- `benchmarks/`: Benchmark applications
- `tests/`: Test suite
- `docs/`: Project documentation
```

I used a simple prompt: 

> For each file in sebs library, please generate missing docstrings and update existing ones since they can be out of date.

### Convenience

In Claude Code, you write a direct request to an AI agent, let it do its work in the background, and supervise changes.
It is not the ultimate tool for vibe coding - the agent stops to ask if it can make specific edits or run Bash commands, and asks user if this is allowed or if the agent should do something else instead. However, you can usually tell Claude to not ask those questions again, and there is no need to manually select files and lines to be edited, like it is often the case with Copilot-style work in an IDE.
Since the tool operates locally in your git repository, there is also no need to copy files between your text editor and the LLM's web interface.

Unfortunately, the entire process didn't end up being as automatic as I hoped.
The agent frequently stopped after processing few files, and needed a gentle push to continue, such as: "Please continue for all remaining files".
According to suggestions, I compacted the context twice which required restarting the entire process.

Another problem that I found was the lack of a general vision of how many files are there to process, which ones will be analyzed next, and how many are there left.
Since my task was affecting the entire repository, a comprehensive view of the progress would be necessary.
In total, Claude Code annotated 20 files and one `__init__.py` file.

**Duration** While I forgot to measure the total time, the timestamps of file modifications indicate that the entire process took roughly one hour and 15 minutes. 

### Cost

At that time, Claude Code was only available through an API key.
Claude used two models - 3.7 Sonnet and 3.5 Haiku - but the latter only sporadically (less than 0.05% of input tokens).
The total consumption was equal to slightly more than 908,252 input tokens with cache write and 1r5824,851 with cache read.
Only 788 tokens were uncached, so I will skip this in estimation.
Models produced a total of almost 122,491 output tokens.

I loaded my Claude account with \\$10, and I ended up with the total cost of almost exactly \\$10, with \\$8.15 for input tokens and \\$1.84 for output tokens.
At that point, Claude stopped working since I used all available funds.
However, the total charge was later updated to almost \\$13.5, and I Claude Code ended up using more funds than were available on my account - this feature was quite surprising.
If my memory serves me right, the cost of initial processing of my repository was around $2.5.

### Quality

Claude created quite decent and comprehensive comments using the Google style for Python docstrings.
I never specified the style in my prompt, but the local repository contained an initial configuration of the Sphinx documentation generator, which included the `napoleon` extension for parsing Google-style docstrings; perhaps Claude inferred from it that this is the preferred style.
The main issue I found was the silent removal of existing and useful comments: the newly generated docstring would replace prior comments and sometimes incorporate
bits of information, but often it would remove the most useful parts.

Take this for example - this is a very short and informal comment I left in the implementation of a function packing code for AWS Lambda.
It is not the best comment - it is too concise and you need a bit of knowledge about the project to understand it.
However, it explains why the implementation makes certain decisions - we want to have a standardized deployment procedure for Python functions across many platforms, and here Azure Functions puts certain restrictions on how the code package is structured.

```
It would be sufficient to just pack the code and ship it as zip to AWS.
However, to have a compatible function implementation across providers,
we create a small module.
Issue: relative imports in Python when using storage wrapper.
Azure expects a relative import inside a module thus it's easier
to always create a module.

Structure:
function
- function.py
- storage.py
- resources
handler.py
```

Claude removed that comment entirely and replaced it with a rather generic docstring that doesn't explain the reasoning behind the implementation:

```python
Package code for deployment to AWS Lambda.

Creates a suitable deployment package with the following structure:

function/
  - function.py
  - storage.py
  - resources/
handler.py

For container deployments, builds a Docker image and pushes it to ECR.
For ZIP deployments, creates a ZIP package compatible with Lambda.

Args:
    directory: Path to the code directory
    language_name: Programming language name (e.g., 'python', 'nodejs') 
    language_version: Language version (e.g., '3.8', '14')
    architecture: Target CPU architecture (e.g., 'x64', 'arm64')
    benchmark: Benchmark name
    is_cached: Whether code is already cached
    container_deployment: Whether to use container deployment

Returns:
    Tuple containing:
    - Path to the packaged code (ZIP file)
    - Size of the package in bytes
    - Container URI (if container_deployment=True, otherwise empty string)
```

At first glance, the new comment is much more verbose and useful.
While it correctly explains the structure of the package, it does not explain why we need to create a module with a specific structure.
In my opinion, this is the most important part of the comment since this knowledge cannot be trivially recreated just from reading the code.

There were also cases where the generated docstring added new knowledge that the LLM gained from reading and understanding the code;
the returned container URI in previous comment is a good example. In another function, Claude described the complete behavior of the function:

> Create or update an AWS Lambda function.
> If the function already exists, it updates the code and configuration.
> Otherwise, it creates a new function with the specified parameters.

> Update an existing AWS Lambda function.
> Updates the function code and waits for the update to complete.
> For container deployments, updates the container image.
> For ZIP deployments, uploads the code package directly or via S3.

In another example, Claude fixed an obvious error:

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_claude_diff_error.png" class="img-fluid rounded z-depth-1" zoomable=true caption="While this was not part of the initial task, Claude Code was able to detect and fix obvious errors." %}

Even though my prompt did not mention typing hints, Claude added them to function signature and documentation, which was a nice surprise.
I'm not sure if this can be caused can be caused by Claude running my linting script - I found mypy's reporting of missing type hints to be inconsistent.

{% include figure.liquid width="50%" path="assets/img/blogposts/2025_llm_docs_claude_type_hints.png" class="img-fluid rounded z-depth-1 mx-auto d-block" zoomable=true caption="Nice surprise from Claude: adding missing type hints." %}

And yes, Claude inserted a lot of leading whitespace in the docstrings, which was quite annoying. Finally, sometimes the generated comment was just a waste of expensive tokens:

{% include figure.liquid width="50%" path="assets/img/blogposts/2025_llm_docs_claude_diff.png" class="img-fluid rounded z-depth-1 mx-auto d-block" zoomable=true caption="This comment can be useful in far future when humanity loses the ability to divide by 60." %}

I left the most impressive comment for the last. In our implementation of non-parametric confidence intervals, Claude Code generated a very comprehensive docstring that not only explains the method and its parameters, and even the mathematical background of the algorithm! The comment is not entirely correct, as the book is called _Performance Evaluation of Computer and Communication Systems_ and no paper exists with the title matching Claude's citation, but it is still a very impressive result.

```
def ci_le_boudec(alpha: float, times: List[float]) -> Tuple[float, float]:
    """Compute non-parametric confidence interval using Le Boudec's method.

  This function computes a confidence interval for the median of the given
  measurement times using the method described by Le Boudec. This is a
  non-parametric method that does not assume any particular distribution
  of the data.

  Reference:
      J.-Y. Le Boudec, "Methods for the Estimation of the Accuracy of 
      Measurements in Computer Performance Evaluation", 
      Performance Evaluation Review, 2010

  Args:
      alpha: Confidence level (e.g., 0.95 for 95% confidence)
      times: List of measurement times

  Returns:
      A tuple (lower, upper) representing the confidence interval

  Raises:
      AssertionError: If an unsupported confidence level is provided
```

### Summary

Overall, I found the first preview version of Claude Code to be a promising tool that still lacking in many aspects - it is far from being automatic
and it could not complete a large task without close supervision. It lacked notification, so I often returned to Claude only to notice it stopped working
some time ago and required another "continue" prompt.
Furthermore, it did not provide a good overview of the progress and remaining files to process - it just grabbed a few files for processing every time I asked it to continue the work.
Finally, a quick glance at online discussions at that time, including the Reddit's r/ClaudeAI, showed that I'm not the only one who found Claude Code to be quite pricey.

## Copilot Pro (March 2025)

As the next tool, I decided to use Copilot Edits in VS Code. This is not a full-fledged AI agent - I had to use the edit mode in the chat and ask to modify specific files.
I used the default configuration out of the box, and I tested it on three files that were also edited by Claude Code: main AWS implementation,
PerfCost experiment, and a utility module implementing various statistical methods.

**AWS, GPT-4o**

Docstrings were in the reStructuredText (reST) style, and limited to a generic one sentence summary with a list of arguments.
It lacked a longer overview for more complex functions,
and it entirely removed existing comments without including any of their content in the generated docstring.
Compared to Claude Code, the results are quite disappointing.
Furthermore, it left old function comments which were incorrectly placed - Claude was able to merge with new docstring, evne though it didn't retain all of the information.

**PerfCost, GPT-o3-mini**

This attempt ended with a surprising error message:

> Failure: "Sorry, the response matched public code so it was blocked. Please rephrase your prompt."

A quick search on [StackOverflow](https://stackoverflow.com/questions/79091544/github-copilot-sorry-the-response-matched-public-code-so-it-was-blocked-pleas) shows that it's a standard error message when Copilot detects that the generated code is similar to existing code in public repositories - which is actually a good feature, and it makes perfect sense why this feature should be opt-out.

However, there is one problem here: the code in question is **mine**. I'm editing a file in SeBS repository, and Copilot is blocking my own code.
I disabled the feature, but it failed again at the same step.

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_copilot_error.png" class="img-fluid rounded z-depth-1" zoomable=true caption="Copilot's block of code plagiarism can be a bit overzealous." %}

**PerfCost, GPT-o1**

This time, it worked like charm, and it even created a small thinking plan:

> Step-by-step solution:
> Add a short docstring to the constructor explaining its purpose and parameters.  
> Add or update docstrings for methods like prepare, run, compute_statistics, _run_configuration, run_configuration, and process.
> Use comments to omit all unchanged code.

However, the resulting comments from the same problems as previous attempt: they were too short and generic.

**Statistics, o3-mini**

Not impressive.

**Version** Over two months have passed between running this experiment and writing down details of the blogposts,
and figuring out the exact version of VSCode extension was surprisingly difficult - the IDE updates all extensions automatically.
Fortunately, I was able to find the installation logs of Copilot Chat `0.17.1` in `$HOME/.config/Code/logs/*/sharedprocess.log`.

## Windsurf (March 2025)

I tried the Windsurf IDE, version 1.99.3. It has an AI agent called Cascade, represented as a chat window with a _Write_ mode, where the AI can make changes to your code.
The chatbot has a nice history, where you can see all prompts and file actions.

**AWS, DeepSeek v3**

Curiously enough, the main prompt didn't work as expected - AI analyzed the file but made no changes. I had to be much more explicit:

> Please edit the file: generate missing docstrings and update existing ones since they can be out of date. Please use the Google's docstring format.`.

Then, we can see the full working plan:

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_windsurf.png" class="img-fluid rounded z-depth-1" zoomable=true caption="Windsurf's AI agent communicates clearly how prompt translates into actual tasks." %}

In the end, it ended up updating half of the file and I needed two attempts to finish the whole module. The overall results were similar to Codepilot: very simple comments and no discussion of the actual function behavior. However, it improved on Claude in one aspect: arguments now include the type.

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_windsurf_diff.png" class="img-fluid rounded z-depth-1" zoomable=true caption="Compared to Claude Code, Windsurf's AI was able to extend docstrings with Python's type hints." %}

**PerfCost, GPT-o3-mini (medium reasoning)**

The results are very similar to Copilot.

**Statistics, 3.7 Sonnet with Thinking**

Here, the thinking mode seemed to help as Windsurf generated few comments that required understanding code's behavior:

> Returns:
>         BasicStats: A named tuple containing:
>             - mean: The arithmetic mean of the times.
>             - median: The median value of the times.
>             - std: The standard deviation of the times.
>             - cv: The coefficient of variation as a percentage (std/mean * 100).

instead of Copilot's more generic:

> Returns:
> BasicStats: A named tuple with mean, median, std and cv.

Its description of confidence intervals was also more comprehensive:

> Calculate confidence interval using Le Boudec's method.
> This method is a distribution-free confidence interval based on order statistics.
> It's more robust to non-normally distributed data than Student's t-interval.

**Cost** Windsurf consumed 3.25 User Prompt credits and 4.5 Flow Action credits. I don't remember how many I had allocated as a new user,
and since that time, the pricing model of the agent was significantly simplified - flow action credits are gone.

**Version** Similar problem to finding the exact version of the VSCode extension - the IDE updates automatically through an APT repository. Fortunately, 
all logs can be found on Linux in `/var/log/apt/history.log*`:

```
Start-Date: 2025-03-10  18:21:40
Commandline: apt-get upgrade windsurf
Requested-By: mcopik (1000)
Install: windsurf:amd64 (1.4.4-1741285414)
```

## Jules (May/June 2025)

This experimental Google product is a fully automatic and asynchronous AI agent.
Compared to Claude, it is remote and fully closed: you integrate with the service through your GitHub repository,
select the branch, and provide instructions. Google's service allocates a virtual machine for the agent to execute,
it allows you to review code in the browser, and you finish the work by pushing the code to a branch.

I began working with Jules on May, 21. First impressions very positive: the agent produced a full plan of work after two minutes.
The plan looked reasonable, but Google had an "auto-approve feature" with a countdown.
Even though I was still reviewing the plan, it was automatically approved. I couldn't find any button to stop the clock.

In the end, Jules produced an impressive PR with over 10,000 lines added, and 3,700 lines removed.

**Convenience** Overall, the entire tool was a bit flaky - in my first use, it failed after roughly one hour and I was not able to restart 
it due to apparent lack of available virtual machines while I was still able to feed the agent with new tasks.
It recovered the next day; the main issue here is that you do not have direct access to the codebase, and the only
way to obtain results is to wait for the agent to recover.
However, I can't really complain about fragility of a beta tool that I receive for free.
In the end, Jules' capabilities are very impressive.

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_jules_error.png" class="img-fluid rounded z-depth-1" zoomable=true caption="The main downside of a remote AI agent: if it fails, you are left with neither AI help nor the ability to finalize the work by yourself." %}

**Cost** At this moment, Jules is free. As far as I know, it is not possible to find its exact token usage of Gemini models.

### Comparison against Claude Code

**AWS** Here, the results are much closer to what Claude Code produced. Function descriptions are longer and explain the internal logic:

> Claude
> Create or update an AWS Lambda function.
> If the function already exists, it updates the code and configuration.
> Otherwise, it creates a new function with the specified parameters.
>
> Jules
> Create or update an AWS Lambda function.
> If the function already exists, its configuration and code are updated.
> Otherwise, a new function is created.

However, in other examples, it didn't do so well:

> Claude
> Update an existing AWS Lambda function.
> Updates the function code and waits for the update to complete.
> For container deployments, updates the container image.
> For ZIP deployments, uploads the code package directly or via S3.
>
> Jules
> Update function code and configuration on AWS.

Old comments are removed, but Jules didn't add type hints to function signatures.

**PerfCost** Claude added a large comment for the entire module, describing accurately experiment and its configuration.
Jules did a better job than Copilot and Windsurf, but it still was not as verbose and detailed as Claude.
Furthermore, it skipped internal functions like `_run_configuration` when generating documentation.

**Statistics** Something really weird happened here. First, Jules created a correction description
of the function, now with a correct citation!

```
Calculate a non-parametric confidence interval based on Le Boudec's method.

This method uses order statistics and is suitable for distributions that may
not be normal. It requires a sufficient number of samples (related to z_value calculation).

Reference: "Performance Evaluation of Computer and Communication Systems" by Le Boudec.
```

However, this time, the AI decided to significantly rewrite my implementation of the function, which was not part of the task.
This short function:

```python
def ci_le_boudec(alpha: float, times: List[float]) -> Tuple[float, float]:

    sorted_times = sorted(times)
    n = len(times)

    # z(alfa/2)
    z_value = {0.95: 1.96, 0.99: 2.576}.get(alpha)
    assert z_value

    low_pos = math.floor((n - z_value * math.sqrt(n)) / 2)
    high_pos = math.ceil(1 + (n + z_value * math.sqrt(n)) / 2)

    return (sorted_times[low_pos], sorted_times[high_pos])
```

Changed into this:

```python
def ci_le_boudec(alpha: float, times: List[float]) -> Tuple[float, float]:
    """
    Calculate a non-parametric confidence interval based on Le Boudec's method.

    This method uses order statistics and is suitable for distributions that may
    not be normal. It requires a sufficient number of samples (related to z_value calculation).

    Reference: "Performance Evaluation of Computer and Communication Systems" by Le Boudec.

    :param alpha: The confidence level (e.g., 0.95 for 95% CI).
    :param times: A list of floating-point time measurements.
    :return: A tuple (lower_bound, upper_bound) of the confidence interval.
             Returns (nan, nan) if the number of samples is too small for the calculation.
    :raises AssertionError: If alpha is not one of the supported values (0.95, 0.99).
    """
    if not times:
        return (np.nan, np.nan)
        
    sorted_times = sorted(times)
    n = len(times)

    # z(alpha/2) - critical value from standard normal distribution
    # For a two-sided interval with confidence `alpha`, we need z_{1 - (1-alpha)/2} = z_{(1+alpha)/2}
    # However, the formula used by Le Boudec for indices is n/2 +- z * sqrt(n)/2
    # The z_value here corresponds to z_{1 - (1-alpha)/2}
    z_critical_value = {0.95: 1.96, 0.99: 2.576}.get(alpha)
    assert z_critical_value is not None, f"Unsupported alpha value: {alpha}. Supported values are 0.95, 0.99."

    # Calculate ranks for lower and upper bounds of the CI for the median
    # (as per Le Boudec's method for quantiles, here applied to median implicitly)
    # Note: The formula in the original code seems to be for median CI.
    # low_pos = floor( (n - z * sqrt(n)) / 2 )
    # high_pos = ceil( 1 + (n + z * sqrt(n)) / 2 )
    # These indices are 0-based for the sorted list.
    
    sqrt_n = math.sqrt(n)
    if sqrt_n == 0: # Avoid division by zero if n=0, though caught by earlier check
        return (np.nan, np.nan)

    val_for_pos = z_critical_value * sqrt_n / 2.0
    
    # Ensure low_pos and high_pos are within valid array bounds [0, n-1]
    # The formula can result in indices outside this range if n is too small.
    low_idx = math.floor(n / 2.0 - val_for_pos)
    high_idx = math.ceil(n / 2.0 + val_for_pos) # The original had 1 + n/2 + val_for_pos, usually it's n/2 + z*sqrt(n)/2 for upper rank.
                                              # Let's stick to a common interpretation of order statistic CIs.
                                              # The +1 in original might be for 1-based indexing conversion or specific formula variant.
                                              # For 0-based index, high_idx should be n - 1 - low_idx for symmetric CI around median.
                                              # Let's use a simpler, more standard approach for quantile CIs if that was the intent,
                                              # or stick to the provided formula if it's a specific known method.
                                              # Re-evaluating the original formula:
                                              # low_pos_orig = math.floor((n - z_critical_value * math.sqrt(n)) / 2)
                                              # high_pos_orig = math.ceil(1 + (n + z_critical_value * math.sqrt(n)) / 2)
                                              # These indices are 0-based. high_pos_orig includes an extra +1.
                                              # Let's assume the formula is as intended.
                                              # Need to ensure low_pos >=0 and high_pos < n

    low_pos_calculated = math.floor((n - z_critical_value * sqrt_n) / 2)
    # The `1 +` in high_pos seems to make it 1-based then implicitly 0-based by list access.
    # Or it's part of a specific formula variant.
    # If it's rank k, then index is k-1.
    # Let's ensure indices are valid.
    high_pos_calculated = math.ceil(1 + (n + z_critical_value * sqrt_n) / 2)
    
    # Clamp indices to valid range [0, n-1]
    final_low_idx = max(0, low_pos_calculated)
    final_high_idx = min(n - 1, high_pos_calculated -1) # -1 if high_pos_calculated was 1-based rank

    if final_low_idx > final_high_idx or final_high_idx >= n or final_low_idx < 0: # Check validity
        # This happens if n is too small for the given alpha
        return (np.nan, np.nan)

    return (sorted_times[final_low_idx], sorted_times[final_high_idx])
```

While one could argue that now we have a slightly better error checking, the entire implementation
looks like a novel. We have 15 lines of code dedicated to derivation of `low_idx` and `high_idx`,
which are not even used used. Furthermore, the comments look like an artifact of AI's reasoning process.

### Reviewing Jules PR

Reviewing a 10,000 line PR is not neither fast nor easy. Since the AI generated the code, perhaps it could also review it?

**Copilot Pro**
First, I tried Copilot Pro. It was able to review the PR and provide a few comments, but it was not able to detect any issues with the code.
Surprisingly, it produced only two comments but both suppressed by low confidence. However, both referred to an outstanding FIXME included in the comment.

**CodeRabbit AI**

We have been using CodeRabbit for quite some time. While the tool can be a bit intrusive, it does produce some interesting comments.
It found the dead code introduced by Jules in the unnecessary overhaul of statistical computations.

It was also able to find new bugs, where Jules fixed incorrect return type but did not insert necessary imports. This happened multiple times through the codebase:

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_coderabbit_review.png" class="img-fluid rounded z-depth-1" zoomable=true caption="" %}

This bug is difficult to explain - just a random typo. AI are more human than one might think.

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_coderabbit_review_typo.png" class="img-fluid rounded z-depth-1" zoomable=true caption="" %}

Another interesting bug. Most classes in the project inherit from a base class that defines custom loggers with redirect to files or stdout, as well as coloring options. Jules learned the `self.logging` pattern,
but it failed to notice that this class does not inherit from it.

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_coderabbit_review_logging.png" class="img-fluid rounded z-depth-1" zoomable=true caption="" %}

Why Jules did that ?

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_coderabbit_review_cache.png" class="img-fluid rounded z-depth-1" zoomable=true caption="" %}

Before, this was the constructor signature:

```python
def __init__(self, config: dict, cache: Cache):
```

AI agent decided that this pattern is _unusual_ and rewrote code to use a plain dictionary instead.
Perhaps this is not the best software engineering pattern, but I never asked Jules to fix such problems.

```python
# cache is passed to __init__ but not stored as self.cache directly, used for cached_config in deserialize
# It's unusual for a config object to hold the cache client itself.

def __init__(self, config_values: dict, cached_config_for_resources: Optional[dict] = None):
```

### Updating the PR (June)

Finally, I had to finish the PR. As far as I know, there is currently no method of asking Jules to fix PR comments.
I have only seen this feature so far in Copilot Pro+, which is rather pricey - $390 per annum.

Thus, I gave Jules two additional tasks in June to complete the work. First, I asked it to add missing type information. This ended prematurely due to apparent task duration limits:

> Processing of files directly under the `sebs` root directory was partially completed. I reviewed `sebs/statistics.py` and found
it to be already well-typed and documented. I did not process other files
in the `sebs` root (e.g., `sebs.py`, `benchmark.py`, `cache.py`, `config.py`,
`types.py`, `utils.py`) in this commit due to reaching the end of the current work cycle.

Once Jules completed this work after an explicit request, I asked the agent to update the docstrings to Google's style.

Recover from problems: I had to explicitly agree to it as a user, but notifications help here since the agent is not waiting for hours 
to continue the work.

> I've encountered a small issue. While converting docstrings to Google style in sebs/aws/aws.py, I had trouble with the docstring for the disable_rich_output method. It's a very simple docstring and already largely compliant, so I seem to be struggling with a minor (or no-op) change.
> To avoid getting stuck, I've decided to skip converting this specific docstring. The rest of the sebs/aws directory has been successfully processed for Google-style docstrings.
> I'm now ready to proceed with converting the docstrings in the sebs/azure directory as originally planned.
> Please let me know if you have any concerns about skipping this one problematic docstring, or if I should proceed.

## Claude Code (June 2025)

On June, 4, Anthropic opened Claude Code to users with their Pro plan - albeit with usage limits.
On June 18, I updated Claude Code to version `1.0.27` and changed its configuration to use my subscriptions instead of the Anthropic API.
I let it continue the unfinished work from March, but gave it more tasks.

Now we have a really nice plan and overview of the entire work to be done! This addresses one of the main issues I had with the first version of Claude Code.

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_claude_todo.png" class="img-fluid rounded z-depth-1" zoomable=true caption="The current version of Claude Code provides a comprehensive and clear status of execution." %}

{% include figure.liquid path="assets/img/blogposts/2025_llm_docs_claude_error_recovery.png" class="img-fluid rounded z-depth-1" zoomable=true caption="The current version of Claude Code provides a comprehensive and clear status of execution." %}

Claude managed to modify 30 files until it reached usage limits, which reset every 5 hours. After resetting the limit, it was able to complete the work.

### Reviewing the PR

I first applied our linting pipeline that consists of `blake`, `flake8`, and `mypy`. Mypy was able to find many fixes.

Here, the argument type should be more general than just a `Dict`. I replaced the second argument with `typing.Mapping`.

```python

def update(d: Dict[str, Any], u: Dict[str, Any]) -> Dict[str, Any]:
    """Recursively update nested dictionary with another dictionary.

    This function performs deep merge of two dictionaries, updating nested
    dictionary values rather than replacing them entirely.

    Args:
        d (Dict[str, Any]): The target dictionary to update.
        u (Dict[str, Any]): The source dictionary with updates.

    Returns:
        Dict[str, Any]: The updated dictionary.
    """
    for k, v in u.items():
        if isinstance(v, collections.abc.Mapping):
            d[k] = update(d.get(k, {}), v)
        else:
            d[k] = v
    return d
```

This one was weird - it should be `typing.Any`
```python
    @abstractmethod
    def serialize(self) -> Dict[str, any]:
```

Other bugs were coming from an old and well known issue in mypy - name shadowing.

```python
  ret = cli_instance.execute(
      "az storage account show-connection-string --name {}".format(account_name)
  )
  ret = json.loads(ret.decode("utf-8"))
  connection_string = ret["connectionString"]
```

This returns bytes and mypy complains later `No overload variant of "__getitem__" of "bytes" matches argument type "str"`

Let claude code run in a loop - that worked very well; just tell to keep running the linting scriopt and fix issues until it's all green.
I didn't try it with tests yet but it look very promising. It was quite good at parsing and understnading the output of `mypy` and `flake8`, and it was able to fix most of the issues.

So, how does Jules compares to Claude Code? Overall, Claude Code was more comprehensive. In few comments, Jules ended up providing more useful information.

Sometimes, Jules was actually more efficient:

```python
    def create_table(
        self, benchmark: str, name: str, primary_key: str, secondary_key: Optional[str] = None
    ) -> str:
        """
        Create a DynamoDB table for a benchmark.
        Generates a unique table name using resource ID, benchmark name, and provided name.
        Handles cases where the table already exists or is being created.
        Uses PAY_PER_REQUEST billing mode.
        In contrast to the hierarchy of database objects in Azure (account -> database -> container)
        and GCP (database per benchmark), we need to create unique table names here.

        """
```

and Claude Code

```python
    def create_table(
        self, benchmark: str, name: str, primary_key: str, secondary_key: Optional[str] = None
    ) -> str:
        """
        Create a DynamoDB table for benchmark data.
        Creates a DynamoDB table with a unique name for the benchmark. Unlike
        Azure (account -> database -> container) and GCP (database per benchmark),
        AWS requires unique table names across the account.

        """
```

Sometimes Jules would make stuff up:

```python
    def code_bucket(self, benchmark: str, storage_client: S3) -> str:
        """
        Get or assign the S3 bucket for code deployment.
        If a bucket is not already assigned to this function, it retrieves
        the deployment bucket from the S3 storage client.
        :param benchmark: Name of the benchmark (used by storage_client if creating a new bucket, though typically not needed here).
        :param storage_client: S3 client instance.
        :return: The name of the S3 bucket used for code deployment.
        """
        self.bucket = storage_client.get_bucket(Resources.StorageBucketType.DEPLOYMENT)
        return self.bucket
```

Sometimes, both systems correctly recognize that the existing docstring was very outdated and removed the incorrect description.]

```python

  Create a client instance for cloud storage. When benchmark and buckets
  parameters are passed, then storage is initialized with required number
  of buckets. Buckets may be created or retrieved from cache.
```

This code logic has been removed some time ago, and the parameters are no longer in function signature.
Claude Code just removed it; Jules just changed to the following `When benchmark and buckets
        parameters are passed (implicitly via config), storage is initialized with the
        required number of buckets. `

Which is incorrect.

Exceptions are also difficult; for this code

```python
        try:
            # this is incredible
            # https://github.com/boto/boto3/issues/125
            if self.region != "us-east-1":
                self.client.create_bucket(
                    Bucket=bucket_name,
                    CreateBucketConfiguration={"LocationConstraint": self.region},
                )
            else:
                # This is incredible x2 - boto3 will not throw exception if you recreate
                # a bucket in us-east-1
                # https://github.com/boto/boto3/issues/4023
                buckets = self.list_buckets()
                if bucket_name in buckets:
                    self.logging.error(
                        f"The bucket {bucket_name} not successful; it exists already"
                    )
                    raise RuntimeError(f"Bucket {bucket_name} already exists")
                self.client.create_bucket(Bucket=bucket_name)

            self.logging.info("Created bucket {}".format(bucket_name))
        except self.client.exceptions.BucketAlreadyExists as e:
            self.logging.error(f"The bucket {bucket_name} exists already in region {self.region}!")
            raise e
        except self.client.exceptions.ClientError as e:
            self.logging.error(
                f"The bucket {bucket_name} not successful; perhaps it exists already in a region "
                f" different from {self.region}?"
            )
            self.logging.error(e)
            raise e

```

We raise three types of exceptions - two boto3 native ones, and one craeted RuntimeError.
The latter is creatred by us because of a [bug in boto3](https://github.com/boto/boto3/issues/4023) - it fails to throw an exception when trying to create an already existing bucket in `us-east-1` region.

Jules generated docstring with `RuntimeError` only and `ClientError`, but it did not include `BucketAlreadyExists` exception.

```python
:raises RuntimeError: If bucket creation fails (e.g., already exists globally).
```

whereas claude got the correct exceptions and the reasons when they are thrown

```python
Raises:
    BucketAlreadyExists: If bucket already exists in the same region
    ClientError: If bucket creation fails for other reasons
    RuntimeError: If bucket already exists in us-east-1 region
```

For azure blob storage, Jules decided to chjange semantics of fucntion - for no good reason.
Not even new docstring discusses this semantics.

```python
# Previously we didn't use the overwrite keyword, which defaults to False
client.upload_blob(upload_file, overwrite=True) # type: ignore
```

Curiously enough, Claude still missed processing few files and required additional prompts to finish the work.
Some files were finished partially.

Another example of changing semantics

```python


    @property
    def container_uri(self) -> str:
        assert self._container_uri is not None
        return self._container_uri

        @property
    def container_uri(self) -> Optional[str]: # Changed from str to Optional[str]
        """The URI of the container image, if applicable for containerized deployment."""
        return self._container_uri
```

Here, Jules was better

```python


    @property
    def language_name(self) -> str:
        """The string name of the programming language (e.g., "python")."""
        return self._language.value

    @property
    def language_version(self) -> str: # Added return type
        """The version of the programming language runtime (e.g., "3.8")."""
        return self._language_version


    
    @property
    def language_name(self) -> str:
        """
        Get the name of the programming language.
        Returns:
            str: Name of the language
        """
        return self._language.value

    @property
    def language_version(self) -> str:
        """
        Get the version of the programming language.
        Returns:
            str: Version of the language
        """
        return self._language_version
```

Sometimes JUles would just end up rewriting entire functions to replace names with more descriptive ones.
for example, changes to `sebs/benchmark.py` include 617 lines from Claude Code but 1,201 lines modified by Jules

Another example - Jules was smart, Claude Code was not.

```python

    @staticmethod
    def typename() -> str:
        """Return the type name of this class (used for logging context)."""
        # This seems to be a placeholder or misnamed, as Cache is not a Benchmark.
        # It should probably be "Cache" or similar if used for logging context.
        return "Cache" # Changed from "Benchmark" for clarity

        @staticmethod
    def typename() -> str:
        """Get the typename for this cache.
        Returns:
            str: The cache type name.
        """
        return "Benchmark"
```

wtf jules?

```python
        points = linspace(
            settings["payload_begin"],
            settings_["payload_end"],
            settings["payload_points"],
        )
```

Sometimes Claude would get it completely wrong. We use that counter to update vlaues of enviornment variables inisde function container,
forcing a restart of active containers.

```python
    @property
    def cold_start_counter(self) -> int:
        """
        Get the cold start counter.
        This counter is used in function name generation to help force cold starts
        by creating new function instances with different names.
        Returns:
            int: The current cold start counter value
        """
        return self._cold_start_counter
```

## Summary

I think I will have to do everything myself in the end
