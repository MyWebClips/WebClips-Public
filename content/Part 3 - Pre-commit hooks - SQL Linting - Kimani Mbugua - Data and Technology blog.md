# Part 3 - Pre-commit hooks - SQL Linting - Kimani Mbugua - Data and Technology blog
[Part 3 - Pre-commit hooks - SQL Linting - Kimani Mbugua - Data and Technology blog](https://www.kimanimbugua.com/post/azure-repos-pre-commit-hooks-part-3/) 

 [Part 3 - Pre-commit hooks - SQL Linting - Kimani Mbugua - Data and Technology blog](https://www.kimanimbugua.com/post/azure-repos-pre-commit-hooks-part-3/) 

 It can be a challenge to keep code formatted consistently and with a lack of consistency, errors soon follow.

In part 3 of this pre-commit hooks series, we’ll focus on how we can use pre-commit hooks in Azure git repos, to automatically check for stylistic and programmatic errors in SQL scripts.

If you are new to pre-commit hooks, I’d recommend that you review [part one](https://www.kimanimbugua.com/post/azure-repos-pre-commit-hooks-part-1/) of this series before you continue as some prerequisites will be required.

Linting
-------

Linting is the automated process of checking for stylistic and programmatic errors in code.

We want to perform linting for a number or reasons such as to:

*   Automate the checks performed
*   Standardise what is checked
*   Apply the process in the development or release workflow
*   Tease out code smells
*   Highlight formatting inconsistencies
*   Raise awareness of potential bugs in code
*   Encourage positive behaviours

> Ultimately, what we really want linting to do is to increase the code quality and correctness.

### Problems with SQL Linting

Languages such as Python have specific guidelines related to formatting and code style and therefore, plenty of linting [options](https://realpython.com/python-code-quality/#linters) exist.

Linting in SQL doesn’t appear as mature and stylistic linting can be an almost religious minefield for some of the following reasons:

*   Lack of official formatting standards
*   Difficult to enforce due to manually enforcing standards or using tooling that isn’t easy to use
*   Organisations have different formatting standards. Just look at some of the variations for aliasing in SELECT statements below:

```
  SELECT 1 AS one;
  SELECT 1 one;
  SELECT 1 AS 'one';
  SELECT one = 1;
  SELECT 1 AS [one];

```

SQL fluff
---------

Even in the absence of formal stylistic standards, there are ways of performing SQL linting.

One of the best SQL linters out there in the wild and the subject of this post is SQL Fluff.

> According to their [documentation](https://docs.sqlfluff.com/en/stable/), “SQL Fluff is an extensible and modular linter designed to help you write good SQL and catch errors before they hit your database”.

An awesome benefit of SQL Fluff is that you can run and fix linting issues using SQL Fluff via pre-commit.

This allows us to “write good SQL and catch errors before they hit your code base.” ❤️

Some additional benefits, which will be discussed later are as follows:

*   Configurable rules
*   [Code extensions for IDE](https://marketplace.visualstudio.com/items?itemName=dorzey.vscode-sqlfluff)
*   Supports multiple dialects e.g. ANSI, TSQL
*   [Well documented](https://docs.sqlfluff.com/en/stable/index.html)

### Linting example

In this section, we’ll show how easy it is to set up the SQL fluff pre-commit hook to apply consistent formatting. As usual, I’m using Visual Studio Code and where required, executing from the terminal.

1.  With [pre-commit installed](https://www.kimanimbugua.com/post/azure-repos-pre-commit-hooks-part-3/(https://www.kimanimbugua.com/post/azure-repos-pre-commit-hooks-part-1/)), open the `.pre-commit-config.yaml` file in the root of the repo and add the `sqlfluff-lint` pre-commit hook as follows

```
-   repo: https://github.com/sqlfluff/sqlfluff
    rev: 0.9.1
    hooks:
    -   id: sqlfluff-lint

```

In addition to the default rules, I’ve added a .sqlfluff [configuration](https://docs.sqlfluff.com/en/stable/configuration.html) file and populated additional rules as follows:

   ![](https://www.kimanimbugua.com/images/pre-commit-sqlfluff-config-file.jpg)
 

2.  In this step, we will have a file with mixed aliasing, new lines, inconsistent case for keywords and end of line spaces. Although this is syntactically correct, it doesn’t make for great reading. Create a file named `ansi.sql` with the following contents:

```
SELEct
    1 one, 
2 AS two 
from dbo.test 
LIMIT 100;

```

3.  Stage the `ansi.sql` file and attempt to commit it into your branch. Given the default settings, the commit should fail with a similar error to the following screenshot:

   ![](https://www.kimanimbugua.com/images/pre-commit-sqlfluff-lint-failed.jpg)
 

4.  With the previous file staged, execute `pre-commit run sqlfluff-lint` and notice that along with the failure, the rules that have been violated are also mentioned e.g., _L001, L009, L012_.

   ![](https://www.kimanimbugua.com/images/pre-commit-sqlfluff-lint-failed-cli.jpg)
 

5.  Amend the `ansi.sql` file with the following contents, stage and run `pre-commit run sqlfluff-lint`. This should pass the rule checks because the file specified is now consistent with the rules we defined.

```
SELECT
    1 AS one,
    2 AS two
FROM dbo.test
LIMIT 100;

```

### Passing arguments

The pre-commit hook we set up is basic but as mentioned in previous posts, these hooks are scripts so we can expect to pass arguments.

In the SQL Fluff [documentation](https://docs.sqlfluff.com/en/stable/cli.html), you can find the many arguments you can pass and for the purposes of this post, we’ll illustrate how we can pass arguments in a pre-commit hook.

##### Arguments example

In the following example, we will:

*   target a specific directory
*   specify the T-SQL dialect
*   exclude [rules](https://docs.sqlfluff.com/en/stable/rules.html#)

To do this via the SQL fluff pre-commit hook, perform the following steps:

1.  Create and stage a file named `t.sql` in the _sql/dir2/_ directory with the following contents:

```
SELECT
    t.Id AS test_id,
    s.name AS script_name
FROM dbo.test t
LEFT JOIN dbo.script s ON t.test_id = s.test_id
LIMIT 100;

```

2.  Amend the `pre-commit-config.yaml` file with the following contents:

```
-   repo: https://github.com/sqlfluff/sqlfluff
    rev: 0.9.1
    hooks:
    -   id: sqlfluff-lint
        args: [--exclude-rules, "L011,L031",
              --dialect, "tsql"
        ]
        files: 'sql/dir2/'

```

3.  Run `pre-commit run sqlfluff-lint` and you should get a failure, similar to the following:

   ![](https://www.kimanimbugua.com/images/pre-commit-sqlfluff-lint-failed-argument.jpg)
 

4.  Explore other arguments and try and fix the file before carrying on.

### Fix linting issues

It’s one challenge to find linting issues in SQL via pre-commit hooks or another tool but it’s quite another to fix them, especially for well-established or legacy code bases.

Another benefit to using SQL Fluff is the ability to ‘fix’ linting issues. Fixing basic linting issues that are widespread, such as _unnecessary trailing whitespace_, can be quite tedious but with SQL Fluff, this can be done quite quickly and effectively as we’ll demonstrate shortly.

1.  With a sql file in the following directory `src/sql/dir2/` (or other as preferred), execute the command `sqlfluff fix 'src/sql/dir2/'` against a script with the following contents:

```
SELECT
    t.Id AS test_id,  
    s.name AS script_name  
FROM dbo.test t
LEFT JOIN dbo.script s ON t.test_id = s.test_id
LIMIT 100;

```

2.  The program will scan that directory or sql files that have violated the rules currently set and prompt for a response to proceed or not.
    
3.  In the following screenshot, I’ve chosen to proceed and the result of the fix is shown. Do note that not all violations are ‘fixable’ and for those that are not, the violations will need to be fixed manually.
    

   ![](https://www.kimanimbugua.com/images/pre-commit-sqlfluff-fix-violation.jpg)
 

Further thoughts
----------------

##### Review fixes

Always review `sqlfluff fix` output because some of the ‘fixes’ aren’t what we would like such as 3-part naming or removing aliasing, as show in the below comparison.

   ![](https://www.kimanimbugua.com/images/pre-commit-sqlfluff-fix-review.jpg)
 

Remember these ‘fixes’ are dependent on rules defined so pay attention to that, resist the urge to blindly accept the fixes and again, always review to make sure it really is a fix.

##### Review rules

One of the key aspects of working with SQL Fluff is the set of rules available. There are over 50 rules that can be configured or ignored and more to be added.

In my experience with SQL Fluff, I find that I tend to exclude a few rules such as “L011” and “L031”, based on a preferred formatting style.

Review the rules so that they work for you and most importantly, that they make sense, especially for established codebases.

##### Other tools

There are other tools that can be used for linting in SQL such as [SQL Enlight](https://sqlenlight.com/) and ScriptDOM but for the most part, SQL Fluff seems, personally, to be easier to use and comprehensive.

Closing thoughts
----------------

This final part of the series has shown how basic formatting standards can be enforced simply and effectively to improve SQL code quality.

An important point to improving code quality is how you bring a tool such as SQL Fluff to a team. The approach from SQL fluff of [how to roll it out to teams](https://docs.sqlfluff.com/en/stable/teamrollout.html), seems to capture this well.

Having covered pre-commit as a framework to manage automated code reviews before check in, we’ve shown how to help protect against secret exposure, as well as how we can enforce formatting and code inspection _before_ code is checked-in to source control.

Regardless of the framework, automation, tooling, standards or process, using all of these together along with sound judgement is what will ultimately get us to where we want to be.

Thanks for reading and I hope you found the pre-commit series useful.

Useful links
------------

[SQL Fluff documentation](https://docs.sqlfluff.com/en/stable/)

[Using SQL Fluff pre-commit hook](https://docs.sqlfluff.com/en/stable/production.html?highlight=pre-commit#using-pre-commit)

[Azure repos pre-commit hooks-part 1](https://www.kimanimbugua.com/post/azure-repos-pre-commit-hooks-part-1/)

[Azure repos pre-commit hooks-part 2](https://www.kimanimbugua.com/post/azure-repos-pre-commit-hooks-part-2/)