---
id: ide-specification
title: Implementing an IDE plugin for analyzing custom rules
sidebar_label: IDE Specifications
description: Implement your own IDE plugin to integrate Codiga Custom Rules
keywords:
  - vscode
  - jetbrains
  - intellij
  - code analysis rules
  - sast
---

## How an analysis happens

### Sending a Query

When triggering an analysis, the IDE gets the list of rules for the current files.

It then sends a query to the service at `analysis.codiga.io` (HTTPS). The query is specified as is:

```json
{
  "filename": "myfile.py",
  "language": "python",
  "fileEncoding": "utf-8",
  "codeBase64": "<code-encoded-in-base64",
  "rules": [
    {
      "id": "ruleset/rulename",
      "language": "python",
      "type": "ast",
      "entityChecked": "functioncall",
      "contentBase64": "<rule-code-encoded-in-base64",
      "pattern": "pattern"
    }
  ],
  "logOutput": false
}
```

There is the details for every field:

- `filename`: the path of the file being analyzed. The path is relative to the project path.
- `language`: the programming language being analyzed
- `fileEncoding`: the file encoding in the IDE (`utf-8` works most of the time)
- `codeBase64`: the code in the IDE, encoded in Base 64
- `rules`: an array of rules, each one having the following attributes
  - `id`: full identifier of the rule
  - `language`: language of the rule. It must match the top-definition in the rule, otherwise, the rule is ignored
  - `type`: if the rule checks the Abstract Syntax Tree (`ast`) of a Pattern (`pattern`)
  - `entityChecked`: **ONLY FOR TYPE `ast`** the AST node/entity being checked (`functioncall`, `functiondefinition`, `exceptionblock`)
  - `contentBase64`: the JavaScript code of the rule
  - `pattern`: **ONLY FOR TYPE `pattern`** the pattern being checked in the code
- `logOutput`: if `true`, the rule output is being captured.

### Getting the results

The server response has the following schema:

```json
{
  "ruleResponses": [
    {
      "id": "ruleset/rulename",
      "violations": [
        {
          "message": "there is an important error!",
          "start": {
            "line": 1,
            "col": 10
          },
          "end": {
            "line": 1,
            "col": 10
          },
          "severity": "CRITICAL",
          "category": "SECURITY",
          "fixes": [
            {
              "description": "there is a fix for you",
              "edits": [
                {
                  "start": {
                    "line": 1,
                    "col": 10
                  },
                  "end": {
                    "line": 1,
                    "col": 10
                  },
                  "content": "myfix",
                  "editType": "update"
                }
              ]
            }
          ]
        }
      ],
      "errors": [],
      "executionError": null,
      "output": null
    }
  ],
  "errors": []
}
```

- `ruleResponses`
  - `errors`: contains the list of potential execution errors for this rule. Potential values are `invalid-pattern`, `error-unknown`, `error-execution`, `rule-timeout`, `invalid-rule-type`, `language-mismatch`
  - `violations`: the list of violations returned by the rule, each violation has
    - `message`: a message to show in the editor about the issue
    - `start`: the position to start to highlight in the IDE (line and col)
    - `end`: the position to end to highlight in the IDE (line and col)
    - `severity`: the severity of the violation, select it to show how to display the violation. Values are `CRITICAL`, `ERROR`, `INFORMATIONAL`, `UNKNOWN`
    - `category`: the category of the violation. Possible values are: `ERROR_PRONE`, `CODE_STYLE`, `BEST_PRACTICE`, `SAFETY`, `SECURITY`, `DESIGN`, `DEPLOYMENT`, `UNKNOWN`
    - `fixes`: a list of fix, each containing
      - `description`: the description of the fix to show in the IDE
      - `edits`: list of edit to apply **sequencially**, each edit has
        - an `editType` to explain what action to do. Values are `add`, `update` or `remove`
        - a `start` position: where to start editing the code
        - an `end` position: where to stop editing the code
        - a `content`: what to add or remove in the code
  - `executionError`: if the error `error-execution` is set, the attribute contains a message that explains why the execution fails
  - `output`: if the request has `logOutput` is true, this attribute contains the output of the rule (e.g. what is written on `console.log`)
- `errors`: list of errors when processing the requests. These are the errors that apply to the request and not the rule. List of errors: `invalid-request`, `language-not-supported`, `code-not-base64`, `rule-not-base64`

### Triggering an analysis

The analysis should be triggered after the user stop typing. Some IDEs provide a hook when to trigger such analysis. If not, the plugin
should detect that the user stopped writing code for at least 500ms and then trigger the analysis by:

- sending the code with all rules
- getting the results and annotating the code

## Getting rules

For each opened file in the editor, we should cache the list of rules that applies. A background job takes care of updating the rules
for a specific file. The rules are retrieved from the [Codiga API](/docs/api).

There are two cases to fetch the rules:

1. There is a `.codiga` file present in the repository
2. No `.codiga` file is present in the repository

### `.codiga` file is present

To find the `.codiga` file, we walk the project directory backwards.

Imagine we have the following file hierarchy

- `module1`
  - `.codiga`
  - `subdir1`
    - `myfile.py`
- `module2`
  - `subdir2`
    - `myfile2.py`
- `.codiga`

If we edit the file `module1/subdir1/myfile.py`, the file `module1/.codiga` will be used to get the rules.

If we edit the file `module2/subdir2/myfile2.py`, the file `.codiga` will be used to get the rules.

#### `.codiga` file structure

The `.codiga` file is a YAML file defined as is:

```yaml
 - rulesets
   - my-python-ruleset
   - my-other-ruleset
     - rule1
       - enabled: false

```

The `rulesets` elements list all the rulesets being used for the IDE.

Rules in the ruleset can be disabled by specifying `enabled: false` to the rule.

### `.codiga` file is absent

If the `.codiga` file is absent, the IDE fetches default rules for the language from the Codiga API.
The background job continues to poll for the presence of the file.

### Debugging

To debug and troubleshoot our analyzer, we can specify rules directly to the IDE. This is done by
having a `.codiga.debug` file. This is a JSON file that contains all the rules information to send
to the analysis service.

There is the definition of such a file:

```json
{
  "rules": [
    {
      "id": "empty-parameters",
      "contentBase64": "ICAgICAgICBmdW5jdGlvbiB2aXNpdChub2RlKSB7CiAgICAgICAgICAgIGNvbnN0IHBhcmFtZXRlcnNXaXRoRW1wdHlBcnJheSA9IG5vZGUucGFyYW1ldGVycy52YWx1ZXMuZmlsdGVyKHAgPT4gcCAmJiBwLmRlZmF1bHRWYWx1ZSAmJiBwLmRlZmF1bHRWYWx1ZS52YWx1ZSA9PT0gIltdIik7CgogICAgICAgICAgICBmb3IodmFyIGkgPSAwIDsgaSA8IHBhcmFtZXRlcnNXaXRoRW1wdHlBcnJheS5sZW5ndGggOyBpKyspIHsKICAgICAgICAgICAgICAgIGNvbnN0IHBhcmFtZXRlciA9IHBhcmFtZXRlcnNXaXRoRW1wdHlBcnJheVtpXTsKICAgICAgICAgICAgICAgIGNvbnNvbGUubG9nKHBhcmFtZXRlci5uYW1lLnZhbHVlKTsKICAgICAgICAgICAgICAgIGNvbnNvbGUubG9nKHBhcmFtZXRlci5zdGFydC5jb2wpOwogICAgICAgICAgICAgICAgY29uc3QgZXJyb3IgPSBidWlsZEVycm9yKHBhcmFtZXRlci5kZWZhdWx0VmFsdWUuc3RhcnQubGluZSwgcGFyYW1ldGVyLmRlZmF1bHRWYWx1ZS5zdGFydC5jb2wsIHBhcmFtZXRlci5kZWZhdWx0VmFsdWUuZW5kLmxpbmUsIHBhcmFtZXRlci5kZWZhdWx0VmFsdWUuZW5kLmNvbCwgImNhbm5vdCB1c2UgZGVmYXVsdCBpbml0aWFsaXplciBbXSBpbiBmdW5jdGlvbiIsICJDUklUSUNBTCIsICJTQUZFVFkiKTsKICAgICAgICAgICAgICAgIGFkZEVycm9yKGVycm9yKTsKICAgICAgICAgICAgfQogICAgICAgIH0=",
      "language": "python",
      "type": "ast",
      "entityChecked": "functiondefinition"
    },
    {
      "id": "timeout-request",
      "contentBase64": "ICAgICAgICBmdW5jdGlvbiB2aXNpdChub2RlKSB7CiAgICAgICAgICAgIG5vZGUuYXJndW1lbnRzLnZhbHVlcy5maWx0ZXIoYSA9PiBhLm5hbWUpLmZvckVhY2goYSA9PiBjb25zb2xlLmxvZyhhLm5hbWUudmFsdWUpKTsKICAgICAgICAgICAgY29uc3QgaGFzVGltZW91dCA9IG5vZGUuYXJndW1lbnRzLnZhbHVlcy5maWx0ZXIoYSA9PiBhLm5hbWUgJiYgYS5uYW1lLnZhbHVlID09ICJ0aW1lb3V0IikubGVuZ3RoID4gMDsKICAgICAgICAgICAgY29uc3QgYXJndW1lbnRzID0gbm9kZS5hcmd1bWVudHMudmFsdWVzOwogICAgICAgICAgICBjb25zdCBuYkFyZ3VtZW50cyA9IG5vZGUuYXJndW1lbnRzLnZhbHVlcy5sZW5ndGg7CiAgICAgICAgICAgIGNvbnN0IGFsbFBhY2thZ2VzID0gbm9kZS5nZXRJbXBvcnRzKCkuZmxhdE1hcChpID0+IGkucGFja2FnZXMubWFwKHAgPT4gcC5uYW1lLnN0cikpOwogICAgICAgICAgICBjb25zdCB1c2VSZXF1ZXN0c1BhY2thZ2UgPSBhbGxQYWNrYWdlcy5maWx0ZXIoaSA9PiBpID09PSAicmVxdWVzdHMiKS5sZW5ndGggPiAwOwogICAgICAgICAgICBjb25zb2xlLmxvZygiSEFTIFRJTUVPVVQ6IitoYXNUaW1lb3V0KTsKICAgICAgICAgICAgaWYoIWhhc1RpbWVvdXQgJiYgdXNlUmVxdWVzdHNQYWNrYWdlICYmIG5vZGUuZnVuY3Rpb25OYW1lLnZhbHVlID09PSAiZ2V0IiAmJiBub2RlLm1vZHVsZU9yT2JqZWN0LnZhbHVlID09PSAicmVxdWVzdHMiKXsKICAgICAgICAgICAgICAgIGNvbnN0IGVycm9yID0gYnVpbGRFcnJvcihub2RlLnN0YXJ0LmxpbmUsIG5vZGUuc3RhcnQuY29sLCBub2RlLmVuZC5saW5lLCBub2RlLmVuZC5jb2wsICJ0aW1lb3V0IG5vdCBkZWZpbmVkIiwgIkNSSVRJQ0FMIiwgIlNBRkVUWSIpOwogICAgICAgICAgICAgICAgY29uc3QgbGluZVRvSW5zZXJ0ID0gYXJndW1lbnRzW2FyZ3VtZW50cy5sZW5ndGggLSAxXS5lbmQubGluZTsKICAgICAgICAgICAgICAgIGNvbnN0IGNvbFRvSW5zZXJ0ID0gYXJndW1lbnRzW2FyZ3VtZW50cy5sZW5ndGggLSAxXS5lbmQuY29sICsgMTsKICAgICAgICAgICAgICAgIGNvbnN0IGVkaXQgPSBidWlsZEVkaXRBZGQobGluZVRvSW5zZXJ0LCBjb2xUb0luc2VydCwgIiwgdGltZW91dD01IikKICAgICAgICAgICAgICAgIGNvbnN0IGZpeCA9IGJ1aWxkRml4KCJhZGQgdGltZW91dCBhcmd1bWVudCIsIFtlZGl0XSk7CiAgICAgICAgICAgICAgICBhZGRFcnJvcihlcnJvci5hZGRGaXgoZml4KSk7CiAgICAgICAgICAgIH0KICAgICAgICB9",
      "language": "python",
      "type": "ast",
      "entityChecked": "functioncall"
    }
  ]
}
```

When such a file is present, we take the content of this file and send the rules directly to the analysis service.

## API Token

If the user has an API token specified, the token is specified with the `X-Api-Token` header.

## Network latency

Fetching rules occur in the IDE and may incur some latency. For this very reason, fetching data from the API should be always done asynchronously and never incur any lag or latency in the IDE.

## Preferences

In the IDE, we should keep the following preferences

- **API Token**: the API token must be stored and secured. Some IDE provide a way to securely store a password/api token. If available, such services must be used.
- **Enabled/disabled**: we should provide a way to enable/disable the analysis
