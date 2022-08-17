# Create a Git hook to check that a commit message starts with a JIRA ticket number

Git has a `commit-msg` hook that can validate commit messages. This can be used to check that commit messages begin with a JIRA ticket number, and it will fail the commit if the number is incorrect or missing.

On our project, JIRA ticket number begin with two letters followed by a hyphen, followed by four digits, e.g. `XY-1234`, and thus commit messages should be formatted `XY-1234: message title text`.

We can create a script in Python to check that this message format has been followed.

```python
#!env python

import re
import sys

with open(sys.argv[1]) as tempfile:
  msg = tempfile.read()

if not re.match(r"[A-Z][A-Z]-\d{4}:\s\w+", msg):
  sys.stdout.write("\033[1;31m")  # Red
  sys.stdout.write("Commit message is not formatted correctly\n")
  sys.stdout.write("\033[0;0m")   # Reset
  sys.exit(1)
	
sys.exit(0)
```

The script is able to access the commit message by opening a file that Git has passed as the first argument.

As well as validating that the message starts with the ticket number, the script also checks for the presence of a colon and a space between the ticket number and the message. 

Save the above script to a file called `commit-msg` in your `<project>/.git/hooks` for it to take effect for just that project.

Alternatively, install it globally for all projects by saving it to a specific directory somewhere on your machine, e.g. `~/git_hooks` and then tell git about it with:

```shell
git config --global core.hooksPath ~/git_hooks
```
