# Part 2 - Detect secrets in Azure repos - Kimani Mbugua - Data and Technology blog
Even with the advent of cloud computing and all manner of technology enhancements, exposing secrets seems to be a problem that won’t go away.

Without the right controls in place, developers can leak secrets that can cause financial and reputational damage to an organisation.

In part 2, we’ll look at how we can use a pre-commit hook to try and detect secrets in our code.

Leaking secrets
---------------

#### What is a secret

> Taken directly from [cybark-secrets-management](https://www.cyberark.com/what-is/secrets-management/#:~:text=What%20is%20a%20Secret%3F,DevOps%20and%20cloud%2Dnative%20environments.), secrets are non-human privileged credentials that refer to a private piece of information that acts as a key to unlock protected resources or sensitive information in tools, applications, containers, DevOps and cloud-native environments.

Some of the most common types of secrets include:

*   Privileged account credentials
*   Passwords
*   Certificates
*   SSH keys
*   API keys
*   Encryption keys

A 2019 [academic study](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_04B-3_Meli_paper.pdf) from the North Carolina State University, trawled through millions (even billions) of GitHub pages to “find that not only is secret leakage pervasive — affecting over 100,000 repositories — but that thousands of new, unique secrets are leaked every day”.

These secrets can be found in a variety of places such as source code, packaged libraries, infrastructure as code scripts, configuration files, and even documentation. They also persist in commit histories and all the environments that they’re deployed to such as staging and production environments.

It’s not just the pervasive nature of the problem that is concerning but how [quickly](https://www.csoonline.com/article/3634784/how-corporate-data-and-secrets-leak-from-github-repositories.html) attackers can exploit the issue.

Preventing secret exposure
--------------------------

So now that we’ve established what secrets are, here are a few reasons why we should care about protecting them.

#### Financial loss

The most obvious reason is that data breaches cost money and according to [IBM’s Cost of a Data Breach Report](https://www.ibm.com/downloads/cas/RZAX14GX), on average, this is in the millions of USD. Sensitive data exposure can lead to a data breach, so it makes sense to protect sensitive data and in turn, prevent those secrets being exposes.

#### Reputation

Just as bad as the loss of money is the loss of reputation and a quick scan of the web will show many notables secret exposure cases.

#### Mindset change

One of the most important reasons to prevent secret exposure via automated checks is to change the behaviours of developers and companies.

By embedding secret detection checks, you help team members understand the importance of not exposing secrets and help prevent them doing so.

Detect secrets
--------------

2 key areas for embedding secrets are:

*   Developer workflows (e.g., Code check in)
*   Code integration and deployment

We’ll focus on the developer workflow area by using a pre-commit hook known as `detect-secrets`.

#### Follow along

If you are new to using pre-commit hooks, you can catch the [first part](https://www.kimanimbugua.com/post/azure-repos-pre-commit-hooks-part-1/) of this series to get a better understanding of what they are and how to use them.

The first part also get you started by making sure you have [pre-commit](https://pre-commit.com/#install) installed.

#### Yelp!

[detect-secrets](https://github.com/Yelp/detect-secrets) is an open-source library that uses [heuristics](https://en.wikipedia.org/wiki/Heuristic_(computer_science)) to determine if secrets have been detected.

> Yelp detect-secrets supports the following:

*   Secret detection
*   Baseline of existing secrets
*   Auditing of the baseline

### Example usage

This section will cover a basic example of how we can use the detect-secrets pre-commit hook.

1.  With pre-commit installed, open the `.pre-commit-config.yaml` file in the root of the repo and add the `detect-secrets` pre-commit hook as follows

```
-   repo: https://github.com/Yelp/detect-secrets
    rev: v1.0.3
    hooks:
    -   id: detect-secrets

```

2.  Create a file named `hello.py` with the following contents:

```
print('Testing secrets')

```

3.  Stage the `hello.py` file and run the `pre-commit run detect-secrets` command. If no secrets have been detected, the run should succeed as follows:

![](https://www.kimanimbugua.com/images/pre-commit-detect-secrets-passed.jpg)

4.  Amend the `hello.py` by adding the following line to the file:

```
secret = 'secretvalueABC123'

```

5.  Stage the `hello.py` file and run the `pre-commit run detect-secrets` command again and observe that the run should now fail similar to the following:

![](https://www.kimanimbugua.com/images/pre-commit-detect-secrets-example.jpg)

#### Ad hoc execution

The are many ways of running the `detect-secrets` python module and not just as a pre-commit hook.

A simple, “big bang” way of executing the command is by running, `detect-secrets scan --all-files`. As you’d expect, this will scan all files recursively for secrets and highlight potential secret exposure.

The module is reasonably well [documented](https://github.com/Yelp/detect-secrets#quickstart) so I’d invite you to go through the documentation if you want to dig deeper into the functionality available.

#### Setting a baseline

For environments that may already have secrets in code, it can be a daunting task to fix this. One way of tackling this problem could be through the use of a baseline.

A baseline is a file that contains secrets that you have detected and can be excluded from further scans. An obvious question to ask would be why would you want to do this? The following [section](https://github.com/Yelp/detect-secrets#about) in the documentation explains this well.

Essentially:

*   Identify existing issues
*   Set a new threshold to evaluate secrets detected from that point onwards

The following is an example of how we’d call this in a pre-commit hook after we’ve set a baseline.

```
-   repo: https://github.com/Yelp/detect-secrets
    rev: v1.0.3
    hooks:
    -   id: detect-secrets
        args: ['--baseline', '.secrets.baseline']

```

Considerations
--------------

##### Think beyond pre-commit

Although the topic of this post is around using pre-commit hooks for detecting secrets, it’s important to understand that the issue can be deeply entrenched so look past just before committing and anticipate situations where pre-commits are not enough or even available.

One way is to embed detect-secrets in release pipelines. A simple example is as follows:

```
- script: pip install detect-secrets==1.0.3
      displayName: "Install detect-secrets"

    - script: |
        detect-secrets --version
        detect-secrets -v scan --all-files --force-use-all-plugins --exclude-files FETCH_HEAD --baseline .secrets.baseline > $(Pipeline.Workspace)/detect-secrets.json
      displayName: "Run detect-secrets tool"

```

#### Other methods

There are other alternatives to [detect-secrets](https://github.com/Yelp/detect-secrets#about) such as [Git-secrets](https://github.com/awslabs/git-secrets), each with their pros and cons. As far as I know, there is no de-facto standard for detecting secrets via hooks, although I do like the philosophy that is behind `detect-secrets` in that it allows you to recognise existing issues as well as detect secrets.

#### Baselines and false positives

*   Keep a close eye on the secret baseline to avoid turning it a way of bypassing secret detecting controls.
*   False positives can get annoying so be patient to start with and recognise that as much as it can be helpful. no tool is perfect.

#### Windows install issues

The eagle-eyed amongst you may have noticed that in the examples I have used, I’ve referred to v1.0.3.

As I’m using a Windows machine to run these examples, I noticed that the latest version of the module does not work on Windows 10 or 11. It is related to the bug in the detect-secret tool (see more in [Issue#452](https://github.com/Yelp/detect-secrets/issues/452)) and I’d suggest you keep an eye out for the fix. To revert to the latest versions, you can simply remove the version tag ==1.0.3 in the pip install command.

#### Notable gaps

> There are some caveats that the module highlights as captured directly from [detect-secrets](https://github.com/Yelp/detect-secrets#caveats)

*   Multi-line secrets
*   Default passwords that don’t trigger the KeywordDetector (e.g., login = “hunter2”)

Summary
-------

In this 2nd part of the series, we looked at how detect-secrets can help identify potential secret leakage.

It’s important to understand that this post is not suggesting that by only using a pre-commit hook, we will solve all our problems with secret leakage.

The key message here is that if we can at least highlight potential issues with secret leakage at a pre-commit stage, then that is way better than not having any checks at all.

Thanks for reading and see you in part 3 where we’ll look at applying standards via pre-commit.

Useful links
------------

[Yelp - Detect secrets](https://github.com/Yelp/detect-secrets)

[How Bad Can It Git? Characterizing Secret Leakage in Public GitHub Repositories](https://www.ndss-symposium.org/wp-content/uploads/2019/02/ndss2019_04B-3_Meli_paper.pdf)

[The secrets about secrets](https://apiiro.com/the-secrets-about-secrets-in-code/)